---
name: chatgpt-page-batch-qa
description: Batch automation for asking many questions through an existing ChatGPT web conversation page from Codex, waiting for each answer, extracting question/answer pairs, saving Markdown and JSONL locally, and resuming failed or interrupted runs. Use when the user explicitly needs browser-controlled ChatGPT page automation instead of the OpenAI API.
---

# ChatGPT Page Batch QA

Use this skill when the user must automate the ChatGPT web page itself from Codex. Prefer the Chrome plugin for an already logged-in ChatGPT page or any `chatgpt.com/c/...` tab. Use the in-app Browser only when the page is actually open there.

## Inputs

Accept questions from one of these sources:

- Plain list in the user message.
- Local `.txt`, `.md`, `.csv`, `.json`, or `.jsonl` file.
- A table-like file where each row contains at least `id` and `question`.

Normalize into records:

```json
{"id":"q001","question":"Can DVD video be enhanced?","status":"pending"}
```

Keep run state in JSONL. Save one record immediately after each completed answer:

```json
{"id":"q001","question":"Can DVD video be enhanced?","answer":"...","status":"done","url":"https://chatgpt.com/c/...","created_at":"2026-06-01T14:42:00+08:00"}
```

Questions and answers may be in any language, including Chinese. The skill file itself is ASCII-only so local validation scripts work reliably on Windows.

## Workflow

1. Confirm or locate the target ChatGPT tab.
   - If the user gives a `chatgpt.com/c/...` URL, find that exact open tab first.
   - If no URL is given, list open ChatGPT tabs and choose the most recent relevant one.
   - Do not reload the page unless the page is broken or the user asks.

2. Claim the tab with the browser automation backend.
   - For Chrome, use the Chrome plugin and claim the user tab returned by `browser.user.openTabs()`.
   - Keep the tab as `handoff` when finalizing so the user can continue from the page.

3. For each pending question:
   - Wait until the ChatGPT input textbox is available.
   - Fill only the current question.
   - Send with Enter or the Send button.
   - Wait for completion using page state, not only a fixed delay.
   - Extract the newest assistant answer.
   - Append the result to JSONL immediately.
   - Regenerate or append the Markdown report.

4. On failure:
   - Mark the record as `failed`.
   - Save the error message and timestamp.
   - Continue with the next question only if the page is still usable.
   - Never discard completed results.

## Completion Detection

Use a combination of these signals:

- The stop-generating control disappears.
- The send button or textbox becomes usable again.
- The newest assistant message text is unchanged for two consecutive checks, 2-5 seconds apart.
- A per-question timeout is reached.

Default timeout: 180 seconds per question. Use a longer timeout for complex research questions.

Do not start the next question while the current answer is still streaming.

## Extraction

Prefer structured DOM extraction over screen scraping. Extract messages from elements with message role metadata when available, for example `data-message-author-role="user"` and `data-message-author-role="assistant"`.

For each run, preserve:

- `id`
- `question`
- `answer`
- `status`
- `conversation_url`
- `created_at`
- `model_or_mode` when visible
- `error` when failed

When exporting Markdown, use this shape:

```md
# ChatGPT Batch QA

Source: https://chatgpt.com/c/...

## Question 1

...

## Answer 1

...
```

Convert visible tables to Markdown tables. Convert code blocks to fenced code blocks. Preserve links when practical.

## File Naming

Default output names in the current workspace:

- `chatgpt-batch-qa.jsonl`
- `chatgpt-batch-qa.md`

If the source conversation has a clear topic, use a descriptive slug such as:

- `chatgpt-old-movie-qa.jsonl`
- `chatgpt-old-movie-qa.md`

Avoid overwriting existing files unless the user asks. If a file exists, append a timestamp or continue from existing JSONL state.

## Resume Rules

Before running, inspect existing JSONL state if present.

- Skip records already marked `done`.
- Retry records marked `failed` only when the user asks or when the failure was a temporary page/network issue.
- Preserve original IDs.
- If the same question appears twice with different IDs, treat them as separate records.

## Safety

This skill controls a logged-in web page. Keep actions limited to typing user-provided questions, sending them, reading answers, and saving local output.

Ask for confirmation before:

- Uploading files.
- Sharing or publishing the conversation.
- Sending private, credential-like, or third-party confidential data.
- Clicking links outside ChatGPT.
- Deleting or modifying existing local result files.

## Practical Browser Notes

Use a fresh DOM snapshot after sending a question and after each major page state change. If a locator fails, take a new snapshot before trying another locator.

Common stable targets:

- ChatGPT input textbox: role `textbox`, often named `Chat with ChatGPT`.
- User and assistant messages: message containers with author role metadata.
- Send behavior: pressing `Enter` in the focused textbox is usually sufficient.

Finalize browser control at the end with the ChatGPT tab kept as `handoff`.
