---
lang: en
date: 2026-11-15 11:00:00 +0900
translation_key: criteria-before-candidates
title: "Criteria Before Candidates: How Castar Recommends Actors for a Role"
description: "For each role, a language model first writes the criteria that matter and how much each one weighs. Only then does a vision-language model score every shortlisted actor against them. Search similarity never enters the score."
summary: "The model writes a role-specific rubric first, then scores each candidate against it. Every point in the final score traces back to a named, weighted criterion."
---


Search answers a question someone typed. Recommendation answers one nobody typed: *a producer has posted a role, so who should they look at first?*

The role is already on Castar: a name, gender, age and height ranges, required skills, and a free-text character description written for applicants. Search could rank actors against that description, and it does, as a first pass. But "how well does this photo match a caption" is a narrow notion of fit. The same actor can be a strong match for *a hard-boiled detective in his forties* and a weak one for *a gentle father in his forties*, and the difference is not one similarity score. It is a different set of things to look for.

So Castar's recommendation splits the job in two. **First, a model writes the criteria for this role. Then a second model scores each candidate against those criteria.** This post walks through both steps, and why the search score doesn't count toward the result.

Every role, criterion and output in this post is made up for illustration.

## The pipeline

<figure class="figure" role="group" aria-labelledby="fig1-cap">
  <div class="figure__frame">
    <svg class="fig-capped" viewBox="0 0 440 300" role="img" aria-labelledby="f1-title f1-desc">
      <title id="f1-title">Two-stage role recommendation</title>
      <desc id="f1-desc">A posted role feeds two branches that run in parallel. Stage 1: strict filters from the role's fields, then search ranking, keeping the top 30. Planner: one language-model call writes three to five weighted criteria. The judge then scores each of the 30 candidates against the criteria, one call per candidate, and the candidates are sorted by weighted score, each with a one-line reason.</desc>
      <defs>
        <marker id="r1a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#8e8e88"/></marker>
      </defs>
      <rect x="150" y="4" width="140" height="34" rx="17" fill="#2b2b29" stroke="#ffffff" stroke-width="1.1"/>
      <text x="220" y="26" font-size="13.5" font-weight="600" text-anchor="middle" fill="#f2f1ec">Posted role</text>
      <g fill="none" stroke="#8e8e88" stroke-width="1.2">
        <path d="M200,38 C200,56 110,52 110,70" marker-end="url(#r1a)"/>
        <path d="M240,38 C240,56 330,52 330,70" marker-end="url(#r1a)"/>
      </g>
      <g fill="#161616" stroke="rgba(255,255,255,0.18)">
        <rect x="0.5" y="74.5" width="219" height="110" rx="10"/>
        <rect x="236.5" y="74.5" width="203" height="110" rx="10"/>
      </g>
      <g font-size="10.5" font-weight="600" letter-spacing="1.6" fill="#8e8e88">
        <text x="14" y="96">STAGE 1 · SHORTLIST</text>
        <text x="250" y="96">PLANNER · CRITERIA</text>
      </g>
      <g font-size="12.5" fill="#b4b3ac">
        <text x="14" y="120">Strict filters from the role</text>
        <text x="14" y="140">Search ranking (photos)</text>
        <text x="14" y="168" fill="#f2f1ec" font-weight="600">Top 30 candidates</text>
        <text x="250" y="120">One LLM call, text only</text>
        <text x="250" y="140">3–5 criteria, weights = 100</text>
        <text x="250" y="168" fill="#f2f1ec" font-weight="600">appearance · fact</text>
      </g>
      <g fill="none" stroke="#8e8e88" stroke-width="1.2">
        <path d="M110,184 C110,204 200,198 200,216" marker-end="url(#r1a)"/>
        <path d="M338,184 C338,204 240,198 240,216" marker-end="url(#r1a)"/>
      </g>
      <rect x="90.5" y="220.5" width="259" height="74" rx="10" fill="#101010" stroke="#b4b3ac"/>
      <text x="104" y="242" font-size="10.5" font-weight="600" letter-spacing="1.6" fill="#8e8e88">JUDGE · ONE CALL PER CANDIDATE</text>
      <g font-size="12.5" fill="#b4b3ac">
        <text x="104" y="264">One photo + profile facts → 1–5 per criterion</text>
        <text x="104" y="284" fill="#f2f1ec" font-weight="600">Weighted score + one-line reason</text>
      </g>
    </svg>
  </div>
  <figcaption id="fig1-cap"><b>Figure 1.</b> Stage 1 and the planner run in parallel. The search ranking decides <i>who</i> gets judged; the criteria decide <i>how</i>. Only the judge's scores make up the final order.</figcaption>
</figure>

### Stage 1: a shortlist from search

The role's structured fields become a query and go through the same parser and search described in our [first post]({{ '/blog/2026/09/from-a-casting-call-to-search-conditions/' | relative_url }}). Two details differ from search:

- **The role's own fields win.** Gender, age and height come from what the producer entered, not from what the parser reads in the description. Required skills come only from the role's required-skills field. Skills the parser finds in the description become *preferred*: they move people up but never exclude anyone.
- **Conditions can loosen.** Search never relaxes a condition; an empty result is an answer to the person who typed the query. Recommendation is different. Nobody typed this query, and a role recommendation with no one in it doesn't help a producer. So if fewer than 10 actors pass, stage 1 widens height (±5, then ±10 cm), then age (±5, then ±10 years, never below 18), then drops required skills one at a time, keeping each dropped skill as a preference. This is a trade-off, and we list it among the open problems below.

The surviving actors are ranked by search, and the top 30 go on to be judged.

### The planner: write the rubric first

While stage 1 runs, one text-only call to the language model reads the role and the project it belongs to (title, category, synopsis) and writes **three to five criteria**. Each criterion has a short label, a one-sentence definition, a type, and a weight. The weights are whole numbers that add up to exactly 100.

The type says what a criterion may be judged from:

- **appearance**: something visible in one profile photo, such as build, face shape, hair, or overall impression.
- **fact**: something in the profile fields, such as age, height, weight, or skills.

For a fictional role, *a detective in a violent-crimes unit, male, 38–50, action experience preferred, "a worn, sharp-eyed veteran"*, a plan could look like this:

<div class="table-wrap" markdown="1">

| Criterion | Type | Weight |
|---|---|---|
| Solid, sturdy build | appearance | 30 |
| Sharp, weathered impression | appearance | 30 |
| Action or martial-arts skills | fact | 25 |
| Age fits a veteran officer | fact | 15 |

</div>

A few rules keep the rubric honest. If the description says "someone like [a famous actor]," the planner has to turn that into concrete visual features; real names are removed if they slip through, and no criterion asks for resemblance to a person. Personality traits are only allowed when the profile can back them up, and then they count as facts; "kind" is not something a face shows. The planner is also told which filters stage 1 already applied, so it can decide whether a condition still deserves weight.

If the planner's output can't be repaired into at least three valid criteria, we fall back to two generic ones: fit to the role image (appearance, 60) and fit to the conditions (fact, 40).

### The judge: one candidate at a time

Each of the 30 candidates then gets one multimodal call. The judge sees:

- **one photo**, the one that matched the role best in stage 1, resized and held in memory only;
- **the role's conditions**, without the free-text description;
- **the criteria**, each marked "judge from the photo" or "judge from the profile facts";
- **profile facts only**: age, height, weight and skills. No names and no other fields.

It answers with a score from 1 to 5 for each criterion, against fixed anchors. A 3 means *average for the actors who usually apply to a role like this*; a 5 means *exceptional, roughly the top 10%*. Appearance criteria are judged from the photo alone and fact criteria from the facts alone.

We don't take the digit at face value. As in our [last post]({{ '/blog/2026/11/reading-the-judge/' | relative_url }}), a single digit throws information away: two candidates who both get a "4" may not look equally strong to the judge. We read the probabilities the model assigns to "1" through "5" and use the expected value, so a score comes out as 3.2 or 4.6 rather than a flat 3 or 4. This costs nothing extra; the probabilities come back with the same call.

The final score is the weighted mean of those per-criterion scores, mapped from 1–5 onto 0–100.

## Why search similarity doesn't count

Stage 1 ranks candidates by how well a photo matches the role's text. It would be easy to blend that similarity into the final score. We don't. **The final order uses only the judge's scores**; the stage-1 rank is kept only to break ties.

There are two reasons. A similarity between a photo and a caption is not a measure of fit, and mixing it in lets a high similarity cover for a weak score on a criterion that matters. More importantly, it would make the score impossible to explain. As it is, every point in a candidate's total traces back to a criterion with a name, a definition and a weight, so the score can always be explained in terms of the role. Stage 1's job is to decide who gets looked at closely. It doesn't get a vote after that.

## One line of reasons

Along with the scores, the judge writes one line about each candidate for the producer. The line has to name one visible feature and one fact from the profile, and it has to be positive: it says why this candidate is worth a look, not what is wrong with them. A validator checks every line. It rejects lines without a visible feature or without a fact that actually matches the profile, and lines that comment on attractiveness or read personality from a face. A rejected line gets one retry; if that fails too, a plain template built from the profile facts is used instead.

## Who is in the pool

Only actors who have agreed to AI features, are 18 or older, and have a public, completed profile are indexed at all, so only they can be recommended. Scores and reasons go to the producer's dashboard and are never shown to actors.

A recommendation for one role takes roughly 7 to 17 seconds in our development measurements: one to two seconds for stage 1, and the rest for 10 to 30 judge calls running six at a time. It runs once when a producer asks for it, not on every page load.

## What this doesn't solve yet

- **We haven't measured it.** There is no evaluation of recommendation quality yet. The obvious first test is how often the candidates producers actually contact appear in the top of the list, and we'll write about it when we have numbers.
- **Loosening conditions is a real trade-off.** It keeps a recommendation from coming back empty, but it can surface someone just outside what the producer entered. How to show that clearly is still open.
- **One photo per candidate.** The judge sees the single photo that best matched the role in stage 1. A candidate whose best photo for this role is a different one can be underrated.
- **Scores are judged one candidate at a time.** Anchors keep the 1–5 scale steady across calls, but the judge never sees two candidates side by side. Our research suggests comparing faces directly carries as much information as reading ratings carefully; we haven't tested that here.
- **The rubric is only as good as the role description.** A sparse description gives the planner little to work with, and the fallback rubric is generic by design.
- **Same model, two jobs.** The planner and the judge are the same language model. Errors in how it reads a role can show up twice.

## Next

Recommendation also runs the other way: actors on Castar see roles that fit them. That side has very different constraints, with no time for a model call at request time and scores that must make sense across roles, and it will be the subject of our next post.
