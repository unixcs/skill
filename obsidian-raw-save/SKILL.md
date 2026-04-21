---
name: obsidian-raw-save
description: Save the previous assistant reply into the user's Obsidian inbox. Use when the user explicitly says `原文保存` or `原文保存：<主题>`.
---

# Obsidian Raw Save

When triggered, save the previous assistant reply to the user's Obsidian inbox as the final confirmed version.

Rules:
- Use this root directory: `C:\Users\fengx\onedrive\Apps\remotely-save\feng\00_Inbox\电脑\`
- Prefer today's date directory named `YYYY-MM-DD`
- If today's date directory does not exist, create it first
- Save the note inside today's date directory
- Only trigger when the user explicitly says `原文保存` or `原文保存：<主题>`
- Always use the previous assistant reply as the content source
- If the user provides `原文保存：<主题>`, use `<主题>` as the filename topic
- If the user only says `原文保存`, infer a short, concrete topic from the previous assistant reply
- Create a new file every time
- Never append to or overwrite an existing note
- If the filename already exists, generate a new unique filename

Content rules:
- Preserve the previous assistant reply as the final confirmed content
- If it is already Markdown, save it directly
- If it is not Markdown, only convert it into Markdown format before saving
- Do not rewrite, summarize, expand, shorten, polish, or add new content
- Do not add titles, metadata, timestamps, frontmatter, or extra sections unless they already exist in the original reply
- The body must remain as close as possible to the original reply
- Markdown conversion may only change presentation, not meaning

Failure handling:
- If there is no previous assistant reply, do not save
- If the previous assistant reply is clearly not the intended final text, ask the user to confirm before saving
- If the previous assistant reply is only a brief acknowledgment, suggestion, or transition message, do not save it directly; ask the user to confirm first

Goal:
- Faithfully save the confirmed final version, not generate a new summary
