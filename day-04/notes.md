# Day 4 — LLMs, Tokens, Model Types, and Hallucinations

## Lesson 1: What Is an LLM

A large language model (LLM) does one thing: given some text, it predicts the next chunk of text. It was trained by reading an enormous amount of writing - books, code, articles, forums - and learning the patterns in how words follow one another. When you send it a question, it isn't looking anything up. It's generating the most plausible continuation of what you wrote, one chunk at a time.

That single idea explains almost everything about how these models behave, so it's worth holding onto: an LLM produces text that sounds right, which is not the same as text that is right.

### What Follows From "Predict the Next Chunk"

A few consequences fall straight out of how the model works, and they shape how you use it:

- **It has a knowledge cutoff.** The model only knows what was in its training data. Ask about something that happened after it was trained, or about a private repo it never saw, and it has nothing to draw on.
- **It doesn't fetch live data on its own.** No web search, no database, no clock, unless a tool is wired up to give it those. On its own it only has the text you send plus what it learned in training.
- **It isn't deterministic.** Send the exact same prompt twice and you can get two different answers. There's deliberate randomness in how it picks each next chunk.
- **It can be confidently wrong.** Because it generates plausible text, a wrong answer looks just as fluent and self-assured as a right one.

### What It's Genuinely Good At

None of that makes the model weak - it makes it a specific kind of tool. LLMs are strong at working with language: drafting and rewriting, summarizing, explaining, translating, turning messy notes into structure, writing and reviewing code, and answering questions about text you hand it.

They're weak wherever fluent-sounding text isn't good enough on its own: exact arithmetic, counting, and any fact you can't afford to have invented. For those, you give the model the real data to work from or you check its output. The rest of this skill is largely about doing exactly that well.


## Lesson 2: Tokens

Models don't read text as letters or whole words. They read tokens - short chunks of text. A token is roughly three quarters of an English word, so "Explain DNS in one sentence" is about seven tokens. Common words are often a single token; rare words and code get split into several.

Tokens are the unit almost everything about a model is measured in, so it's worth getting a feel for them early.

### Input Tokens and Output Tokens

Two amounts matter, and they are counted separately:

- **Input tokens** - everything you send the model: your question, plus any text or history you include with it.
- **Output tokens** - everything the model writes back.

A short question with a short answer is only a handful of tokens each way. Paste in a long document and the input tokens climb into the thousands. Ask for a long essay and the output tokens climb instead. Every reply comes with a count of how many tokens went in and how many came back, so you can always see the size of an exchange. You will see those exact numbers for yourself once you start making requests.


## Lesson 3: Why Tokens Matter

Tokens aren't just a technical detail about how models read text. They're the meter. Almost every modern LLM is priced by the token, so the number of tokens a request uses is the number that shows up on the bill. Understand tokens and you understand both what a model can do and what it takes to run one.

### Pricing Is Per Token, Both Directions

You aren't charged per question or per request. You're charged per token, and input and output are counted separately:

- **Input tokens** - everything you send: your prompt, any documents or data you paste in, and the earlier conversation.
- **Output tokens** - everything the model generates back.

Output tokens usually cost more than input tokens, because generating text is the expensive part. The exact rates change constantly and differ by model, so the specific numbers matter less than the shape: two requests that produce the "same" answer can use wildly different token counts, and pay accordingly.

### Where the Tokens Pile Up

A few everyday patterns quietly multiply your token count:

- **A quick question.** A one-line prompt and a short answer might be a few dozen tokens total. Cheap, fast.
- **Pasting a big document.** Drop a long report into the prompt and you're sending thousands of input tokens - on every single request that includes it, not just once.
- **A long back-and-forth.** Models have no memory between calls, so a chat app re-sends the entire conversation each turn. Turn twenty pays for turns one through nineteen all over again. The conversation gets more expensive the longer it runs.
- **A long answer.** Asking for a full essay when a sentence would do spends output tokens you didn't need.

### See It With Claude

The claude command is already wired up, and its JSON output reports exactly how many tokens a call used. Ask the same question two ways - once wide open, once with a limit - and watch the output tokens:

```bash
claude -p "Explain what is DNS" --output-format json | jq '.usage.output_tokens'
claude -p "Explain what is DNS in one sentence" --output-format json | jq '.usage.output_tokens'
```

The first prompt is open-ended, so the model rambles on and spends a lot of output tokens. The second adds three words - "in one sentence" - and the count drops sharply. Same question, but how you phrase the ask decides how many tokens the answer costs. Look at the whole usage block to see both directions at once:

```bash
claude -p "Explain what is DNS in one sentence" --output-format json | jq '.usage | {input_tokens, output_tokens}'
```

`input_tokens` is what you sent, `output_tokens` is what came back. Keep an eye on those two numbers as you write prompts and you'll quickly build an instinct for what's expensive and what isn't.

### The Levers You Control

Once you see requests in tokens, the ways to keep them lean are obvious, and they're the same skills that make answers better:

- Send only the context that's relevant, not the whole file.
- Ask for a short answer in the prompt and cap generation with `max_tokens`, leaving room for reasoning tokens as well as visible text.
- Summarize or trim old turns in a long conversation instead of dragging the full history along.
- Reach for a smaller model when the task doesn't need the biggest one.

This is why the prompting and context topics later in this skill aren't only about getting a better answer. A tighter prompt is also a cheaper, faster one.

## Lesson 4: Open Models vs. Frontier Models

The models you'll meet fall into two broad camps, and knowing which is which helps you pick the right one and understand the tradeoffs.

### Frontier Models

Frontier models are the biggest, most capable models, built by a handful of well-funded labs - think OpenAI's GPT, Anthropic's Claude, and Google's Gemini. Their weights are closed: you can't download them or run them yourself. You rent access through an API. They usually sit at the top of the benchmarks, and they usually carry the highest price per token.

### Open Models

Open models (also called open-weight models) publish their weights, so anyone can download them, run them on their own hardware, and fine-tune them. Meta's Llama, DeepSeek, Zhipu's GLM, Mistral, and OpenAI's gpt-oss are examples.

The five open models you will use in this course are:

- `deepseek-v4-flash`
- `deepseek-v4-pro`
- `glm-5.2`
- `gpt-oss-120b`
- `minimax-m3`

You do not need to know their differences yet - a later topic covers when to reach for which. For now, just know these are the names you will pick from in the hands-on tasks ahead.

Because you can run them yourself, open models tend to be cheaper to operate, can run offline or inside your own network, and don't lock you to one vendor. The gap to the frontier has been closing quickly - for a great many real tasks, a good open model is more than enough.

### How to Think About the Choice

Neither camp is simply "better." It's a tradeoff:

- **Frontier** - reach for peak capability on the hardest tasks, and the convenience of a managed API, when the higher cost is worth it.
- **Open** - reach for control, privacy (the data can stay on your own infrastructure), lower cost, and no vendor lock-in.

The good news for learning: the request shape is the same either way. Everything you practice here against open models works unchanged if you later point it at a frontier model. A whole topic later in this skill is devoted to choosing between the specific models available to you for a given task.

## Lesson 5: Hallucinations and Limits

A hallucination is when a model states something false as if it were true. Not a glitch, not a rare bug - it's the same mechanism that makes the model useful, seen from its bad side. The model generates plausible text. Most of the time plausible and correct line up. When they don't, you get an answer that reads perfectly and is simply wrong.

Because the wrong answer is just as fluent and confident as a right one, you can't spot a hallucination by tone. This is the single most important thing to internalize before you lean on AI for real work.

### Where Hallucinations Show Up

A few patterns come up again and again:

- **Invented specifics.** Function names, command flags, API endpoints, or library methods that sound right but don't exist.
- **Fake sources.** Citations, URLs, book titles, and quotes that were never real. The model knows what a citation looks like, so it produces one.
- **Confident guesses about things it can't know.** Anything after its training cutoff, or anything private it never saw, gets filled in with a convincing guess rather than an "I don't know."

### The Other Hard Limits

Beyond hallucination, two limits are worth naming plainly:

- **No live knowledge.** Out of the box the model has no access to the current date, the web, or your systems. If you don't give it a fact, it's working from training data that has a cutoff.
- **Weak at exact math and counting.** It predicts text, so arithmetic, tallying items, and precise character counts are shaky. Use a real tool for those.

### How to Work With This, Not Against It

You don't avoid hallucinations by hoping the model is careful. You design around them:

- **Give it the facts.** When the answer depends on specific information, put that information in the prompt instead of trusting the model's memory. A later topic is entirely about grounding answers in your own data.
- **Ask for sources, then check them.** Useful, but only if you actually open the source. A cited link that 404s is a tell.
- **Verify anything that has a cost if it's wrong.** Treat AI output as a fast first draft from a sharp but unreliable colleague, not as a final answer.

That mindset - use it for speed, keep your hand on the wheel - is what separates people who get real value from AI from those who get burned by it.
