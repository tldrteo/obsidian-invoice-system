<%*
// ============================================================
// CONFIGURATION — edit these once, applies to every invoice
// ============================================================
const CONFIG = {
  supplier: {
    // Visible on the invoice (titles + diacritics are fine)
    legalName: "Your Legal Name",
    // Inside the QR code payload — MUST match your bank-registered
    // name exactly (typically no titles, no diacritics) or SEPA
    // transfers may be rejected for a name mismatch.
    bankName: "Your Bank Name",
    address1: "Street + No.",
    address2: "City + Postal Code",
    country: "Country",
    // Visible "Reg. No." on the invoice — your local business
    // registration number (Czech IČO, German Steuernummer, UK
    // Companies House number, etc.). Use blank string if none.
    regNo: "12345678",
    iban: "XX0000000000000000",
    bic: "BANKBIC22",
  },
  // Path to your invoices folder (relative to vault root, no trailing slash)
  invoicesFolder: "Invoices",
  // Days from issue date until payment is due
  paymentDays: 14,
};
// ============================================================

// Normalize folder path (remove any trailing slash defensively)
const INVOICES_FOLDER = CONFIG.invoicesFolder.replace(/\/+$/, "");

// Safety guard — refuse to run on the template file itself
const currentPath = tp.file.path(true);
if (currentPath.includes("Templates/") || tp.file.title === "invoice-template") {
  new Notice(`Don't run this on the template itself. Create a new note in ${INVOICES_FOLDER}/ instead.`, 6000);
  throw new Error("Aborted: running on template file");
}

// Auto-generate invoice number based on existing invoices for current year
// Format: YYYY + 3-digit zero-padded sequence (e.g., 2026001, 2026002, ...)
const year = moment().year();
const yearStr = String(year);
let maxSeq = 0;

try {
  const allFiles = app.vault.getMarkdownFiles().filter(f => f.path.startsWith(INVOICES_FOLDER + "/"));
  for (const f of allFiles) {
    const cache = app.metadataCache.getFileCache(f);
    const num = cache?.frontmatter?.invoice_number;
    if (num) {
      const numStr = String(num);
      if (numStr.startsWith(yearStr) && numStr.length >= 5) {
        const seqStr = numStr.slice(yearStr.length);
        const seq = parseInt(seqStr, 10);
        if (!isNaN(seq) && seq > maxSeq) maxSeq = seq;
      }
    }
  }
} catch (e) {
  console.error("Couldn't scan existing invoice numbers:", e);
}

const nextSeq = maxSeq + 1;
const invoiceNumDefault = `${yearStr}${String(nextSeq).padStart(3, "0")}`;

// Prompts — invoice + client
const invoiceNum = await tp.system.prompt("Invoice number", invoiceNumDefault);
const clientName = await tp.system.prompt("Client name");
const clientAddr = await tp.system.prompt("Client street address");
const clientCity = await tp.system.prompt("Client city + postal code");
const clientCountry = await tp.system.prompt("Client country");
const clientRegNo = await tp.system.prompt("Client Reg. No. (leave empty if individual)");

// Number formatter (English-style thousands separator, e.g., 1,234.56)
const fmt = (n) => parseFloat(n).toLocaleString("en-US", { minimumFractionDigits: 2, maximumFractionDigits: 2 });

// Line items — first is required, additional are optional via loop
const items = [];

// First item (required)
const firstDesc = await tp.system.prompt("Item description");
if (!firstDesc || firstDesc.trim() === "") {
  new Notice("Invoice must have at least one item.", 6000);
  throw new Error("No items added");
}
const firstQty = await tp.system.prompt("Quantity", "1");
const firstRate = await tp.system.prompt("Rate per unit (EUR)");
items.push({
  desc: firstDesc,
  qty: parseFloat(firstQty),
  rate: parseFloat(firstRate),
  amount: parseFloat(firstQty) * parseFloat(firstRate)
});

// Additional items — loop until empty description
while (true) {
  const desc = await tp.system.prompt("Add another item? (description, or leave empty to finish)");
  if (!desc || desc.trim() === "") break;
  const qty = await tp.system.prompt("Quantity", "1");
  const rate = await tp.system.prompt("Rate per unit (EUR)");
  items.push({
    desc: desc,
    qty: parseFloat(qty),
    rate: parseFloat(rate),
    amount: parseFloat(qty) * parseFloat(rate)
  });
}

const totalAmount = items.reduce((s, i) => s + i.amount, 0).toFixed(2);

// Issue date — defaults to today
const issueInput = await tp.system.prompt("Issue date (DD.MM.YYYY)", tp.date.now("DD.MM.YYYY"));
const issueMoment = moment(issueInput, "DD.MM.YYYY");
const dueMoment = issueMoment.clone().add(CONFIG.paymentDays, "days");
const issueDateISO = issueMoment.format("YYYY-MM-DD");
const issueDate    = issueMoment.format("D.M.YYYY");
const dueDateISO   = dueMoment.format("YYYY-MM-DD");
const dueDate      = dueMoment.format("D.M.YYYY");

// Random 10-digit variable symbol (payment reference for bank reconciliation)
const vs = Math.floor(1000000000 + Math.random() * 9000000000).toString();

// EPC QR payload (SEPA Credit Transfer standard, EPC069-12 v002)
// Field order is strict — each line is a separate field
const qrConfig = JSON.stringify({
  text: [
    "BCD",                                       // Service tag
    "002",                                       // Version
    "1",                                         // Character set (1 = UTF-8)
    "SCT",                                       // Identification (SEPA Credit Transfer)
    CONFIG.supplier.bic,                         // Beneficiary BIC
    CONFIG.supplier.bankName,                    // Beneficiary name (bank-registered, no diacritics)
    CONFIG.supplier.iban,                        // Beneficiary IBAN
    `EUR${totalAmount}`,                         // Amount (total of all line items)
    "",                                          // Purpose code (empty)
    "",                                          // Structured reference (empty)
    `Invoice ${invoiceNum} VS:${vs}`,            // Unstructured remittance info
  ].join("\n"),
  width: 140,
  margin: 2,
  errorCorrectionLevel: "M"
});

// Build HTML rows for line items
const itemRows = items.map(i => `<tr>
<td style="border-bottom:1px solid #f3f3f3; padding:14px 8px;">${i.desc}</td>
<td style="border-bottom:1px solid #f3f3f3; padding:14px 8px; text-align:right;">${i.qty}</td>
<td style="border-bottom:1px solid #f3f3f3; padding:14px 8px; text-align:right;">${fmt(i.rate)}</td>
<td style="border-bottom:1px solid #f3f3f3; padding:14px 8px; text-align:right;">${fmt(i.amount)}</td>
</tr>`).join("\n");

// Conditional Reg. No. line for BILL TO (only renders if user provided one)
const clientRegLine = clientRegNo
  ? `<div style="color:#333; margin-top:8px;"><span style="color:#888;">Reg. No.:</span> ${clientRegNo}</div>`
  : "";

// Auto-rename the file
await tp.file.rename(`${invoiceNum} ${clientName}`);
-%>
---
type: invoice
invoice_number: <% invoiceNum %>
client: <% clientName %>
client_reg_no: <% clientRegNo %>
amount: <% totalAmount %>
currency: EUR
issue_date: <% issueDateISO %>
due_date: <% dueDateISO %>
status: unpaid
vs: <% vs %>
---

<div style="font-family:'Helvetica Neue','Helvetica','Arial',sans-serif; color:#1a1a1a; line-height:1.5; max-width:900px; margin:0 auto; padding:20px 10px;">

<div style="text-align:right; padding-bottom:14px; margin-bottom:30px; border-bottom:1px solid #e5e5e5;">
<span style="color:#888; margin-right:10px;">Invoice No.</span><span style="font-size:1.6em; font-weight:700; color:#111;"><% invoiceNum %></span>
</div>

<table style="width:100%; border-collapse:collapse; margin-bottom:25px;">
<tr>
<td style="width:50%; vertical-align:top; padding:0 25px 0 0; border:none;">
<div style="color:#999; font-size:0.78em; font-weight:600; letter-spacing:0.1em; margin-bottom:10px;">SUPPLIER</div>
<div style="font-weight:700; font-size:1.1em; margin-bottom:6px; color:#111;"><% CONFIG.supplier.legalName %></div>
<div style="color:#333; line-height:1.45;"><% CONFIG.supplier.address1 %><br><% CONFIG.supplier.address2 %><br><% CONFIG.supplier.country %></div>
<div style="color:#333; margin-top:8px;"><span style="color:#888;">Reg. No.:</span> <% CONFIG.supplier.regNo %></div>
</td>
<td style="width:50%; vertical-align:top; padding:0 0 0 25px; border:none;">
<div style="color:#999; font-size:0.78em; font-weight:600; letter-spacing:0.1em; margin-bottom:10px;">BILL TO</div>
<div style="font-weight:700; font-size:1.1em; margin-bottom:6px; color:#111;"><% clientName %></div>
<div style="color:#333; line-height:1.45;"><% clientAddr %><br><% clientCity %><br><% clientCountry %></div>
<% clientRegLine %>
</td>
</tr>
</table>

<hr style="border:none; border-top:1px solid #e5e5e5; margin:25px 0;">

<table style="width:100%; border-collapse:collapse; margin-bottom:35px;">
<tr>
<td style="width:45%; vertical-align:top; padding:0 15px 0 0; border:none;">
<table style="border-collapse:collapse;">
<tr><td style="color:#888; padding:3px 20px 3px 0; border:none;">Variable symbol</td><td style="padding:3px 0; border:none;"><% vs %></td></tr>
<tr><td style="color:#888; padding:3px 20px 3px 0; border:none;">Payment method</td><td style="padding:3px 0; border:none;">Bank transfer</td></tr>
<tr><td style="color:#888; padding:3px 20px 3px 0; border:none;">IBAN</td><td style="padding:3px 0; border:none;"><% CONFIG.supplier.iban %></td></tr>
<tr><td style="color:#888; padding:3px 20px 3px 0; border:none;">BIC</td><td style="padding:3px 0; border:none;"><% CONFIG.supplier.bic %></td></tr>
</table>
</td>
<td style="width:30%; vertical-align:top; padding:0 15px; border:none;">
<table style="border-collapse:collapse;">
<tr><td style="color:#888; padding:3px 20px 3px 0; border:none;">Issue date</td><td style="padding:3px 0; border:none;"><% issueDate %></td></tr>
<tr><td style="color:#888; padding:3px 20px 3px 0; border:none;">Due date</td><td style="padding:3px 0; font-weight:700; border:none;"><% dueDate %></td></tr>
</table>
</td>
<td style="width:25%; vertical-align:top; padding:0 0 0 15px; border:none; text-align:center;">

```qrcode-complex
<% qrConfig %>
```

<div style="color:#999; font-size:0.7em; margin-top:4px;">Scan to pay</div>

</td>
</tr>
</table>

<table style="width:100%; border-collapse:collapse;">
<thead>
<tr>
<th style="border-bottom:1px solid #e5e5e5; padding:12px 8px; text-align:left; color:#888; font-weight:500; font-size:0.85em; letter-spacing:0.03em;">Item description</th>
<th style="border-bottom:1px solid #e5e5e5; padding:12px 8px; text-align:right; color:#888; font-weight:500; font-size:0.85em; letter-spacing:0.03em;">Qty</th>
<th style="border-bottom:1px solid #e5e5e5; padding:12px 8px; text-align:right; color:#888; font-weight:500; font-size:0.85em; letter-spacing:0.03em;">Unit price (EUR)</th>
<th style="border-bottom:1px solid #e5e5e5; padding:12px 8px; text-align:right; color:#888; font-weight:500; font-size:0.85em; letter-spacing:0.03em;">Total (EUR)</th>
</tr>
</thead>
<tbody>
<% itemRows %>
<tr>
<td colspan="3" style="padding:18px 8px 14px 8px; text-align:right; font-weight:600; color:#555; border-top:2px solid #333; letter-spacing:0.05em;">TOTAL</td>
<td style="padding:18px 8px 14px 8px; text-align:right; font-weight:700; font-size:1.15em; border-top:2px solid #333;"><% fmt(totalAmount) %> EUR</td>
</tr>
</tbody>
</table>

</div>
