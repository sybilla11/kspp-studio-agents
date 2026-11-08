---
name: analyze-story
description: Analyze a user story or ticket before implementation. Produces gaps and open questions, Given/When/Then acceptance criteria, a codebase impact map, and an ordered TDD test plan, using the story-analyst agent.
argument-hint: "<story text | path/to/story.md> [--save]"
---

Analyze this user story with the `story-analyst` agent from the kspp-analysis plugin.

Input: $ARGUMENTS

Steps:

1. **Get the story.**
   - If the input is empty, ask the user to paste the story or give a path to a file containing it, then stop.
   - If the input (ignoring a trailing `--save`) is a path to an existing file, read that file and use its contents as the story.
   - Otherwise, the input itself is the story text.
2. **Run the analysis.** Delegate to the `story-analyst` agent, passing the full story text verbatim, prefixed with: "Analyze this user story against the codebase in the current working directory." Don't summarize or rewrite the story before passing it on.
3. **Show the full report.** The report is the deliverable: the developer works from every acceptance criterion, every row of the impact map, and every step of the test plan, so a summary throws away the part they need. Output the agent's report in full, exactly as returned, with all seven sections. Don't summarize, shorten, or add commentary before it; anything you want to add goes after it.
4. **Save it if asked.** If the input ended with `--save`, write the report to `docs/stories/<slug>.md`, where `<slug>` is a short kebab-case name derived from the story (create the folder if needed), and tell the user the path. Otherwise, after the report, offer once to save it there.
