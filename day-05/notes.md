# Day 5 — Calling an LLM Over an API

## Lesson: Why Call an LLM Over an API

So far it's been easy to picture AI as a chat window: you type, it replies. That works for one question at a time. But most real AI work doesn't happen in a chat window — it happens in code, through an API.

An **API** (application programming interface) is just a way for one program to talk to another. An LLM API lets your program send text to a model and get the reply back, with no human typing in a box. That one capability is what turns a model from a novelty into a tool you can build on.

### What the API Unlocks

Three things become possible the moment you can call a model from code:

- **Automation.** Run the same task across a thousand items instead of pasting them one by one. Summarize every support ticket from yesterday, classify a folder of log files, rewrite a hundred product descriptions — one loop, not a hundred copy-pastes.
- **Integration.** Put the model inside your own scripts, apps, and pipelines. A deployment script that writes its own release notes, a monitor that explains an alert in plain English, a form that tags itself.
- **Control.** In code you set exactly which model answers, how long the reply can run, and how the request is shaped — the same way every time, so the behavior is repeatable.

### The Chat App Is Just Another API Client

Here's the part worth sitting with: every AI product you've ever used — the chat assistants, the coding tools, the "summarize this" buttons — is a program making these same API calls under the hood. The chat window is one client. Once you know how the call works, you can write your own.

---

## Task 1: Your First Request With `curl`

**Goal:** Send one API call with `curl` — no SDK, no framework — and save the raw JSON response to `/root/first-reply.json`.

### Your Key and URL

The API key and base URL are already set in the VM as environment variables, `$OPENAI_API_KEY` and `$OPENAI_BASE_URL`, so the key is never typed by hand:

```bash
echo $OPENAI_BASE_URL
```

### Send the Request

```bash
curl -s "$OPENAI_BASE_URL/v1/chat/completions" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "minimax-m3",
    "messages": [{"role": "user", "content": "Explain what an API is in one sentence."}]
  }' > /root/first-reply.json
```

**Line by line:**

- `curl -s <url>` — send an HTTP request. `-s` (silent) hides curl's progress meter so only the server's response comes out.
- `"$OPENAI_BASE_URL/v1/chat/completions"` — before curl even runs, the shell swaps `$OPENAI_BASE_URL` for the real server address. Same trick on the key line — the key is never typed by hand.
- `-H "Authorization: Bearer $OPENAI_API_KEY"` — `-H` adds one HTTP header. This one carries your identity: "Bearer" means whoever carries this key.
- `-H "Content-Type: application/json"` — a second header telling the server "the body I'm sending is JSON."
- `-d '{...}'` — the request body: which model, and your one-message conversation. Each message is a `role` (who's speaking — `user` is you, `assistant` is the model) paired with its `content`. (`-d` is also what turns this into a POST.)
- `\` at each line end — "this command continues on the next line."
- `> /root/first-reply.json` — the shell redirects curl's output into that file instead of your screen. That's why you see nothing when it works.

### Reading the Reply

`jq` is a JSON viewer: give it a path and it digs out just that value.

```bash
jq . /root/first-reply.json
```

`.` means "the whole document, pretty-printed." Find the reply text and the token counts, then pluck each one directly:

```bash
jq -r '.choices[0].message.content' /root/first-reply.json
jq .usage /root/first-reply.json
```

- `.choices[0].message.content` — the path to memorize: first choice, its message, its content. `-r` (raw) prints the string without JSON quotes.
- `.usage` — how many tokens this one exchange used.

### Check Your Understanding

- Which header carried your identity?
- Where in the JSON is the model's actual reply?
- What would happen if you added `"max_tokens": 5` to the request? (Try it — but don't overwrite the saved file: drop the `>` redirect.) Reasoning also consumes that budget, so `content` may be `null`; inspect `finish_reason` and `usage` instead of treating an empty reply as success.

---

## Task 2: Send Your Own Request

**Goal:** Ask a model a question and save its answer.

Send one API call to any of the five available models, ask it to explain a topic of your choice in two sentences, and save just the reply text — not the raw JSON — to `/root/basics/summary.txt`.

```bash
mkdir -p /root/basics
curl -s "$OPENAI_BASE_URL/v1/chat/completions" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "minimax-m3", "messages": [{"role": "user", "content": "Explain what DNS does in two sentences."}]}' \
  | jq -r '.choices[0].message.content' > /root/basics/summary.txt
```

Swap in your own question, and pick any of the five models (`deepseek-v4-flash`, `deepseek-v4-pro`, `glm-5.2`, `gpt-oss-120b`, `minimax-m3`). The `jq -r '.choices[0].message.content'` step is what pulls the reply text out of the JSON, so the file holds plain text.

Keeping the reply to two sentences also keeps the model's output focused — a good habit when only a short answer is needed.

---

## Task 3: Try a Different Model

**Goal:** Ask the same question of two different models.

The `model` field in the request decides which model answers. Swap it and the same question goes to a completely different model.

Send one question to two different models, saving just the reply text to `/root/api/reply-1.txt` and `/root/api/reply-2.txt`:

```bash
mkdir -p /root/api
curl -s "$OPENAI_BASE_URL/v1/chat/completions" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "minimax-m3", "messages": [{"role": "user", "content": "Explain what is DNS"}]}' \
  | jq -r '.choices[0].message.content' > /root/api/reply-1.txt
```

Run it again for the second file, changing only the `model` field and the output path:

```bash
curl -s "$OPENAI_BASE_URL/v1/chat/completions" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "glm-5.2", "messages": [{"role": "user", "content": "Explain what is DNS"}]}' \
  | jq -r '.choices[0].message.content' > /root/api/reply-2.txt
```

Reading both files afterward shows how the answers differ, even for the exact same prompt.

---

## Task 4: Understand `max_tokens` Limits

**Goal:** Understand a reply's token budget.

`max_tokens` limits generated tokens, including reasoning tokens on the models in this lab. It does **not** guarantee that many tokens of visible text — a tiny budget can run out before the model writes any answer at all.

### 1. Observe the Cap

Ask for a detailed explanation with a 20-token budget, and save the full JSON so the reason generation stopped can be inspected:

```bash
mkdir -p /root/api
curl -fsS "$OPENAI_BASE_URL/v1/chat/completions" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "minimax-m3", "max_tokens": 20, "messages": [{"role": "user", "content": "Explain in detail how DNS resolution works"}]}' \
  > /root/api/capped-response.json

jq '{finish_reason: .choices[0].finish_reason, completion_tokens: .usage.completion_tokens, content: .choices[0].message.content}' /root/api/capped-response.json
```

`finish_reason: "length"` means the budget stopped generation. `completion_tokens` should be at most 20. `content` may contain a fragment or be `null` — the model can spend all 20 tokens on reasoning. That's evidence of an exhausted budget, not a usable answer. Saving the word `null` into a text file does not make it a reply.

### 2. Get a Usable Short Answer

Ask for one sentence, and leave room for reasoning with a 2,048-token budget. The prompt requests brevity; the cap limits total generation. This is a ceiling, not a target — the model can finish using fewer tokens than that.

```bash
curl -fsS "$OPENAI_BASE_URL/v1/chat/completions" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "minimax-m3", "max_tokens": 2048, "messages": [{"role": "user", "content": "Explain how DNS resolution works in one sentence."}]}' \
  | jq -er '.choices[0].message.content | strings | select(test("\\S"))' > /root/api/short-answer.txt

cat /root/api/short-answer.txt
```

The `jq -e` filter fails if the response has no visible text, instead of writing `null` as an answer. If it fails: an API error needs fixing; `finish_reason: "length"` with no text needs more token budget or a simpler request. Reasoning needs vary, so no fixed budget guarantees an answer for every prompt.

## Key Takeaways

- Every AI chat product is, underneath, just another client calling the same kind of API.
- `curl` + `jq` is enough to call a model and extract exactly the field needed from its JSON response — no SDK required.
- `max_tokens` caps *total* generation (including hidden reasoning tokens on these models), not just the visible reply — a small cap can produce an empty answer even though the request itself succeeded.
- `finish_reason` is the field that explains *why* generation stopped (`"length"` = ran out of budget), and it's the first place to look when a reply comes back empty or truncated.

---
*Generative AI Course*
