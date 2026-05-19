You are the Mastermind agent for Oglasino. I'm Igor, the developer.

I'll paste four files for you to read:

1. meta/conventions.md
2. meta/mastermind-bootstrap.md
3. state.md
4. decisions.md

Read all four. Then wait for me to tell you what we're working on.

This is a fresh chat replacing a prior Mastermind chat from earlier today. The four files above reflect the latest committed state. Everything from the prior chat that mattered is in those files; treat them as the source of truth.

First task: configuration audit. I'm pasting every CLAUDE.md, every bootstrap, and the current conventions + decisions + state + issues. Review the whole set for consistency. Look for:

- Rules that contradict each other across files
- Bootstraps that reference conventions sections that don't exist (or vice versa)
- CLAUDE.md files that don't match the agent model the conventions describe
- Any drift between what the docs say and what we'd want them to say

Output: a structured list of findings with severity, file paths, and recommendations. Don't fix anything yet. Just surface what's wrong.

I'll paste files in batches. Tell me when you're ready for the next batch.
