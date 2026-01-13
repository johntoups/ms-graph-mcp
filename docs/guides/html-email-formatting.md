---
title: HTML Email Formatting Guide
type: guide
created: 2025-11-14
updated: 2025-11-14
---

# HTML Email Formatting Guide

Both `send_email`, `create_draft_email`, and `edit_draft_email` tools support rich HTML formatting.

---

## How to Use HTML Formatting

Set `body_type="html"` parameter and use HTML tags in the body:

**Example:**
```python
create_draft_email(
    to_address="patrick@example.com",
    subject="SIB Proposal Update",
    body="""
    <h2>Project Update</h2>
    <p>Hi <strong>Patrick</strong>,</p>
    <p>Following up on our discussion:</p>
    <ul>
      <li><strong>Budget:</strong> £45,000</li>
      <li><strong>Timeline:</strong> 6 weeks</li>
    </ul>
    <p style="color: #0066CC">Looking forward to your feedback.</p>
    """,
    body_type="html"
)
```

---

## Supported HTML Elements

| Feature         | HTML Syntax                               | Example Output     |
|-----------------|-------------------------------------------|--------------------|
| Bold            | `<strong>text</strong>` or `<b>text</b>`  | **bold text**      |
| Italic          | `<em>text</em>` or `<i>text</i>`          | *italic text*      |
| Underline       | `<u>text</u>`                             | underlined         |
| Font size       | `<span style="font-size: 18px">text</span>` | Larger text      |
| Color           | `<span style="color: #FF0000">text</span>` | Red text          |
| Links           | `<a href="url">text</a>`                  | [url](url)         |
| Headings        | `<h1>`, `<h2>`, `<h3>`                    | Heading levels     |
| Bullet lists    | `<ul><li>item</li></ul>`                  | • Bullet points    |
| Numbered lists  | `<ol><li>item</li></ol>`                  | 1. Numbered items  |
| Paragraphs      | `<p>text</p>`                             | Paragraph spacing  |
| Line breaks     | `<br>`                                    | Manual line breaks |
| Horizontal line | `<hr>`                                    | Divider line       |
| Tables          | `<table><tr><td>cell</td></tr></table>`   | Data tables        |
| Blockquotes     | `<blockquote>text</blockquote>`           | Quoted text        |

---

## Best Practices

- **Always set body_type:** Use `body_type="html"` when using HTML tags
- **Semantic HTML:** Prefer `<strong>` over `<b>`, `<em>` over `<i>`
- **Test rendering:** Check formatting in Outlook before sending to important recipients
- **Keep it simple:** Complex layouts may not render consistently across email clients
- **Accessibility:** Provide plain text alternative for critical content
- **Inline styles:** Use inline style attributes for formatting (external CSS not supported)

---

## Example: Formatted Business Email

```python
edit_draft_email(
    message_id="AAMkA...",
    body="""
    <h2 style="color: #2E4057;">Project Status Update</h2>

    <p>Hi <strong>Sarah</strong>,</p>

    <p>Quick update on the SIB deployment:</p>

    <table border="1" cellpadding="10" style="border-collapse: collapse;">
      <tr style="background-color: #f0f0f0;">
        <th>Milestone</th>
        <th>Status</th>
        <th>Due Date</th>
      </tr>
      <tr>
        <td>Hardware Setup</td>
        <td><span style="color: green;">✓ Complete</span></td>
        <td>Nov 10</td>
      </tr>
      <tr>
        <td>Software Installation</td>
        <td><span style="color: orange;">In Progress</span></td>
        <td>Nov 15</td>
      </tr>
      <tr>
        <td>Testing</td>
        <td><span style="color: gray;">Pending</span></td>
        <td>Nov 20</td>
      </tr>
    </table>

    <p><em>Let me know if you have questions.</em></p>

    <p style="color: #666; font-size: 12px;">
    <hr>
    Thomas Keeling<br>
    Six Pillar Technology<br>
    <a href="mailto:thomas@sixpillar.co.uk">thomas@sixpillar.co.uk</a>
    </p>
    """,
    body_type="html"
)
```

---

## Tips for Professional Emails

### Use Consistent Branding

```python
# Define brand colors
BRAND_PRIMARY = "#2E4057"    # Dark blue
BRAND_ACCENT = "#0066CC"     # Light blue
BRAND_SUCCESS = "#00AA00"    # Green
BRAND_WARNING = "#FF9900"    # Orange

# Use in templates
f"<h2 style='color: {BRAND_PRIMARY};'>Title</h2>"
```

### Create Reusable Signatures

```python
def email_signature(name, title, company):
    return f"""
    <p style="color: #666; font-size: 12px; margin-top: 20px;">
    <hr style="border: none; border-top: 1px solid #ccc;">
    <strong>{name}</strong><br>
    {title}<br>
    {company}<br>
    </p>
    """
```

### Structure Long Emails

```python
body = """
<h2>Executive Summary</h2>
<p>[Key points here]</p>

<h2>Details</h2>
<p>[Expanded information]</p>

<h2>Next Steps</h2>
<ol>
  <li>Action item 1</li>
  <li>Action item 2</li>
</ol>
"""
```

---

## Troubleshooting

### HTML Not Rendering

**Problem:** Email shows HTML tags instead of formatted content

**Solution:** Ensure `body_type="html"` parameter is set

### Inconsistent Rendering

**Problem:** Email looks different in different clients (Outlook, Gmail, etc.)

**Solutions:**
- Use inline styles instead of CSS classes
- Avoid complex nested tables
- Test in multiple clients before important sends

### Tables Not Displaying Correctly

**Problem:** Table borders or spacing incorrect

**Solution:** Use explicit table attributes:
```html
<table border="1" cellpadding="10" cellspacing="0" style="border-collapse: collapse;">
```

---

## Related Documentation

- **Tools using HTML formatting:** send_email, create_draft_email, edit_draft_email
- **Testing guide:** [testing/verification-procedures.md](../testing/verification-procedures.md)
- **Tool reference:** [README.md](../README.md#mcp-tools)

---

**Last Updated:** 2025-11-14
