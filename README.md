


# PocketToons Analytics Q&A Agent

## What I Built and Why

I built an AI agent where anyone on the team can just type a question like 
"what was our DAU last week?" and get a real answer with actual numbers , 
without writing a single line of SQL.

The problem I was solving is simple. In most companies, if a product manager 
wants a quick number, they have to ask a data analyst, who writes SQL, runs it, 
and sends back the result. That whole process takes time. This agent cuts out 
the middle step — you ask in plain English, it figures out the SQL itself, 
runs it, and gives you the answer.

I picked Option B because it felt the most real and directly useful for a 
product analytics role.

---

## How to Run It

You need Google Colab and a free Groq API key from https://console.groq.com

Run notebooks in this order:

1. `Generate_data.ipynb` — creates the database
2. Download `pockettoons.db` and save it to your PC
3. `Analytics_queries.ipynb` — manual SQL queries, upload the db here
4. `Ai_agent.ipynb` — the AI agent, upload db and paste your API key
5. `demo.ipynb` — full walkthrough showing everything working

> API key is removed from notebooks for security. 
> Add your own free key from https://console.groq.com

---



## The Dataset

I created synthetic data for a fictional webtoon streaming app with 4 tables:

- **users** — 10,000 users with country, platform, plan type, signup date
- **sessions** — 50,000 app sessions with timestamps
- **transactions** — 28,000 payments (success, failed, refunded)
- **content_views** — 30,000 episode watches with completion percentage
The data covers October 2024 to January 2025.
I picked these 4 tables because they cover the questions a product team 
actually asks every day. If someone asks about DAU, that comes from sessions. 
Revenue questions come from transactions. Content performance comes from 
content_views. User segmentation comes from users. Every question maps cleanly 
to one or more of these tables.

---
## Project Structure

generate_data.ipynb → synthetic data generation
analytics_queries.ipynb → manual SQL validation
ai_agent.ipynb → core NL-to-SQL pipeline
demo.ipynb → walkthrough/demo notebook
pockettoons.db → SQLite database

---

---

## How the Agent Works

When you ask a question, 4 things happen in order:

**Step 1 — Is the question clear enough?**
First the agent checks if your question is too vague to answer. Something like 
"how are we doing?" doesn't tell us what metric you want, so the agent asks you 
to clarify instead of guessing. A wrong confident answer is worse than asking 
one follow up question.

**Step 2 — Write the SQL**
If the question is clear, the agent sends it to the AI model along with the 
full database structure. The model reads the question and writes a SQL query 
to answer it.

**Step 3 — Run the SQL**
That SQL query gets executed on the real database and returns actual numbers.

**Step 4 — Explain the answer**
The actual numbers get sent back to the AI model which then writes a plain 
English explanation of what the numbers mean.

---

## Why I Made Two Separate AI Calls

I could have done everything in one call but I split it into two on purpose.

The first call only has one job that is write correct SQL. That's it. No 
distractions.

The second call gets the actual numbers from the database. Because it can 
see the real data, it can't make up numbers — it just explains what's there.

If I combined them into one call, the model would be trying to write SQL 
AND explain results at the same time, which makes both worse. Keeping them 
separate also makes it much easier to figure out where something went wrong 
— is it the SQL that's wrong, or is it the explanation?

---

## How Long It Takes(Latency)

Each question takes about 4 to 5 seconds to answer. That's the two AI calls 
plus the database query. For a small team analytics tool this is completely 
fine. If this were a high traffic tool with thousands of questions per day, 
I would add caching so the same question asked twice returns the saved answer 
instantly instead of calling the AI again.

---

## Why I Chose These Tools

**Groq with Llama 3.3 70B**
Groq is completely free which was the main reason. I actually tested a smaller 
model first (Llama 3.1 8B) but it kept writing wrong SQL for complex queries 
especially the D7 retention calculation which needs a CTE and a JOIN. The 
larger model handled those correctly every time so I switched.

**SQLite**
No setup needed, just a file. The SQL it uses is almost identical to what 
BigQuery or Snowflake uses so swapping it out later would be straightforward — 
only the part that runs the query would need to change, everything else stays 
the same.

**No agent framework like LangChain**
I looked at using LangChain but for what I needed — take a question, write SQL, 
run it, explain it — adding a framework would have meant more code not less. 
The whole agent is about 80 lines of Python that I can read and debug easily. 
I'd rather have simple code I understand than a framework doing things behind 
the scenes.

---

## When SQL Goes Wrong

The AI doesn't always write perfect SQL. Here's what I did about it:

**Give it clear rules upfront**
I told the model things like — country values in our database are codes like 
IN and US, not full words like India or United States. Never filter by date 
range unless the user specifically asked for it. Always start the D7 retention 
window from the signup date itself. Most wrong SQL comes from the model not 
having enough context, so I put as much context as possible in the instructions.

**Catch errors cleanly**
If the SQL fails for any reason, the code catches it and shows a friendly 
message asking the user to rephrase. The app never crashes with a raw error.

**Ask before guessing**
The ambiguity check before generating SQL is actually the most important safety 
layer. If the question is vague, the agent asks what you meant. Better to ask 
than to answer the wrong question confidently.

**What I'd add with more time**
If the SQL fails, automatically send the error message back to the model and 
ask it to fix the query. Most SQL errors can be fixed in one retry. I handled 
it manually this time but this would be the first thing I'd add.

---

## How I Checked If It Was Actually Correct

I wrote the same questions as manual SQL queries in `Analytics_queries.ipynb` 
and compared the numbers the agent gave vs what the manual SQL gave. Those 
manual queries are the ground truth.

Here's what I checked:

| Question | Match? |
|---|---|
| MAU per month | Same numbers |
| DAU on a specific date |  Same numbers |
| Revenue by country |  Same numbers |
| Revenue from US users | Same numbers |
| Premium users in India |  Same after fixing country code |
| Sessions January vs December |  Same numbers |
| Genre views by month |  Same numbers |
| D7 retention by cohort |  Slightly different |

7 out of 8 matched exactly. The one that didn't was D7 retention — the agent 
was starting the 7 day window from the day after signup instead of the signup 
day itself. I caught it by comparing with the manual query and fixed it by 
adding a clearer rule in the instructions.

That's 87% accuracy. More importantly catching that one difference is exactly 
why I built the manual SQL file — so I had something to compare against.

If I were doing this properly for production I'd have a set of 50 or 100 
labeled questions and run them automatically every time I changed anything 
in the agent.

---

## What Would Happen at Real Scale

Right now this uses a small SQLite file with 4 tables. A real analytics 
warehouse might have 100 or more tables.

The main problem that creates is you can't paste 100 table descriptions into 
every prompt — it gets too long and expensive and the SQL quality actually 
gets worse with too much irrelevant information.

The way I'd solve it is to store all table descriptions separately and when 
a question comes in, first figure out which 4 or 5 tables are actually relevant 
to that question, then only put those into the prompt. That way the prompt 
stays focused no matter how many tables exist.

Other things that would matter at real scale:

- Save answers to repeat questions so the same question doesn't cost an API 
  call every time
- Add timeouts so a slow query doesn't hang forever
- Make sure users can only see data they're allowed to see
- Keep logs of every question, what SQL was generated, how long it took, 
  whether it succeeded — so you can see if something is breaking
- Connect to BigQuery or Snowflake instead of SQLite — only the part that 
  runs the query changes, nothing else

**Rough cost at 10,000 questions per month:**
Groq is free so right now the cost is basically zero. If this moved to a paid 
model it would be around $90 per month based on typical token usage. With 
caching for repeat questions that drops to around $60.

---

## What I Didn't Finish

I want to be upfront about what I ran out of time for:

**Auto-fixing wrong SQL** — right now if SQL fails the user sees "try 
rephrasing." Better behavior would be to automatically try to fix it once 
before giving up. I know how to build this, just didn't get to it.

**Caching repeat questions** — if the same question gets asked twice it 
runs the whole pipeline again. Easy to fix with a simple dictionary that 
saves previous answers.

**Bigger eval set** — I manually checked 8 questions. A proper eval would 
have 50+ questions and run automatically. This matters a lot in production 
because every time you change the prompt you need to know you didn't break 
something else.

**Automatic sanity checking** — if the agent says DAU is higher than the 
total number of users, something is clearly wrong. Adding a check like that 
before showing the answer would catch a whole class of bugs automatically.

I made the call to get the core pipeline working well rather than having 
more features that are half built.

---

## What's Actually Broken or Imperfect

- The agent sometimes uses full country names like India instead of the 
  country code IN. I added rules to prevent this and it mostly works but 
  edge cases still slip through occasionally.

- D7 retention numbers differ slightly from manual SQL depending on how 
  the model interprets the start of the 7 day window. Fixed in the prompt 
  but worth knowing about.

- Every time you open a new Colab session you have to re-upload the database 
  file. This is just how Colab works — in a real setup all the notebooks 
  would connect to the same database server automatically.

---

## Tools Used

Python, SQLite, Groq API, Llama 3.3 70B, Pandas, Matplotlib, Google Colab

---














