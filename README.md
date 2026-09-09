# Manager Email Bridge

Adds dynamic tags and legacy Liquid support to Sales Invoice emails. Uses Manager’s existing email composer, PDF attachment and Send button.

## Install

1. In Manager, go to **Settings → Custom Buttons → New**.
2. Set **Name** to `Enhanced Email` *(or anything you'd like)*.
3. Set **Source** to **Inline** and paste the entire contents of [manager-email-bridge.html](manager-email-bridge.html) into the HTML field.
4. Set **Placement** to `sales-invoice-view`.
5. Save, open a Sales Invoice, and click **Enhanced Email**.

Add your placeholders to Manager’s existing **Sales Invoice Email Template**, then reopen the invoice with the new custom button. Review the prepared email and click Manager’s normal **Send**.

Manager’s email settings must already be configured. The customer needs an email address, and **Hide balance due** must be off on the invoice.

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

Unknown tags stay unchanged. Amounts keep Manager’s currency formatting, with trailing whitespace removed so punctuation can follow directly: `The amount due is {BalanceDue}.` If the balance cannot be determined safely, the extension shows an error.

Example:

```text
Dear {CustomerName},

Invoice {InvoiceNumber} has an outstanding balance of {BalanceDue}, due on {DueDate}.
```

## Amount formatting

`{InvoiceTotal}` and `{BalanceDue}` keep their existing Manager formatting. Add a suffix to change how an amount appears in either the subject or body. Every option below works with **both** names.

Examples for an amount normally displayed as `$ 1,234.56`:

| Tag | Output |
|---|---|
| `{InvoiceTotal}` | `$ 1,234.56` |
| `{InvoiceTotal_NoCurrency}` | `1,234.56` |
| `{InvoiceTotal_Compact}` | `$1,234.56` |
| `{InvoiceTotal_Spaced}` | `$ 1,234.56` |
| `{InvoiceTotal_NoDecimals}` | `$ 1,235` |
| `{InvoiceTotal_2Decimals}` | `$ 1,234.56` |
| `{InvoiceTotal_NoGrouping}` | `$ 1234.56` |
| `{InvoiceTotal_Compact_2Decimals_NoGrouping}` | `$1234.56` |
| `{InvoiceTotal_Spaced_2Decimals_NoGrouping}` | `$ 1234.56` |
| `{InvoiceTotal_NoCurrency_NoDecimals_NoGrouping}` | `1235` |

Combine options in this order, omitting any you do not need:

1. **Currency:** `NoCurrency` removes the currency prefix/suffix, `Compact` removes the gap beside it, or `Spaced` uses one space beside it.
2. **Decimals:** `NoDecimals` rounds to a whole number, or `2Decimals` shows exactly two decimal places (including `.00`). Without either, the displayed precision is preserved.
3. **Grouping:** `NoGrouping` removes thousands separators. Without it, Manager’s grouping is preserved.

For example:

```text
The amount due is {BalanceDue_Compact_2Decimals}.
Amount excluding currency: {BalanceDue_NoCurrency_2Decimals_NoGrouping}.
```

These options use the invoice’s configured currency, prefix/suffix position and number separators. They do not force dollars or convert currencies: `€ 1.234,56` becomes `€1.234,56` with `Compact`, or `1.234,56` with `NoCurrency`.

Decimal options round for display only, with halves rounded away from zero (`1.005` → `1.01`; `1234.50` → `1235` with no decimals). They do not change the invoice, PDF or actual balance. A paid balance remains zero, and a partial balance stays separate from the full invoice total.

Liquid equivalents work too:

```liquid
{{ InvoiceTotal_NoCurrency }}
{{ BalanceDue_Compact_2Decimals }}
```

Unknown suffixes or combinations in the wrong order stay unchanged as simple tags. If Manager’s amount or number format cannot be read safely, the extension shows an error instead of guessing. The original amount tags do not require the additional number-format lookup.

## Custom fields

Use `{CustomField_X}`, replacing `X` with the **exact custom field name**, including spaces and capitalisation:

| Field name in Manager | Tag |
|---|---|
| `Project Name` | `{CustomField_Project Name}` |
| `Purchase Order` | `{CustomField_Purchase Order}` |
| `JobNumber` | `{CustomField_JobNumber}` |

These tags work in both subject and body. For example:

```text
Invoice {InvoiceNumber} for {CustomField_Project Name}
Your purchase order reference is {CustomField_Purchase Order}.
```

Enable **Display on view** for the field and give it a value. The bridge uses the formatted text Manager returns for custom fields displayed on the Sales Invoice, including customer fields shown there. Dates and numbers keep Manager’s formatting; body values are HTML-escaped.

Hidden fields, image fields, business-detail fields and line-item fields are not supported. Missing or unknown tags remain unchanged. If two displayed fields have the same name, that tag also remains unchanged; give the fields unique names. Renaming a field requires updating its tag. Field names containing braces or line breaks are not supported.

For optional values, use Liquid instead so missing fields can be omitted:

```liquid
{% if CustomFields["Project Name"] != blank %}
Project: {{ CustomFields["Project Name"] }}
{% endif %}
```

## Compatibility with old Liquid

You can mix simple tags and Liquid in the same template. The standard tags and amount-format variants also work as Liquid variables, such as `{{ CustomerName }}` or `{{ BalanceDue_Compact }}`. Custom fields use `CustomFields["Field Name"]` as shown above.

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

Compatibility covers the variables listed here; other old theme variables and line items are not included. Custom fields use the `CustomFields["Field Name"]` syntax above. Body values are already HTML-escaped, so remove an extra `| escape` filter from these values if your old template uses one.

## Change wording when an invoice is overdue

`IsOverdue` is a Liquid boolean (`true` or `false`) available in both the **subject and body**. Use `{% if IsOverdue %}` to choose overdue wording and `{% else %}` for normal wording. Finish the condition with `{% endif %}`.

It is `true` when Manager reports an outstanding invoice as overdue and the invoice has a visible, explicitly configured due date. Manager uses its current balance and server date to determine that status. A partially paid invoice can still be overdue; a fully paid invoice is not. Invoices due today, with a future due date, or without a visible, explicitly configured due date use the normal wording.

### Subject example

```liquid
{% if IsOverdue %}Payment reminder for Invoice {InvoiceNumber}{% else %}Invoice {InvoiceNumber} from {BusinessName}{% endif %}
```

For invoice `100` from `Example Business`, this produces:

- **Overdue:** `Payment reminder for Invoice 100`
- **Normal:** `Invoice 100 from Example Business`

### Body example

```liquid
Dear {CustomerName},

{% if IsOverdue %}
A friendly reminder that invoice {InvoiceNumber} was due on {DueDate}. The remaining amount due is {BalanceDue}.
{% else %}
Please find invoice {InvoiceNumber} attached. Thank you for your business.
{% endif %}
```

To add a sentence only when overdue, leave out the `else` branch:

```liquid
{% if IsOverdue %}Please arrange payment of the outstanding {BalanceDue} when you can.{% endif %}
```

Use `IsOverdue` exactly as shown, without quotes or single braces. `{IsOverdue}` is not a supported simple tag. You can also use Liquid amount variables such as `{{ BalanceDue }}` inside either branch. Conditions change the wording; Manager’s native composer, PDF attachment and Send button work as usual.
