# Attachment Tool Selection Guide

## Overview

Email attachment operations require choosing between **container-filesystem tools** (for server-side automation) and **base64-transfer tools** (for interactive user workflows). This guide ensures you pick the right tool on first attempt.

---

## Quick Decision Tree

```
Does the user want the actual file on their laptop/workspace?
├─ YES → Use base64 tools (download_attachment_base64, add_attachment_base64)
└─ NO → Is this server-side automation?
    ├─ YES → Use filesystem tools (download_attachment, add_attachment_to_draft)
    └─ NO → Do you need to extract text or process the file?
        ├─ YES → Use filesystem tools + extract_attachment_text
        └─ NO → Re-evaluate: why are you downloading?
```

---

## Tool Categories

### Category 1: Base64 Tools (Interactive User Workflows)

**When to use:**
- User requests: "download attachment to Downloads", "save invoice locally"
- User requests: "attach this file from Documents", "attach the report I created"
- File needs to be on **laptop or workspace** filesystem
- User will open/view/edit the file

**Tools:**
- `download_attachment_base64` - Returns base64 content, Claude writes to laptop/workspace
- `add_attachment_base64` - Accepts base64 content from laptop/workspace files

**Key phrases that trigger these:**
- "save to my Downloads"
- "attach this file from my Documents"
- "download and open"
- "get the invoice locally"
- "attach the report I just created"

---

### Category 2: Filesystem Tools (Server-Side Automation)

**When to use:**
- Server-side email processing automation
- File will be processed immediately (text extraction, parsing)
- File will be deleted after processing
- User doesn't need to see/access the file

**Tools:**
- `download_attachment` - Downloads to container `/root/.athena/attachments/`
- `add_attachment_to_draft` - Attaches from container filesystem
- `extract_attachment_text` - Reads from container filesystem

**Key phrases that trigger these:**
- "extract text from all invoices"
- "process email attachments and summarize"
- "forward email with attachments" (use forward_email, no download needed)
- Automation workflows (slash commands)

---

## Tool Descriptions

### download_attachment_base64 (NEW)

**Purpose:** Download email attachment and return base64 content to Claude for writing to laptop/workspace

**Use when:**
- User wants file saved to Downloads, Documents, or specific location
- File will be opened/viewed by user
- Interactive workflow (user-triggered action)

**Returns:**
```json
{
  "filename": "invoice_2025.pdf",
  "base64_content": "JVBERi0xLjQKJ...",
  "size_bytes": 245789,
  "mime_type": "application/pdf"
}
```

**Workflow:**
```python
# 1. Call MCP tool
result = download_attachment_base64(message_id, attachment_id, mailbox_id)

# 2. Write to laptop/workspace
Write(f"/home/iris/Downloads/{result['filename']}",
      base64_decode(result['base64_content']))
```

**Limitations:**
- Max file size: 25MB (Exchange Online limit)
- Base64 adds ~33% overhead in JSON response
- Not suitable for batch automation (use filesystem tools instead)

**Do NOT use when:**
- File will only be processed server-side
- Immediately calling extract_attachment_text afterwards
- Batch processing multiple attachments

---

### download_attachment (EXISTING)

**Purpose:** Download email attachment to container filesystem for server-side processing

**Use when:**
- Server-side automation workflows
- File will be processed immediately (extract_attachment_text)
- File doesn't need to reach user's laptop/workspace
- Batch processing email attachments

**Returns:**
```
"✅ Downloaded invoice.pdf to /root/.athena/attachments/1731789234_invoice.pdf"
```

**Workflow:**
```python
# 1. Download to container
result = download_attachment(message_id, attachment_id, mailbox_id)

# 2. Extract path from result string
path = extract_path_from_result(result)  # /root/.athena/attachments/...

# 3. Process immediately
text = extract_attachment_text(path)
```

**Limitations:**
- File stays in Docker container
- Not accessible from laptop/workspace without docker cp + scp
- Requires extract_attachment_text for text extraction

**Do NOT use when:**
- User wants file saved locally
- User will open/view the file
- File needs to go to laptop/workspace

---

### add_attachment_base64 (NEW)

**Purpose:** Attach file from laptop/workspace to draft email by passing base64 content

**Use when:**
- User requests: "attach this file from Documents"
- File exists on laptop/workspace filesystem
- Interactive workflow (user has the file)

**Parameters:**
```python
add_attachment_base64(
    message_id: str,
    filename: str,
    base64_content: str,
    mailbox_id: str = "thomas@sixpillar.co.uk"
)
```

**Workflow:**
```python
# 1. Read file from laptop/workspace
file_content = Read("/home/iris/Documents/report.pdf")  # Returns base64

# 2. Attach via MCP tool
add_attachment_base64(message_id, "report.pdf", file_content, mailbox_id)
```

**Limitations:**
- Max file size: 3MB per file (Graph API limit for simple upload)
- File must be readable by Claude
- Base64 encoding adds overhead

**Do NOT use when:**
- File already in container (use add_attachment_to_draft)
- File size > 3MB (requires chunked upload, not yet supported)

---

### add_attachment_to_draft (EXISTING)

**Purpose:** Attach file from container filesystem to draft email

**Use when:**
- File was just downloaded via download_attachment
- Server-side automation workflow
- File already exists in container

**Parameters:**
```python
add_attachment_to_draft(
    message_id: str,
    file_path: str,  # Container path: /root/.athena/attachments/...
    mailbox_id: str = "thomas@sixpillar.co.uk"
)
```

**Workflow:**
```python
# 1. Download to container first
result = download_attachment(msg_id_1, att_id, mailbox_id)
path = extract_path(result)  # /root/.athena/attachments/file.pdf

# 2. Attach to different draft
add_attachment_to_draft(msg_id_2, path, mailbox_id)
```

**Limitations:**
- File must already exist in container at specified path
- Cannot attach files from laptop/workspace directly
- Max file size: 3MB (Graph API limit)

**Do NOT use when:**
- File is on laptop/workspace (use add_attachment_base64)
- File doesn't exist in container yet

---

### extract_attachment_text (EXISTING)

**Purpose:** Extract text content from PDF/Word/Excel/PowerPoint files in container

**Use when:**
- File was downloaded via download_attachment
- Need to analyze document content
- Server-side automation (no user needs the file)

**Supports:**
- PDF (.pdf)
- Word (.docx)
- Excel (.xlsx)
- PowerPoint (.pptx)
- Text files (.txt, .md, .log, etc)

**Workflow:**
```python
# 1. Download to container
result = download_attachment(message_id, attachment_id, mailbox_id)
path = extract_path(result)

# 2. Extract text
text = extract_attachment_text(path, max_length=10000)

# 3. Process text (summarize, extract data, etc)
```

**Limitations:**
- File must be in container filesystem
- Max extracted text: 10,000 characters (configurable)
- Only supports listed file types

**Do NOT use when:**
- User wants the actual file (use download_attachment_base64 instead)

---

## Common Scenarios

### Scenario 1: "Download this invoice and save it to my Downloads folder"

**Correct approach:**
```python
result = download_attachment_base64(message_id, attachment_id, mailbox_id="thomas@sixpillar.co.uk")
Write(f"/home/iris/Downloads/{result['filename']}", base64_decode(result['base64_content']))
```

**Why:** User wants file on laptop → use base64 tool

**Wrong approach:** ❌ download_attachment (file stuck in container)

---

### Scenario 2: "Process all invoice attachments and summarize their contents"

**Correct approach:**
```python
for attachment_id in attachment_ids:
    result = download_attachment(message_id, attachment_id, mailbox_id)
    path = extract_path(result)
    text = extract_attachment_text(path)
    # Process text...
```

**Why:** Server-side automation, no user needs files → use filesystem tools

**Wrong approach:** ❌ download_attachment_base64 (wasteful base64 transfer, Claude would need to write temp files anyway)

---

### Scenario 3: "Attach the quarterly-report.pdf from my Documents folder to the draft"

**Correct approach:**
```python
content = Read("/home/iris/Documents/quarterly-report.pdf")  # Returns base64
add_attachment_base64(draft_message_id, "quarterly-report.pdf", content, mailbox_id)
```

**Why:** File on laptop → use base64 tool

**Wrong approach:** ❌ add_attachment_to_draft (expects container path, file not there)

---

### Scenario 4: "Forward this email with all its attachments to john@example.com"

**Correct approach:**
```python
forward_email(message_id, "john@example.com", mailbox_id)
```

**Why:** Attachments stay in Microsoft Graph, no download needed

**Wrong approach:** ❌ download_attachment + add_attachment_to_draft (unnecessary complexity)

---

### Scenario 5: "Download this contract, extract the text, and save the PDF to Documents"

**Correct approach:**
```python
# Get file content via base64
result = download_attachment_base64(message_id, attachment_id, mailbox_id)

# Save to laptop
Write(f"/home/iris/Documents/{result['filename']}", base64_decode(result['base64_content']))

# Extract text from base64 content (write temp file if needed)
# OR use container-based extraction if PDF parsing tools only available there
```

**Why:** User wants both the file AND the text → base64 tool gets both

**Alternative (if text extraction only works in container):**
```python
# Download to container for text extraction
result_fs = download_attachment(message_id, attachment_id, mailbox_id)
path = extract_path(result_fs)
text = extract_attachment_text(path)

# Also get file to laptop
result_b64 = download_attachment_base64(message_id, attachment_id, mailbox_id)
Write(f"/home/iris/Documents/{result_b64['filename']}", base64_decode(result_b64['base64_content']))
```

**Note:** Downloading twice is acceptable if one download is for user access and the other for server-side processing

---

## Decision Matrix

| User Intent | Destination | Tool to Use | Reason |
|-------------|-------------|-------------|--------|
| "Save invoice to Downloads" | Laptop | download_attachment_base64 | User wants file locally |
| "Extract text from all PDFs" | Server | download_attachment + extract_attachment_text | Server-side automation |
| "Attach report from Documents" | Email | add_attachment_base64 | Source is laptop |
| "Forward email with attachments" | Email | forward_email | No download needed |
| "Download and summarize contract" | Laptop + Server | download_attachment_base64 (can extract text from base64 too) | User wants file + analysis |

---

## Failure Modes

### Using download_attachment when user wants file locally
**Symptom:** File downloaded but user can't access it
**Fix:** Use download_attachment_base64 instead
**Prevention:** Look for keywords: "save", "download to", "my Downloads", "my Documents"

### Using add_attachment_to_draft with laptop file path
**Symptom:** "File not found: /home/iris/Documents/report.pdf"
**Fix:** Use add_attachment_base64 with Read(file) instead
**Prevention:** Check if file path is on laptop/workspace (not container)

### Using download_attachment_base64 for batch automation
**Symptom:** Works but wasteful (large JSON responses, temp files needed)
**Fix:** Use download_attachment + extract_attachment_text for server-side processing
**Prevention:** Look for automation keywords: "process all", "summarize attachments", "batch"

---

## File Size Limits

| Limit | Value | Applies To | Workaround |
|-------|-------|------------|------------|
| **Exchange attachment** | 25MB | All download tools | None - hard limit |
| **Graph API upload** | 3MB | add_attachment_*, send_email_with_attachments | Use chunked upload (not yet implemented) |
| **Context window** | **600-800 KB** | **download_attachment_base64** | **Use download_attachment + SSH workaround** |
| **Base64 encoding overhead** | +33% | Base64 tools | 1MB file → 1.3MB in context |

### Critical: Context Window Limitation

**download_attachment_base64 has a hidden limit:**
- Base64 content appears in Claude's context (consumes tokens)
- 1MB file ≈ 325K tokens in 200K context window = **FAIL**
- Practical limit: **500-800 KB max** for reliable operation
- Above this limit: Use download_attachment + SSH workaround

**Recommended workflow:**
1. Call list_attachments first to check size
2. If size > 500 KB: Use download_attachment (container filesystem)
3. If size < 500 KB: Use download_attachment_base64 (direct to laptop)

---

## Summary

**Use base64 tools when:**
- User wants file on laptop/workspace
- Interactive workflows
- File will be opened/viewed/edited by user

**Use filesystem tools when:**
- Server-side automation
- File processed immediately (text extraction)
- Batch operations
- User doesn't need the file

**When in doubt:**
- Ask: "Does the user want to see/access this file?"
- YES → base64 tools
- NO → filesystem tools

---

**Last Updated:** 2025-11-16
