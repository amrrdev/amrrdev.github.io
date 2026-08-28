You are rewriting a series of technical blog posts about systems engineering into longer, textbook-style chapters. The posts live in `_posts/*.md` (Jekyll-style: YAML frontmatter between `---` delimiters at the top, then markdown body). A `ROADMAP.md` defines the series order, stages, and what each article should cover.

For EACH post, do the following. Process them one at a time, reading the post and `ROADMAP.md` (and sibling posts when needed for context) before rewriting.

## 0. Series context (do this first for every post)
Read `ROADMAP.md`. Find the article's stage, subject area, and series_order. Read the 1-2 posts immediately before it in `series_order` so you understand what the reader already knows. Use that to write a grounding opening if the post starts abruptly (see rule 1).

## 1. Remove entirely
- Any "Where this article fits" section, or any section that only narrates what came before and what comes next. Delete it completely; do not summarize it elsewhere.
- Any "If you want to build this later" section, or any equivalent closing "build a project" prompt. Delete it completely.
- Any dashed horizontal rule lines (`---`) used as visual separators INSIDE the body. Do NOT remove the `---` delimiters of the YAML frontmatter block. (Note: many posts have no such body rules; only remove them if present.)

## 2. Reword every section heading
- No short, formulaic, listicle-style titles (e.g. "Why X Matters", "What Y Means", "X and Y").
- Rewrite each heading as a fuller, natural phrase a human author would use in a book: descriptive, sometimes conversational, varied in structure and length across the document. Do NOT repeat the same grammatical pattern for every heading.
- Keep the heading's underlying topic identical. Change phrasing only, not what the section is about or its position.
- Good examples of the target style (calm, descriptive, book-like, varied): "What we mean by a resource", "The life cycle of a resource", "Cleanup failures are correctness bugs", "Connection pools in practice", "When local limits conflict", "A leak that became an outage". Avoid punchy one-liners, aphorisms, or "X: the Y that Z" patterns. Avoid em-dashes in headings.
- Keep all headings at their current level (`##` for sections, `###` for subsections) unless the post already uses a different consistent scheme.

## 3. Expand, never compress
- Keep every existing idea, example, diagram, and code block. Do not cut anything (aside from the removed sections in rule 1).
- For each section, add depth: more worked examples, more "why this matters in practice," more connective reasoning, more concrete numbers or scenarios. Where the original states a concept, follow it with at least one elaboration: a consequence, an example of it going wrong, or a nuance from experience.
- Do not introduce new top-level sections or restructure order unless something is factually missing that the roadmap topic requires.

## 4. Preserve exactly
- The YAML frontmatter block at the top: unchanged.
- Tone: plain, direct, second-person-adjacent, explanatory. No marketing language, no hype words, NO em-dashes anywhere in the body.
- All existing Mermaid diagrams and code blocks: unmodified, unless expanding the surrounding explanation requires a new one (new ones are fine; existing ones stay verbatim).
- The "Interview definition," "Common misconceptions," and "Summary" sections' PURPOSE is preserved, though their headings are still reworded per rule 2. Definitions may be rephrased into the same tone while keeping their meaning.

## 5. Tone target (apply to the ENTIRE body, not just the intro)
Match the conversational, direct voice used in the intro of the "Resource Ownership and Limits" post. Reference style:
- Direct address with "you": "Once you accept that a resource is finite and shared, two questions stop being optional."
- Plain, concrete explanations with real scenarios and numbers: "A typical Linux process defaults to a limit of 1024 open file descriptors."
- Talks the reader through failure modes naturally: "A double-free is not a classroom problem."
- No hype, no em-dashes, no abstract throat-clearing. Explain terms before using them; ground abstract topics in what the reader already knows from earlier chapters.
If a post currently reads abstractly or jumps straight into jargon without defining it, ADD a short grounding paragraph at the top (after any removed metadata) that defines the key terms and says why the topic matters in the systems-programming context.

## 6. Output
Return the complete rewritten article as a single markdown file, ready to publish. Do not include commentary about what you changed. Overwrite the original post file with the rewritten content.

## 7. Heading approval note
If the user has already approved specific heading wording for a post in a prior pass, keep those approved headings and only re-tone the body. Otherwise apply rule 2 fresh
