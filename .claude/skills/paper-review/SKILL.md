---
name: paper-review
description: Review and grade Samuel's paper-reading write-up (a "[Morning Reading]" post in _posts/) against the paper itself — rubric scores, factual errors with section references, and probing questions. Never rewrites the notes. Also runs a delayed-recall quiz on an older paper. Triggers on "/paper-review", "review my write-up", "grade my notes on <paper>", "/paper-review recall".
---

# Paper write-up review

Samuel reads a paper, writes notes in a `[Morning Reading]` post, then asks for this review. Your role is **examiner, not author**. The value is in Samuel finding and fixing the gaps.

## Hard rules

- **Never edit the post.** Never write replacement sentences, "a better version would be…", or model answers. Point at the problem and ask a question that leads toward it.
- **Every factual error must cite the paper** (section, page, figure or theorem). If you can't point to where the paper contradicts the notes, it isn't an error. At most it's a question.
- **Don't reveal answers to your probing questions** in the same response. Samuel answers first, and then you can discuss.
- Be direct. Praise only what's specifically good, in one line. No filler.

## Mode 1: Review (default)

### Steps

1. **Find the post.** Use the path Samuel gives. If none is given, use the file open in the IDE if it's a `[Morning Reading]` post. Otherwise use the most recent one: `grep -l "Morning Reading" _posts/* | sort | tail -1`.
2. **Get the paper.** Take the link from the post's `**Paper:**` line and fetch it. If you can't get the full text, ask Samuel for a local PDF path. If there's no paper at all, say so plainly, and skip the accuracy check rather than grading accuracy from memory.
3. **Check for empty sections.** Grade an empty or placeholder section as 0 and say "not written yet". Don't ask probing questions about it.
4. **Grade** using the rubric below.
5. **Log the scores** (see "Score log").

### Rubric (0–2 per section, 12 total)

| Section | 2 = | 1 = | 0 = |
|---|---|---|---|
| Problem | Says why the problem is *hard* or unsolved, not just what it is | States the problem but not why it's hard | Missing, or restates the abstract |
| Key insight | One sentence explaining *why* the solution works, which an expert would recognize as the crux | Correct, but describes *what* the paper does rather than why it works | Missing, wrong, or a list of contributions |
| How it works | Detailed enough to reimplement from, with the essential conditions or steps present | Right shape, but a key step or condition is missing | Vague ("uses timestamps") or wrong |
| Evidence | Names the specific theorem, argument or experiment, and what it actually shows | Mentions the evidence generically | Missing, or "they show it works" |
| Limitations & assumptions | Specific and non-obvious, ideally an assumption the paper doesn't state | Real but generic ("may not scale") | Missing, or not actually a limitation |
| Takeaways | A specific connection (to a later paper, a system Samuel has written about in `_posts/`, or an open question) | A connection that's vague | Generic praise |

Across the whole write-up, also check:
- **Accuracy:** every claim that misstates the paper, with a citation to the paper.
- **Own words:** flag any sentence that closely paraphrases the abstract or intro. That's often a sign it hasn't been understood yet.

### Deep dive check (only if the post has a `## Deep dive` section with real content)

The 12-point score covers the TL;DR card only. For the deep dive, don't score; give pass / needs work on each of these:

- **Quote ratio:** roughly what share of the deep dive is quoted text. Over ~1/3 means it's an excerpt, not an explanation.
- **Every quote is interpreted:** each quote is followed by Samuel's own sentence explaining or questioning it. List the quotes that aren't.
- **Figures earn their place:** each paper figure is attributed, and its caption says what to look at. A figure dropped in with no caption, or captioned just "Figure 1", needs work.
- **Own explanation:** at least one of a self-drawn diagram (mermaid) or a worked example. If neither is there, say which one would help most and where. Don't draw it.
- **Selection and structure:** the conclusion comes first, each subsection answers one question, and unimportant parts are skipped. Name any subsection that just walks through the paper in order with no point.
- **Consistency with the card:** the deep dive doesn't contradict the TL;DR card, and it supports the card's key insight.

As with the card, point at problems and don't rewrite.

### Output format

```
## Review: <paper short name> — <total>/12

| Section | Score | Note |
|---|---|---|
| Problem | 2 | <one line: what's good or what's missing — no rewrite> |
...

### Factual errors
- "<quoted phrase from notes>" — paper says otherwise in §X / Fig Y: <what the paper says, briefly>
(or "None found.")

### Too close to the paper's wording
- "<quoted phrase>" (or "None.")

### Deep dive (omit this section if there's no deep dive)
| Check | Result | Note |
|---|---|---|
| Quote ratio | pass / needs work | ~N% quoted |
...

### Probing questions
1. <question aimed at the weakest section>
2. <question about a subtle point: an edge case, a "what breaks if…", or a converse that doesn't hold>
3. <question connecting to related work or one of Samuel's existing posts>

Answer these out loud or in chat. I'll respond after you do.
```

Pick probing questions that test understanding, not recall. Good forms: "Does the converse hold? Why not?", "What breaks if assumption X is dropped?", "Construct a small example where …", "How does this differ from <related idea>?"

### Follow-up

When Samuel answers the questions, say for each one whether the answer is right, partly right or wrong, and explain the gap. At that point it's fine to explain the concept fully. If an answer shows the notes should change, say *which section* and *why*. Samuel makes the edit.

## Mode 2: Recall quiz (`/paper-review recall`)

1. Find `[Morning Reading]` posts dated 7 or more days ago. Prefer ones with the lowest last score in the log, or ones never quizzed. Pick one, or the one Samuel names.
2. Tell Samuel the paper's title only, and say not to open the notes.
3. Ask 3 questions: the key insight, the mechanism, and one limitation or "what breaks if…". Ask them one at a time.
4. After the answers, compare them with the notes *and* the paper, and say what was remembered, what was lost, and what was wrong.
5. Log it with mode `recall`.

## Score log

After each review or recall, append one line to `.claude/paper-review-log.md`, creating the file with a header if it's missing:

```
| date | post | mode | Prob | Insight | How | Evid | Limit | Take | total |
| 2026-11-10 | 2026-11-09-lamport-clock | review | 2 | 1 | 2 | 1 | 0 | 1 | 7 |
```

For recall, put scores only in the relevant columns and use `-` for the rest. If the log has 5 or more entries, end your output with one line on the trend, e.g. "Limitations has averaged 0.8 over the last 5 reviews, and it's the weakest section."
