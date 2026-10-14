# New Grad Interview Landscape (Oct 2025 to Oct 2026)

Research done on 2026-10-13 to decide the scope and depth of this repo. It covers new grad (NG) software engineer loops over the past 12 months, with a focus on the last 6 months.

## Evidence quality

Recent, dated, firsthand NG reports are scarce. LeetCode Discuss and 1point3acres could not be read (blocked or login required), and no relevant Reddit threads surfaced. Much of the "2026" material comes from interview prep vendors that recycle each other's content. Ratings used below:

- **Strong**: official company source, or press coverage confirmed by the company
- **Medium**: dated firsthand candidate report
- **Weak**: vendor guide, anonymous or undated post, or a single unsourced claim

## 1. AI-assisted coding rounds

This is the clearest change of the past year.

| Company | What is known | Evidence |
| --- | --- | --- |
| Meta | Candidates in select roles are expected to use the built-in CoderPad AI assistant. Started around Oct 2025. Multi-file codebase, about 60 minutes, staged tasks (fix a failing test, add a feature, optimize). Whether E3 (NG) loops include it is not verified. | Strong (official page), NG level unverified |
| Google | Pilot of a Gemini "code comprehension" round for junior and mid-level roles on select US teams, announced May 2026, rolling out in H2 2026. Read, debug, and optimize existing code. Classic no-AI round remains. | Strong (press, confirmed by Google) |
| Canva | AI-assisted coding replaced the CS Fundamentals round. Candidates use their own tools on open-ended product problems. | Strong (official blog, Jun 2025) |
| Shopify | Execs say AI is allowed and encouraged in pair programming interviews. | Medium (public exec statements) |
| Rippling, Stripe, Amazon, Coinbase, LinkedIn, Datadog | Reported AI-assisted rounds of various formats. | Weak |
| Anthropic, Cursor | Reportedly do not allow AI in interviews. | Medium (press) |

Common patterns across companies:
- Working in existing code (read, debug, extend, optimize) more than writing from scratch
- Verifying AI output is graded more than the output itself. Over-reliance is penalized, and under-use can be too
- Breaking vague tasks into sub-tasks, writing tests, and narrating reasoning
- Classic DSA rounds still exist alongside at Meta and Google

AI-assisted coding prep is out of scope for this repo for now.

## 2. Design rounds in NG loops

A full distributed system design round for NG has not clearly become standard. Design does show up, mostly in lighter forms. Most common first:

1. **OOD / low-level design**: parking lot, vending machine, LRU cache, logging framework
2. **Component / API design**: rate limiter, schema or API design, Stripe-style integration tasks
3. **Small-scale system design**: URL shortener, notification system, room reservation
4. **Project deep dive with "how would you scale it"**: becoming formal (Google pilot adds a technical design discussion of a past project; Bloomberg EM rounds drill into project internals)
5. **Domain-flavored design** at a few firms: order book or matching engine (Citadel), storage and concurrency (Databricks)

| Company | NG design round | Evidence |
| --- | --- | --- |
| Google L3 | No standalone system design. Pilot adds a design discussion of a past project. | Strong (press) |
| Meta E3 | No design round (two coding plus behavioral). | Weak (vendor, consistent across sources) |
| Amazon SDE I | Disputed: OOD or light system design, sometimes as follow-ups in coding rounds. | Weak |
| Microsoft | Mostly OOD or low-level design. | Weak |
| Bloomberg | Historically design only in the EM round (OOD, API, project deep dive). A reported 2027 NG format (two AI-assisted coding rounds plus one system design round) could only be traced to one unsourced GitHub repo. | Medium (older firsthand), new format unverified |
| Databricks | One architecture / system design round in the onsite, focused on concurrency and scalability. | Medium |
| Uber | Mostly coding plus behavioral; some teams include light system design. | Medium |
| Stripe | Integration and bug-fix rounds instead of system design. | Weak |
| Citadel | Vendors list one trading-flavored design round. | Weak |
| OpenAI, Anthropic | Full system design for experienced hires; NG format unclear. | Weak |

## 3. What depth NG candidates are evaluated on

- Clarify requirements, lay out a coherent API and data model, and name at least one explicit trade-off
- Back-of-the-envelope estimation is rarely mentioned for NG
- The deepest questioning tends to come in project deep dives, where interviewers keep asking until the candidate runs out of answers

Level-by-level reference ([AlgoMaster](https://algomaster.io/learn/system-design-interviews/expectations-by-level)):

| | Junior | Mid-level | Senior |
| --- | --- | --- | --- |
| Depth | Explains concepts | One important component | Two or three critical areas |
| Trade-offs | Names options | Justifies main choices | Compares alternatives and recommends one |
| Scope | Happy path | Happy path plus basic failures | Scale, failures, bottlenecks |

## Decisions for this repo

- **Target the mid-level bar** as a safety margin above NG expectations. Keep estimation quick and rough.
- **Add component / API design problems**, since they are a common NG design format.
- **Keep Advanced problems as stretch goals.**
- **Out of scope here**: OOD (studied in a separate project that runs code), project deep dives, and AI-assisted coding prep.

## Sources

AI-assisted coding:
- Meta Careers, hiring process (official, undated): https://www.metacareers.com/hiring-process/
- Canva Engineering, "Yes, you can use AI in our interviews" (official, 2025-06-11): https://canva.dev/blog/engineering/yes-you-can-use-ai-in-our-interviews
- Google pilot guide, Northeastern Career Design (2026-05-13): https://careers.northeastern.edu/blog/2026/05/13/googles-ai-assisted-coding-interview-2026-guide/
- Google pilot coverage (2026-05): https://finance.yahoo.com/sectors/technology/articles/asking-kid-math-test-without-151500986.html
- Simplify, Meta new grad guide (2026-06, updated 2026-07): https://simplify.jobs/blog/meta-new-grad-swe-interview
- Blind, first reviews of Meta AI-assisted coding (2025-10-11): https://www.teamblind.com/post/first-reviews-of-meta-ai-assisted-code-interviews-are-in-12ouo1yt
- Pragmatic Engineer, Shopify on AI: https://newsletter.pragmaticengineer.com/p/how-ai-is-changing-software-engineering
- Blind, LinkedIn AI-assisted round (2025-10-28): https://www.teamblind.com/post/linkedin-ai-assisted-coding-round-qz4kb1gx
- dev.to, eight companies' AI coding interviews (vendor, 2026-10-12): https://dev.to/max_nyriev/eight-companies-eight-different-ai-coding-interviews-k60
- Karat, 2026 engineering interview trends: https://karat.com/engineering-interview-trends-2026

Design rounds:
- AlgoMaster, expectations by level: https://algomaster.io/learn/system-design-interviews/expectations-by-level
- Hello Interview, what is expected at each level: https://hellointerview.com/blog/the-system-design-interview-what-is-expected-at-each-level
- Roundz, Bloomberg NG interview experience (2025): https://roundz.substack.com/p/interview-experience-bloomberg-software-engineer-new-graduate
- atomeocean, Bloomberg 2026-cycle firsthand report (about 2025-10): https://atomeocean.com/interview-experience/bloomberg/bloomberg-sde-intern.html
- GitHub Jouujo/AI-Coding-Prep (unsourced claim of Bloomberg 2027 format): https://github.com/Jouujo/AI-Coding-Prep
- Blind, Bloomberg NG system design round (2021-08): https://www.teamblind.com/post/Bloomberg-new-grad-system-design-round-poeBFRTE
- PracHub, Bloomberg system design questions: https://prachub.com/companies/bloomberg/categories/system-design
- Hello Interview, Meta E3 guide: https://www.hellointerview.com/guides/meta/e3
- Databricks NG interview (LeetCode): https://leetcode.com/discuss/interview-question/5669336/Databricks-New-grad-Interview-Experience/
- Uber NG interviews (Glassdoor): https://www.glassdoor.com/Interview/Uber-Software-Engineer-New-Grad-Interview-Questions-EI_IE575263.0,4_KO5,31.htm
- Citadel technical interview (vendor): https://techscreen.app/articles/citadel-technical-interview-process-2026
- Simplify, SWE recruitment roadmap 2027: https://simplify.jobs/blog/swe-interview-prep-roadmap-2027
