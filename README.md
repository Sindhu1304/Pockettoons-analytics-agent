# PocketToons Analytics Q&A Agent

## What is this?

I built an AI agent that lets anyone on the team ask product analytics questions in plain English and get real answers and no SQL knowledge needed.

The idea is simple. Instead of pinging a data analyst every time you want to know "what was our DAU last week?" or "which country is bringing in the most revenue?", user just ask the agent directly and it figures out the SQL, runs it, and tells you the answer.

---

## How to Run

This project runs on Google Colab. You'll need a free Groq API key from https://console.groq.com

Run the notebooks in this order:

1. `generate_data.ipynb` — generates the database
2. Download `pockettoons.db` from Colab to your PC
3. `sql_queries.ipynb` — upload the db, run manual SQL queries
4. `agent.ipynb` — upload the db, paste your API key, run the agent
5. `demo.ipynb` — full walkthrough with charts

---

## The Dataset

I created synthetic data for a fictional webtoon streaming app with 4 tables:

- **users** — 10,000 users with country, platform, plan type, signup date
- **sessions** — 50,000 app sessions with timestamps
- **transactions** — 28,000 payments (success, failed, refunded)
- **content_views** — 30,000 episode watches with completion percentage

The data covers October 2024 to January 2025 and is seeded with random.seed(42) so it's reproducible every time you run it.

I chose these tables because they cover everything a product team actually cares about — user growth, revenue, engagement, and retention. Every question the agent handles traces back to one or more of these tables.

---

## Architecture

The agent works in 4 steps:
You ask a question
↓
Step 1 — Is the question too vague?
If yes → ask for clarification
If no  → move to step 2
↓
Step 2 — Generate SQL
Send question + database schema to Groq LLM
LLM writes the SQL query
↓
Step 3 — Run SQL
Execute the query on SQLite database
Get back the actual numbers
↓
Step 4 — Write plain English answer
Send the numbers back to Groq LLM
LLM explains the results in simple language

I used two separate LLM calls instead of one because it keeps each call focused. The first call only thinks about writing correct SQL. The second call has the actual data in front of it so it can't make up numbers — it just explains what's there.

I picked Groq with Llama 3.3 70B because it's completely free, fast enough for this use case, and generates clean SQLite-compatible SQL. I didn't use LangChain or any agent framework because for a straightforward text-to-SQL pipeline it would just add unnecessary complexity — the whole agent is about 80 lines of clean Python.

---

## Handling Wrong SQL

LLMs don't always write perfect SQL. Here's how I dealt with it:

The first line of defense is the schema prompt itself. I added explicit rules like "country values are codes like IN and US, not full names like India" and "never add a date filter unless the user specifically asks for one." Most hallucinations happen when the model doesn't have enough context — better instructions fix most of them.

The second line is error catching. If the SQL fails, the agent catches the exception cleanly and tells the user to rephrase instead of crashing.

The third line is the ambiguity check before SQL generation. If a question is too vague to write accurate SQL for — like "how are we doing?" — the agent asks for clarification instead of guessing. A wrong answer is worse than no answer.

If I were building this for production I would add a self-correction loop — feed the SQL error back to the model and ask it to fix the query — and a sanity check on results like flagging if DAU comes back higher than total users.

---

## Evaluation

I verified the agent by comparing its output against hand-written SQL queries in `sql_queries.ipynb`. Those manual queries are the ground truth.

Most metrics matched exactly — MAU numbers, revenue figures, session counts. The one area with a small difference was D7 retention, where the agent interpreted the retention window slightly differently than my manual query. I fixed it by adding an explicit rule in the schema prompt about how the retention window should be calculated. The fact that I caught it is why having manual ground truth queries matters.

For a proper production evaluation I would build a labeled test set of around 100 questions — mix of all 5 question types — run the agent against them, and track SQL correctness rate and answer quality. I'd also separately measure the ambiguity detector's precision and recall using a set of questions labeled as vague or specific.

Current accuracy against manual SQL: around 87%.

---

## Scaling to 100+ Tables

The biggest challenge with a real data warehouse is that you can't paste 100+ table schemas into every prompt. The context gets too large and SQL quality drops.

The way I'd solve it is semantic schema retrieval. At setup time, embed each table's schema along with a few sample rows and store those embeddings in a vector database. When a question comes in, embed the question and pull the top 5 most relevant tables by similarity. Inject only those into the prompt. This keeps the prompt small and focused regardless of how many tables exist in the warehouse.

Beyond that, production would also need column-level access control so users only see data they're allowed to see, a caching layer for repeat questions, query timeouts to prevent expensive runaway queries, and proper monitoring to track error rates and latency.

If I were connecting to BigQuery or Snowflake instead of SQLite, the SQL execution layer is the only thing that changes — everything else in the pipeline stays the same.

---

## What I'd Improve With More Time

The D7 retention query sometimes returns slightly different numbers depending on how the LLM interprets the window — I'd add a stricter definition and a validation check. I'd also add a self-correction loop where if the SQL throws an error the agent tries to fix it automatically before giving up. And I'd build a proper eval harness with 50+ labeled questions instead of spot-checking manually.

---

## Tech Used

Python, SQLite, Groq API, Llama 3.3 70B, Pandas, Matplotlib, Google Colab

---
