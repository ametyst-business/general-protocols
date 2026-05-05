# Notion Content Rules

## MANDATORY — applies to every Notion write operation

### Format
All page body content MUST be written as a single `code` block with `language: "markdown"`. Never convert content to Notion-native blocks (paragraphs, headings, bullets, etc.) unless the user explicitly asks for it.

### Verbatim content
When pushing a local file to Notion, the content MUST be the **verbatim exact copy** of the source file. Read the file and use its content as-is. Do NOT reconstruct, rephrase, summarize, or shorten the content under any circumstance. If the content on Notion does not match the local file byte-for-byte, it is wrong.

### Character limit
Each `rich_text` content item has a 2000-character limit. If the content exceeds 2000 characters, split it into multiple `rich_text` items within the same `code` block. Split at natural line breaks — never mid-sentence or mid-word.

### Verification
After writing, the content on Notion must be identical to the source file. If you are unsure, re-read the Notion page and compare against the local file.
