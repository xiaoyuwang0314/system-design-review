# System Design Review

These are my personal notes for system design interview preparation. I use this repo to review core concepts, work through classic design problems, and record what I learn from mock interviews.

This is a study log, not an authoritative reference. Notes may be incomplete or contain mistakes, and they reflect my own understanding at the time of writing.

## Start here

- [`ROADMAP.md`](ROADMAP.md): phases, exit criteria, and what to study next
- [`DEPTH-GUIDE.md`](DEPTH-GUIDE.md): how deep to go on each component, and when to stop
- [`research/ng-interview-landscape-2026.md`](research/ng-interview-landscape-2026.md): what new grad interview loops looked like from Oct 2025 to Oct 2026, and why this repo has its current scope

## Scope

This repo covers distributed system design and component / API design, targeting the mid-level bar as preparation for new grad interviews. Object-oriented design is studied in a separate project because it involves running code. AI-assisted coding and project deep dives are out of scope for now.

## Structure

| Folder | What goes there |
| --- | --- |
| [`fundamentals/`](fundamentals/) | Core building blocks: scalability, caching, databases, replication, sharding, consistency, messaging, and so on |
| [`problems/`](problems/) | Worked solutions to classic design questions and component / API design problems, one file per problem |
| [`research/`](research/) | Research on interview formats that informs the scope and depth of these notes |
| [`mock-interviews/`](mock-interviews/) | Logs of practice sessions: the prompt, what I did, feedback, and what to fix next time |
| [`templates/`](templates/) | Reusable templates for writing up problems and mock sessions |

## How I use it

1. Follow the current phase in the roadmap. Study a fundamentals topic and write it up in my own words.
2. Pick a problem, solve it on paper under a time limit, then write the clean version using [`templates/problem-template.md`](templates/problem-template.md).
3. Run a mock interview and log it using [`templates/mock-interview-template.md`](templates/mock-interview-template.md).
4. Revisit weak spots that show up in mock feedback.

## Built with AI

I write and organize these notes together with [Claude Code](https://claude.com/claude-code), Anthropic's agentic coding tool, running **Claude Opus 5.5** (`claude-opus-5-5`). Claude helps me set up the repo, explain concepts, draft and review write-ups, and act as the interviewer in mock sessions. The study, the practice, and the final judgment on what goes into these notes are mine.

## License

[MIT](LICENSE). Feel free to use, copy, and adapt anything here.
