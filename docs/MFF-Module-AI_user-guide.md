# AI module: user guide

The AI tab is a chat with the AI model that runs on the facility's own GPU server. Nothing you write leaves the facility.

## Before you start
- You need access to the AI module, and "workspace storage" ticked on your account (Users & Groups). Without it the page says so: your conversations are kept in your own server storage.
- Administrators cannot read your chats through MFF: titles, texts and files are only in your own storage, not in the database.

## Chatting
- **Enter** sends, **Shift+Enter** makes a new line. **Stop** ends an answer; what was written so far is kept.
- **Think first** (Qwen): slower, better for hard questions. The thinking is shown separately; click it to read it.
- Qwen accepts conversations of up to 262,144 tokens (about 196,000 words). Administrators can load Qwen in a smaller FP8 form; it answers almost twice as fast, and more long conversations fit at the same time.
- You can leave or reload while an answer is being written: it goes on, and the page picks it up again.
- **Answer again** replaces the last answer. **Edit** changes your last question and asks again.
- Chats are listed by date, with search in titles and text. **…** next to a chat renames or deletes it. Deleted chats can be restored from "Deleted" until they are removed for good (30 days unless changed on Setup).

## Files
- Attach, drag in or paste: PDF, Word (.docx), Excel (.xlsx), PowerPoint (.pptx), text files (code, CSV, JSON, ...) and PNG, JPEG or WebP images. The original is kept unchanged; click it to download it.
- A long file is read only as far as it fits; the answer says so. If the tool "Read attached file" is installed, the AI can search the rest.
- Not supported: scanned PDFs (upload the pages as images), old .doc/.xls files, dates in Excel (they show as day numbers). If the tools are installed, the AI can change an attached Word or Excel file, or make a new file, and gives it to you as a download. The original is never changed: each edit is a new file (version 2, 3, ...). Check the result: a tool that did not work is marked in red above the answer. Formulas are recalculated when the file is opened in Excel; formulas that reach outside the file are refused.

## Skills
Your facility can write skills: short instructions for particular tasks, for example how a buffer recipe should be written here. When your question matches a skill, the AI loads it and follows its steps; the chat shows this as "Used a tool: Load skill". Only the Qwen model uses skills; with another model loaded, the AI answers without them. Skills are written and managed by administrators and developers on Setup -> Skills.

## Good to know
- The AI can make mistakes: check important answers.
- One answer at a time per person, and 30 questions per 10 minutes (defaults). When many people ask at once, questions wait in a queue and the page shows your place.
- When the model is being switched or a benchmark runs, a banner says so; questions wait or can be sent again afterwards.
