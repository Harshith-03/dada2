# DADA2 — Agentic RAG over SEC filings

**Fill in both sections at the bottom before you submit.** Keep the deployed
app line exactly in this form; it is how your app gets found and graded.

## Getting started

There is no separate spec. The instructions are in the notebooks, and
`DADA2_00_Start_Here.ipynb` is the one to open first: it explains the
assignment, the setup, the grading, and what each of the four notebooks after
it asks for. Work through them in order.

Get the files the same way as every class day: pull, then work in a copy:

```bash
cd lecture-files
git pull
cd ..
cp -r lecture-files/DADA2 DADA2     # work in the copy, not in lecture-files/
cd DADA2
cp example.env .env                  # then fill in your keys
```

Then open `DADA2_00_Start_Here.ipynb` from your `DADA2` copy and run its
setup check.

Deploying is `DEPLOY.md`.

---

## Submission

```
Deployed app: https://dada2-hvr.onrender.com
```

ADD YOUR NAME TO YOUR RENDER PROJECT SO THE LINK CAN BE PERSONALIZED.

PLEASE ENSURE YOUR URL WORKS BY TESTING IT IN INCOGNITO MODE.

### WHAT TO SUBMIT
Upload the entire project folder (DADA2) via GitHub integration on Canvas

### Write-up

Paste the write-up from notebook 04 (TODO 4.3) here, under the same five
headings.

**Conditions.** Chunks of 1,200 characters with 150 overlap (2,124 chunks from 27 section files), embedded with qwen3-embedding-0.6b on Cloudflare Workers AI (1,024 dimensions) into the Pinecone index `dada2-1024`. Retrieval returns k=5 passages; the reranker is Cohere `rerank-v3.5` over a pool of 10; RRF uses k=60. The chat model is `gemini-3.5-flash-lite`, used for filter extraction, query rewrites, planning, answers and judging. The decision model is Cloudflare's `clef-flash`. The router comparison used all 24 labeled questions; the agent comparison used 14 (8 specific, 6 comparative).

**The routers.** The decision model got 20/24 right at 0.85 s per call; the chat model got 19/24 at 0.72 s. That is one question of accuracy in the decision model's favour and 0.13 s of speed in the chat model's, so neither result is a meaningful win. What the decision model is for is not accuracy or speed: it returns a probability for every option, which is what lets `route` fall back to `filter+rrf` below 0.6 and `diagnose` refuse an `absent` below 0.8. It also costs nothing against the 500-a-day Gemini limit, which mattered in practice in this assignment.

**The agents.** The ladder scored 0.93 (13/14) at 4.1 chat calls and 13.0 s per question. The diagnosis agent scored 0.79 (11/14) at 2.6 chat calls, 2.3 decision calls and 9.4 s per question: about 37% fewer chat calls and 28% less time. Re-judging the diagnosis agent on the last ten questions found two misses, not three, so on 14 questions the judge itself moves the score by about one question. The two misses are different in kind. On NVIDIA's fiscal 2026 H20 charge, the judge's reason was that fiscal 2026 "has not occurred": it graded against its own training knowledge instead of the filing, which is a flaw in the evaluation rather than the agent. On NVIDIA's fiscal 2024 target markets, the answer said the passages did not cover the topic, though the business section does. The mechanism is in my code: the diagnosis agent takes the first stopping verdict after one filtered search and only searches again if the verdict says to, while the ladder climbs to a stronger search on every failure. Spending fewer searches is the saving, and a stop after one weak search is its price.

**What did not help.** Multi-query fusion (`filter+rrf`). In notebook 02's sample, the filter alone scored recall [R1] and MRR [M1], and fusion scored [R2] and [M2], for an extra chat call per question. The reason is the filter: once it restricts the search to one filing, every rewrite searches the same few dozen chunks, so the lists mostly agree, and RRF can only reshuffle passages that were already found, which can move the one right passage down a place.

**What I would ship.** The diagnosis agent, with two changes. First, accept `sufficient` only above a probability threshold and send anything below it to `stronger`, the way `absent` is already second-guessed, so a single weak search no longer ends the question. Second, give the judge the gold file instead of letting it grade from memory, since a judge that does not believe fiscal 2026 exists cannot score questions about it. Before setting the threshold I would want to know the mix of my users' questions: the filter wins on questions naming one filing and loses on comparative ones, so the share of comparative questions decides whether early stopping is a cheap shortcut or the main source of wrong answers.