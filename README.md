# Manager Email Bridge

Adds dynamic tags and legacy Liquid support to Sales Invoice emails. Uses Manager’s existing email composer, PDF attachment and Send button.

## Install

1. In Manager, go to **Settings → Custom Buttons → New**.
2. Set **Name** to `Enhanced Email` *(or anything you'd like)*.
3. Set **Source** to **Inline** and paste the entire contents of [manager-email-bridge.html](manager-email-bridge.html) into the HTML field.
4. Set **Placement** to `sales-invoice-view`.
5. Save, open a Sales Invoice, and click **Enhanced Email**.

Add your placeholders to Manager’s existing **Sales Invoice Email Template**, then reopen the invoice. Review the prepared email and click Manager’s normal **Send**.

Manager’s email settings must already be configured. The customer needs an email address, and **Hide balance due** must be off on the invoice. Tested on Manager **26.8.4.3664**.

## Dynamic tags

These tags work in both the **subject and body**. Copy the spelling and capitalisation exactly.

| Tag | Value |
|---|---|
| `{BusinessName}` | Business name |
| `{CustomerName}` | Customer name |
| `{CustomerEmail}` | Customer email address |
| `{InvoiceNumber}` | Invoice reference |
| `{InvoiceReference}` | Same as InvoiceNumber |
| `{InvoiceDate}` | Formatted invoice date |
| `{DueDate}` | Formatted due date, or blank if missing/hidden |
| `{InvoiceTotal}` | Full invoice total |
| `{BalanceDue}` | Outstanding balance, including zero for a fully paid invoice |

Unknown tags stay unchanged. If the balance cannot be determined safely, the extension shows an error.

Example:

```text
Dear {CustomerName},

Invoice {InvoiceNumber} has an outstanding balance of {BalanceDue}, due on {DueDate}.
```

## Compatibility with old Liquid

You can mix simple tags and Liquid in the same template. Every tag above also works as a Liquid variable, such as `{{ CustomerName }}` or `{{ BalanceDue }}`.

These older variables are supported:

```liquid
{{ reference }}
{{ recipient.name }}
{{ recipient.email }}
{{ business.name }}
```

Existing Liquid loops, conditions and captures can use these exact labels:

| Collection | Supported labels | Value property |
|---|---|---|
| `fields` | `Invoice date`, `Due date` | `field.text` |
| `table.totals` | `Total`, `Balance due` | `total.text` |

For example, this old balance lookup works unchanged:

```liquid
{% for total in table.totals %}
  {% if total.label == 'Balance due' %}
    {{ total.text }}
  {% endif %}
{% endfor %}
```

Compatibility covers the variables listed here; other old theme variables, line items and custom fields are not included. Body values are already HTML-escaped, so remove an extra `| escape` filter from these values if your old template uses one.
