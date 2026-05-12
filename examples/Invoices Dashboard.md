# Invoices Dashboard

> Auto-updated from frontmatter of every note in `Invoices/`. Mark an invoice as paid by changing its frontmatter `status: unpaid` to `status: paid`.
>
> **If your invoices folder is not named `Invoices`**, find and replace `"Invoices"` below with your folder path (e.g., `"Work/Invoices"`).

---

## Outstanding (unpaid total)

```dataview
LIST WITHOUT ID
  "**" + sum(rows.amount) + " EUR**" + " across " + length(rows) + " unpaid invoice(s)"
FROM "Invoices"
WHERE type = "invoice" AND status = "unpaid"
GROUP BY true
```

## Overdue

```dataview
TABLE WITHOUT ID
  file.link AS Invoice,
  client AS Client,
  amount + " EUR" AS Amount,
  dateformat(due_date, "dd.MM.yyyy") AS Due,
  (date(today) - due_date).days + "d" AS "Overdue"
FROM "Invoices"
WHERE type = "invoice" AND status = "unpaid" AND due_date < date(today)
SORT due_date ASC
```

## All unpaid

```dataview
TABLE WITHOUT ID
  file.link AS Invoice,
  client AS Client,
  amount + " EUR" AS Amount,
  dateformat(issue_date, "dd.MM.yyyy") AS Issued,
  dateformat(due_date, "dd.MM.yyyy") AS Due
FROM "Invoices"
WHERE type = "invoice" AND status = "unpaid"
SORT due_date ASC
```

## Revenue this year

```dataview
LIST WITHOUT ID
  "**" + sum(rows.amount) + " EUR**" + " from " + length(rows) + " invoice(s)"
FROM "Invoices"
WHERE type = "invoice" AND issue_date.year = date(today).year
GROUP BY true
```

## By client (all time)

```dataview
TABLE WITHOUT ID
  client AS Client,
  sum(rows.amount) + " EUR" AS Total,
  length(rows) AS Invoices
FROM "Invoices"
WHERE type = "invoice"
GROUP BY client
SORT sum(rows.amount) DESC
```

## All invoices (most recent first)

```dataview
TABLE WITHOUT ID
  file.link AS Invoice,
  client AS Client,
  amount + " EUR" AS Amount,
  dateformat(issue_date, "dd.MM.yyyy") AS Issued,
  status AS Status,
  vs AS "VS"
FROM "Invoices"
WHERE type = "invoice"
SORT issue_date DESC
```
