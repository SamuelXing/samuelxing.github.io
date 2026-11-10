---
name: morning-reading
description: Set up a new "[Morning Reading]" paper post skeleton in _posts/ when Samuel gives a paper name (e.g. "/morning-reading Paxos Made Simple", "morning reading: <paper>", "today's paper is <paper>"). Creates the file only — Samuel writes the notes himself.
---

# Morning Reading setup

Samuel reads one paper each morning and takes notes in an unpublished post. Your job is **setup only**: create the skeleton so Samuel can start writing. Do NOT fill in the note sections — answering them is the point of the exercise. (If Samuel explicitly asks for help with content, that's a separate request.)

## Steps

1. **Identify the paper.** From the name given (may be partial or misspelled), determine the canonical title, authors, year, and venue. If you're not certain, do a quick web search. Find a freely accessible PDF link if one exists (author page, ACM/USENIX open access, arXiv).

2. **Check for duplicates.** `grep -ril` the title and key terms in `_posts/`. If a post on this paper already exists, stop and tell Samuel instead of creating a second one.

3. **Pick the slug and date.**
   - Date = today (local date).
   - Slug = short kebab-case name for the paper (e.g. `lamport-clock`, `paxos-made-simple`, `raft`).
   - File: `_posts/YYYY-MM-DD-<slug>.md`
   - Permalink: `/posts/YYYY/MM/<slug>/` (same year/month as the date).

4. **Pick tags.**
   - 2–3 topic tags. Reuse existing tags where possible; list them with `grep -h "^  - " _posts/*.md | sort | uniq -c | sort -rn | head -40`.
   - Always include `paper-reading`.
   - Course tag: copy it from the most recent `[Morning Reading]` post (`grep -l "Morning Reading" _posts/*`), e.g. `UIUC CS 525`. Use a different one only if Samuel says so.

5. **Write the file** using exactly this structure (the six headings mirror `_posts/Reading-template.md`):

```markdown
---
title: '[Morning Reading] <Canonical Paper Title>'
date: YYYY-MM-DD
permalink: /posts/YYYY/MM/<slug>/
published: false
tags:
  - <topic-tag>
  - <topic-tag>
  - paper-reading
  - <course tag>
---

**Paper:** [<Title>](<pdf link>) — <Authors>, <Venue> <Year>

<details markdown="1">
<summary><b>Detailed Notes</b></summary>

### Problem — what problem, and why is it hard? (1-2 sentences)

### Key insight — the one idea that makes the solution work (1 sentence)

### How it works — the mechanism, at a level you could re-explain (2-4 sentences)

### Evidence — how do they show it works? (eval, proof, or argument) (1-2 sentences)

### Limitations & assumptions — what does it assume, and where does it break? (1-2 sentences)

### Takeaways — impact, connections to other papers, open questions (1-2 sentences)

</details>
```

If no PDF link was found, link the DOI or omit the link.

6. **Report back** briefly: the file path as a clickable link, the citation line, and any existing posts in `_posts/` that relate to this paper (e.g. a post on a system that builds on it), so Samuel can cross-link. Don't commit.
