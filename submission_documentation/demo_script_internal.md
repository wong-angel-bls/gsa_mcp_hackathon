# Demo Script: BLS Public Data MCP Server
## Internal BLS Audience — Capability Showcase (~6–7 minutes)

---

## PRE-RECORDING SETUP CHECKLIST

Before hitting record, have these tabs/windows open and ready:
- [ ] **Tab 1:** watsonx Orchestrate — chat with **`gsa_ds_bls_baseline_agent`** (no tools) loaded
- [ ] **Tab 2:** watsonx Orchestrate — chat with **`gsa_ds_bls_agent`** (MCP-augmented) loaded
- [ ] **Tab 3 (optional):** BLS data homepage — https://data.bls.gov/timeseries/ — for reference
- [ ] Clear any prior conversation history from both agent chats before recording

---

## SECTION 1 — INTRO (0:00–0:45)

**[On screen: Title slide or repo README]**

> "Hi everyone — I'm Vicky, and I'm here with my colleagues Angel and Ayme. We're all Data Scientists with the Office of Prices and Living Conditions here at BLS.
>
> Today we want to walk you through something we built as part of the government-wide MCP Server Hackathon — a **BLS Public Data MCP Server**. The idea is straightforward: AI agents are increasingly being used across government to answer questions, but they can't natively reach our data. Their training knowledge is frozen at a point in time, and they don't have a reliable way to query the BLS Public Data API on their own.
>
> BLS data has a learning curve. The structure isn't always intuitive — especially for people who are new to our data or new to a role here. We organize data into dozens of surveys, each with its own series ID format made up of multiple coded segments. Knowing which series to use for a given question isn't obvious, and that's a real barrier for anyone getting up to speed.
>
> So, as part of the GSA MCP Server Hackathon — an eight-week challenge to build MCP servers that connect AI agents to real government data, with less than six weeks of active coding time — we built a server that sits between an AI agent and the BLS API, handling series discovery, ID validation, ambiguity detection, and live data retrieval. We built it using Bob — IBM's AI coding assistant — which helped us move quickly within that tight timeframe. The result is an agent that can answer plain-language questions about BLS data with accuracy and full provenance. Let's show you what that looks like."

---

## SECTION 2 — THE PROBLEM: BASELINE AGENT (0:45–1:15)

**[Navigate to: watsonx Orchestrate → chat with `gsa_ds_bls_baseline_agent`]**

> "I'm in watsonx Orchestrate — IBM's AI agent platform that lets you build and chat with agents that can call external tools and APIs. Before I show you what our server does, let me show you the problem it solves. Here's a baseline BLS assistant with no tools attached — it answers entirely from training data."

**[TYPE IN CHAT:]**
```
What was the U.S. national unemployment rate for each month of 2024?
```

> "And here's what it returns — something like: *'I'm sorry, but I don't have the monthly unemployment rate figures for 2024. My training data only goes through early 2024, and I don't have real-time access to retrieve newer numbers.'* The model is being honest — it knows what it doesn't know. But that data is publicly available right now on the BLS website. The gap isn't in the data — it's in the agent's ability to reach it."

---

## SECTION 3 — THE SOLUTION: MCP-AUGMENTED AGENT (1:15–5:15)

**[Navigate to: watsonx Orchestrate → chat with `gsa_ds_bls_agent`]**

> "Now let's switch to our MCP-augmented agent — same model, but connected to our BLS Public Data MCP Server and its twelve tools. I'll ask it the same question first, then show you the broader range of what it can do."

---

### Question 1 — Live data retrieval (1:15–2:00)

**[TYPE IN CHAT:]**
```
What was the U.S. national unemployment rate for each month of 2024?
```

**[While waiting for the response:]**
> "The agent is running our `answer_bls_question` tool — going straight to the BLS API, not generating from memory."

*If it's taking longer:* > "What's happening under the hood is the server is parsing the question, matching it to the right series in our catalog, validating the series ID locally, and then making a live call to the BLS Public Data API. All of that is happening in real time."

**[Pause — point out the result]**

> "We get the actual monthly unemployment rate numbers for all of 2024, sourced directly from BLS, with the series cited. This is data that a standard AI model couldn't give you — its training doesn't cover it. Here, it's live."

---

### Question 2 — Ambiguity detection (2:00–3:00)

**[TYPE IN CHAT:]**
```
What is the current annual inflation rate based on the Consumer Price Index?
```

> "This one is worth pausing on. 'Inflation rate' is genuinely ambiguous — it could mean the raw CPI index level, the month-over-month change, or the 12-month percent change. Rather than silently picking one, the server surfaces a clarification question. That's intentional design — we built in an ambiguity detection step because getting the wrong measure and not knowing it is worse than being asked to clarify."

**[TYPE IN CHAT — select the 12-month percent change option]**
```
3
```

**[While waiting:]**
> "It's now fetching the live data."

*If it's taking longer:* > "It's making a live call to the BLS API — same data you'd get if you queried the API directly, just surfaced through natural language."

**[Pause — point out the result]**

> "And we get the year-over-year CPI change — the measure most commonly reported as the annual inflation rate. Worth noting: the agent returns December 2024 as the most recent figure, which reflects a known LLM behavior around date reasoning. The data itself is accurate — it came straight from the BLS API — but how the model interprets 'current' is something we're continuing to work on."

---

### Question 3 — Series discovery (3:00–3:45)

**[TYPE IN CHAT:]**
```
What BLS data is available about job openings and hiring trends?
```

> "Now something a little different — an exploratory question. This is the kind of thing someone might ask if they're new to BLS data or trying to figure out which survey covers a topic they care about. The agent calls our `suggest_series` tool, which searches a curated flagship series catalog locally — no API call needed. It returns ranked suggestions with the series name, the survey it belongs to, and a plain-language rationale for why it matches."

*If it's taking longer:* > "This one actually runs entirely locally — no network call. It's searching our embedded series catalog. If it's taking a moment, it's the LLM reasoning about the response, not a data fetch."

**[Pause — point out the ranked suggestions]**

> "So instead of a user having to navigate bls.gov and understand the difference between JOLTS and CES to find job openings data, the agent surfaces that for them directly."

---

### Question 4 — Series ID decoding (3:45–4:30)

Now I'm going to ask the agent a question that highlights our `decode_series_id` tool

**[TYPE IN CHAT:]**
```
Decode this BLS series ID for me: CES0000000001
```

> "BLS series IDs encode a lot of information in a compact format — survey, geography, industry, data type, seasonal adjustment — but reading them isn't intuitive. Our `decode_series_id` tool breaks down every segment into plain language using our embedded BLS catalog. No API call, all local."

*If it's taking longer:* > "Same as before — this is all local. The catalog is embedded in the server, so there's no API call. Just the model formatting the response."

**[Pause — point out the decoded output]**

> "So `CES0000000001` decodes as: Current Employment Statistics, national, all industries, all employees, seasonally adjusted — the nonfarm payrolls headline series. That's information that used to require knowing the format spec by heart or digging through BLS documentation."

---

### Question 5 — Metro area employment (4:30–5:15)

**[TYPE IN CHAT:]**
```
How many people were employed in the New York City metro area in 2024?
```

> "A geographic question — this exercises our area code lookup and the State and Metro Area Employment survey. And again, notice the clarification step — the agent asks which employment measure we want: nonfarm payrolls, unemployment rate, or job openings. Same intentional design as before — it won't assume."

**[TYPE IN CHAT — select nonfarm payrolls]**
```
a
```

**[While waiting:]**
> "It's now resolving 'New York City metro area' to the correct BLS area code and fetching the data."

*If it's taking longer:* > "The server is resolving the metro area to the correct BLS area code, then fetching employment data from the SM survey. Geographic lookups add a small step to the pipeline."

**[Pause — point out the result]**

> "And we get nonfarm payroll employment for the New York metro, sourced from the SM survey. The server resolved 'New York City metro area' to the correct BLS area code automatically — no need for the user to know that code exists."

---

## SECTION 4 — LIMITATIONS & CLOSING (5:15–6:15)

**[Optional: return to slides or stay in the agent]**

> "Before we close, we want to be clear about what this is — it's an MVP, a prototype built to demonstrate that this capability is possible.
>
> With that framing in mind, here's what we know the prototype doesn't yet cover: the server includes twelve BLS surveys and a curated set of flagship series, but BLS publishes far more data than what's in the current catalog. And as you saw with the inflation and New York questions, the agent will sometimes ask a clarifying question before fetching data — that's intentional, but it's something that would need refinement at scale.
>
>
> And probably the most important gap: **we built this without subject matter experts in the loop.** The series descriptions, the flagship catalog, the ambiguity detection rules — all of that was developed by our team based on our own knowledge of BLS data. The domain expertise to make something like this truly authoritative needs to come from the economists, program area experts, and survey teams who own this data day to day.
>
> What we hope you take away is that the capability is real. A prototype that lets someone ask a plain-language question and get back a sourced, accurate BLS data answer — built in under six weeks — shows this is achievable. Whether it makes sense to take it further is a conversation we'd want to have with you. Thank you."

---

## TIMING GUIDE

| Section | Target Time | Cumulative |
|---|---|---|
| Intro | 0:45 | 0:45 |
| Baseline agent (problem) | 0:30 | 1:15 |
| Q1 — Live data retrieval | 0:45 | 2:00 |
| Q2 — Ambiguity detection (inflation) | 1:00 | 3:00 |
| Q3 — Series discovery (job openings) | 0:45 | 3:45 |
| Q4 — Series ID decode | 0:45 | 4:30 |
| Q5 — Metro area employment | 0:45 | 5:15 |
| Limitations & what's next | 1:00 | 6:15 |

**Total target: ~6–6.5 minutes**

---

## TIPS

- This audience knows BLS data — lean into the technical details (survey names, series ID structure) rather than explaining what BLS is
- The baseline vs. MCP contrast is the clearest way to show value — keep both tabs visible side by side if screen space allows
- The key narrative contrast is always: **baseline = frozen training data → MCP agent = live verified data**
- The limitations section is a strength, not a weakness — being candid about what's missing, especially the subject matter expert gap, builds credibility
- If a tool call takes time, refer to what's happening under the hood ("it's querying the BLS API directly")
