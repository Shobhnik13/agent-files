---
name: docsummarize
description: Summarize a docs site, tech book, codebase, paper or tool into one newbie-friendly, self-contained local HTML deep-dive (sidebar nav, TL;DR, mental-model diagrams, ELI5 boxes, caveats, glossary, sources). Use when the user invokes /docsummarize or asks to summarize/explain docs, a book, or a codebase in this style.
argument-hint: <url | file | book | codebase dir> [focus]
---

# docsummarize

Turn a large technical source into one local HTML page that a newbie can read once and understand, while staying technically accurate for an engineer.

Reference shell: `~/.claude/skills/docsummarize/template.html`. Read it before writing. Copy its `<style>` block verbatim and reuse its components. Do not invent new styling.

## Steps

1. **Read the source fully.** Docs site: fetch the main pages, the README and the architecture pages. Book: read the chapters given. Codebase: trace the real flow end to end (entry point → core modules → storage/output) before you write. Do not summarize from file names alone.
2. **Find 1–3 fundamental ideas.** Everything else in the subject exists to support these ideas. They become the TL;DR cards.
3. **Write the page** with the fixed section order below.
4. **Save** it as `<topic-slug>-docs-dive.html` in the current working directory, unless the user gives a different path. Print the path. Do not publish it anywhere.

## Fixed section order

| # | Section | Content |
|---|---------|---------|
| — | Title + lede | `<h1>` "X, explained from <starting point> up". The lede is one sentence about what the reader will understand by the end. Below it, put the tag legend. |
| 0 | TL;DR | One `.callout` with a `.big` 2–3 sentence summary. Then a `.grid2` of cards, "The fundamental idea #N — name". Then an `.eli5` box. |
| 1 | The problem it solves | Explain why the subject exists. Use a table: failure / why it happens mechanically / what this gives you. Then add an `.eli5` box. |
| 2 | Mental model | An inline SVG `.fig` with a caption that separates the layers or parts that readers often mix up. Then an `.eli5` box. Then a `.callout.alt` that says why the separation matters. |
| 3 | Deep dive | Numbered subsections (3.1, 3.2 …), each listed as a `nav a.sub`. Use the loop shown below. |
| 4 | Comparisons *(optional)* | Include only if the subject has real alternatives. Give one subsection for each alternative, then a capability matrix with `td.y/.p/.n`. |
| 5 | Worked example | One realistic scenario. Show "old/naive way" and "this way" side by side in `.grid2`. |
| 6 | When to use what *(optional)* | Include only if comparisons exist. Use a table: situation → pick. |
| 7 | Honest caveats | A `.callout.warn` numbered list: limits, trade-offs, maturity, licence, and benchmark or marketing caveats. Then a `.callout.good` one-line summary. |
| 8 | Glossary | One `<details><summary>` for each jargon term used on the page. |
| — | Sources | A `.src` list with links, each with a note on what it covered. Then a `.kbd` line: "Compiled <date>. Claims marked; verify before relying on them." |

Renumber the sections and the nav if you omit an optional section.

## Deep-dive loop (each subsection)

1. A trust tag and the technical explanation. Explain the **mechanism** (how and why it works), not only what it does.
2. A visual artifact when it helps: an SVG `.fig`, a `pre` with code or notation, a `.tbl`, or a `.grid2`/`.grid3` of cards.
3. An `.eli5` box with an everyday analogy.
4. **The trade-off** or **why it matters**, in one short paragraph.

## Voice rules

- Write for a newbie and stay accurate for an engineer. Never dumb down by removing the mechanism.
- Every non-trivial concept gets an `.eli5` analogy. Take analogies from everyday life: shop, library, phone contacts, WhatsApp, newspaper office, restaurant, notebook. Use simple names (Rahul, Priya) and familiar places. Each analogy must map onto the real mechanism and must also show the catch.
- Use trust tags inline: `tag v` **verified** (stated in the source), `tag m` **claim** (author/vendor says so, unverified), `tag o` **analysis** (your interpretation).
- Do not invent facts, numbers, versions, names or benchmarks. If something is uncertain, say so and tag it.
- Be skeptical of marketing. Name it as marketing.
- Use short sentences and put **bold** keywords at the start of bullets. Use no filler and no praise.

## Visual rules

- Use only the CSS variables and classes from the template. SVG boxes use `box`, `boxA` (accent), `boxB` (alt), `boxG` (good/storage/output) and `boxW` (warn/in-progress). Lines use `ln`/`lnA` with an arrow `marker`, and each SVG gets its own unique marker id.
- Give each SVG a `viewBox` of about 900 wide. Keep text at 11–15px. Use `class="mut"` for sub-labels.
- The page must stay one self-contained file with no external scripts, fonts or images.

## Source-type notes

- **Docs site / tool:** follow the template as written.
- **Book:** "The problem" covers what the book argues against or fixes. The deep dive gives one subsection for each key idea, not for each chapter. Skip comparisons unless the book makes them itself.
- **Codebase:** the mental model shows the actual module or service layout. The deep dive follows the request or data flow, and cites `path/file.ext:line` in `code`. The worked example traces one real request through the code.
