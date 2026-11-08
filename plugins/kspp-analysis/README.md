# KSPP Studio Analysis Agent (`kspp-analysis`)

Pre-implementation analysis for user stories. Turns a story or ticket into gaps and open questions, Given/When/Then acceptance criteria, a codebase impact map with file and line evidence, and an ordered TDD test plan.

| Component | Invoke as | What it is |
|---|---|---|
| [`story-analyst`](agents/story-analyst.md) | Picked automatically when you ask Claude to analyze a story, or `kspp-analysis:story-analyst` | Read-only subagent (`Read`, `Glob`, `Grep`) that does the analysis |
| [`analyze-story`](skills/analyze-story/SKILL.md) | `/kspp-analysis:analyze-story <story or path> [--save]` | Command that hands the story to the agent and shows the full report |

Install instructions, the report format, and a real example are in the [repository README](../../README.md).
