# Contributing

Pull requests welcome. A few notes to keep things smooth:

## What this repo is

A workflow for generating SEPA-compatible EUR invoices inside Obsidian. The scope is intentionally narrow — invoices, dashboard, payment QR. It's not aiming to become a full ERP.

## Easy first contributions

- **Translations**: translate visible strings (SUPPLIER, BILL TO, Invoice No., etc.) and submit as a separate template variant (e.g., `invoice-template-de.md`, `invoice-template-fr.md`).
- **Documentation fixes**: typos, clearer install steps, screenshot additions.
- **Regional Reg. No. labels**: e.g., "VAT ID" for UK, "Steuernummer" for Germany, "IČO" for Czech/Slovak.

## Larger features

These would be valuable but require more thought:

- **VAT support** — needs config for VAT rate, a VAT line in the items table, and inclusion of VAT in the EPC amount field. Important: invoices for VAT-registered businesses have strict legal requirements per country. Anyone implementing this should research their target jurisdiction.
- **Czech SPAYD QR** — alongside (or instead of) EPC, useful for Czech-only clients. Format documented at [qr-platba.cz](https://qr-platba.cz/).
- **Swiss QR-bill** — different payload format, would be its own template variant.
- **Recurring invoices** — schedule a monthly client charge.
- **Expense tracker companion** — a sister template + dashboard for tracking expenses, with combined P&L view.

## Code style

- Keep the CONFIG block at the top of the template, well-commented.
- Inline `style="..."` for HTML — no external CSS dependencies.
- Comment any non-obvious logic (especially around EPC payload structure or invoice-number scanning).
- The template should NEVER include personal data when committed — only placeholders.

## Testing your changes

Before submitting a PR:

1. Replace CONFIG values with test data
2. Generate at least 3 invoices to confirm:
   - Auto-numbering increments correctly
   - QR code is scannable and shows correct data in a bank app
   - PDF export preserves layout
3. Test the Dashboard updates when status changes
4. Make sure the safety guard still triggers when running on the template itself

## Sanitizing before commit

Before pushing, double-check the template doesn't contain:
- Real IBAN / BIC values
- Real names, addresses, or Reg. Nos.
- Real invoice data in the examples folder

The `.gitignore` excludes the `Invoices/` folder, but always double-check.
