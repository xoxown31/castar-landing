---
lang: en
date: 2026-11-15 12:00:00 +0900
translation_key: a-percentile-not-a-similarity
title: "A Percentile, Not a Similarity: Recommending Roles to Actors"
description: "Raw image–text similarity can't be compared across roles. Castar ranks the roles it shows an actor by where that actor stands among everyone eligible for each role, does all model work ahead of time, and answers in about a millisecond."
summary: "A similarity of 0.31 means different things for different roles. We rank roles for an actor by percentile within each role's eligible pool, and precompute everything, including the one-line reasons."
---


Our [last post]({{ '/blog/2026/11/criteria-before-candidates/' | relative_url }}) was about recommending actors to a producer. This one is about the other direction: when an actor opens Castar, which roles should they see first?

It sounds like the same problem turned around. It isn't, for two reasons.

**There is no time to think.** A producer asks for a recommendation once per role and can wait a few seconds for a model to judge 30 candidates. An actor's list loads every time they open the page. Nothing slow can run at request time: no language model, no vision model, not even a text encoder.

**The scores have to be comparable across roles.** A producer's list compares many actors for one role. An actor's list compares one actor across many roles. That turns out to be the hard part.

## Why raw similarity doesn't work

Castar's search scores an actor against a description with an image–text model: the cosine similarity between the description's embedding and the actor's photos. Within one role, a higher cosine means a better visual match, and ordering actors by it is reasonable.

Across roles it means very little. Some descriptions sit close to almost every face in embedding space and give everyone a high cosine; others are specific and give everyone a low one. An actor's cosine of 0.31 might put them near the top for one role and below average for another.

<figure class="figure" role="group" aria-labelledby="fig1-cap">
  <div class="figure__frame">
    <svg class="fig-capped" viewBox="0 0 440 214" role="img" aria-labelledby="f1-title f1-desc">
      <title id="f1-title">The same similarity, two different percentiles</title>
      <desc id="f1-desc">Two rows of dots show the similarity scores of every eligible actor for two fictional roles. For role A the scores are low overall, and one actor's score of 0.31 is near the top, the 88th percentile. For role B the scores are high overall, and the same 0.31 is below the median, the 35th percentile.</desc>
      <g font-size="10.5" font-weight="600" letter-spacing="1.6" fill="#8e8e88">
        <text x="0" y="14">SIMILARITY OF EVERY ELIGIBLE ACTOR (ILLUSTRATIVE)</text>
      </g>
      <g font-size="13" font-weight="600" fill="#f2f1ec">
        <text x="0" y="62">Role A</text>
        <text x="0" y="142">Role B</text>
      </g>
      <g stroke="#3d3d3a" stroke-width="1">
        <line x1="70" y1="58" x2="430" y2="58"/>
        <line x1="70" y1="138" x2="430" y2="138"/>
      </g>
      <!-- x = 70 + (s-0.20)*1800 for s in [0.20,0.40] -->
      <g fill="#3d3d3a">
        <circle cx="88" cy="58" r="5"/><circle cx="106" cy="58" r="5"/><circle cx="124" cy="58" r="5"/><circle cx="133" cy="58" r="5"/><circle cx="142" cy="58" r="5"/>
        <circle cx="151" cy="58" r="5"/><circle cx="160" cy="58" r="5"/><circle cx="169" cy="58" r="5"/><circle cx="178" cy="58" r="5"/><circle cx="196" cy="58" r="5"/>
        <circle cx="205" cy="58" r="5"/><circle cx="223" cy="58" r="5"/><circle cx="241" cy="58" r="5"/><circle cx="268" cy="58" r="5"/><circle cx="304" cy="58" r="5"/>
        <circle cx="304" cy="138" r="5"/><circle cx="322" cy="138" r="5"/><circle cx="331" cy="138" r="5"/><circle cx="340" cy="138" r="5"/><circle cx="349" cy="138" r="5"/>
        <circle cx="358" cy="138" r="5"/><circle cx="367" cy="138" r="5"/><circle cx="376" cy="138" r="5"/><circle cx="385" cy="138" r="5"/><circle cx="394" cy="138" r="5"/>
        <circle cx="403" cy="138" r="5"/><circle cx="412" cy="138" r="5"/><circle cx="268" cy="138" r="5"/><circle cx="250" cy="138" r="5"/><circle cx="286" cy="138" r="5"/>
      </g>
      <g fill="#f2f1ec" stroke="#101010" stroke-width="1.5">
        <circle cx="268" cy="58" r="7"/>
        <circle cx="268" cy="138" r="7"/>
      </g>
      <g font-family="'IBM Plex Mono', ui-monospace, monospace" font-size="12" text-anchor="middle" fill="#f2f1ec">
        <text x="268" y="86">0.31 → 88th pct</text>
        <text x="268" y="166">0.31 → 35th pct</text>
      </g>
      <g font-family="'IBM Plex Mono', ui-monospace, monospace" font-size="11" fill="#8e8e88" text-anchor="middle">
        <text x="70" y="204">0.20</text><text x="250" y="204">0.30</text><text x="430" y="204">0.40</text>
      </g>
    </svg>
  </div>
  <figcaption id="fig1-cap"><b>Figure 1.</b> One actor, two fictional roles. The raw similarity is the same; where it stands among everyone eligible for the role is not. Numbers are illustrative.</figcaption>
</figure>

So we don't rank roles by similarity. We rank them by **percentile**: for each role, where this actor's similarity falls among all the actors who are eligible for that role.

## How a role gets a score

Everything below happens ahead of time, in the background sync that refreshes Castar's search index every ten minutes.

### A caption for each role

A role's description is written for actors to read, and it mixes appearance with pay, schedule, and personality. For each new or changed role, one language-model call rewrites the parts about appearance into **one short English sentence in the style of an image caption**, the same format search uses for its queries. If the description says nothing about appearance, the model returns nothing, and the role is marked as conditions-only. Captions are cached, so an unchanged role is never sent to the model twice.

The caption is then embedded with the same text encoder that search uses.

### Who counts as eligible

An actor is eligible for a role if their gender matches (unless the role accepts any), their age and height fall within the role's ranges with a margin of five years and five centimeters, and they have the role's required skills. **A missing value never excludes anyone**: an actor who left height blank isn't shut out of roles that ask for it.

The margins are looser than in search on purpose. A producer's search is a filter; here the actor is the one deciding, and a role that is one year outside their range is still worth showing them.

### Similarity, then percentile

For every role and every indexed actor, the sync computes the similarity between the role's caption and each of the actor's photos and keeps the best one. Then, within each role, it converts the score to a percentile against the **eligible pool**, the actors who pass that role's conditions. Ties are split evenly (a mid-rank), so actors with the same score get the same percentile.

All of this is stored as one matrix per sync. At request time, serving an actor's list is a lookup and a sort, about a millisecond on the server.

## What an actor sees

Roles with a meaningful percentile come first, highest percentile first. A deadline within the next seven days breaks ties. Roles shown without a visual match follow, ordered by deadline.

A role falls back to conditions-only for several reasons: it has no appearance description, its caption hasn't been built yet, the actor has no usable photo, or **fewer than 10 actors are eligible**, which is too few to compare against. And one more: **if the actor's similarity is below the median of the role's eligible pool, we don't show a visual match at all.** The role still appears if the actor meets its conditions, but without a claim about fit. As a result, any visual match an actor sees is in the upper half of the pool. We see no reason to tell someone they're in the bottom third for a role.

## Reasons, written ahead of time

For each role shown with a visual match, the actor also sees one sentence about why. Writing that sentence takes a vision-language model, which can't run at request time, so these too are generated during the sync and stored.

The model gets the actor's photo that best matches the role, the role's name and its caption, and writes one short, friendly sentence naming one or two visible features that relate to the role. It is not allowed to mention numbers, age, height, weight, attractiveness, or personality read from a face, or to compare anyone with a real person, and it may not overclaim. The tone depends on the percentile: at or above the 90th percentile, a sentence can say the actor *suits* the role; below it, the most it can say is that something *resembles* the role.

Two practical problems came up:

- **Repetition.** Thirty actors seeing the same role shouldn't all read the same sentence. Sentences for one role are written one after another, and a new sentence too close to a recent one is rejected and rewritten with the recent ones shown as examples to avoid.
- **Failures.** Every sentence goes through a validator. A sentence that fails gets one rewrite with the reason it was rejected; if that fails too, the actor sees a plain template instead, such as *"Meets the conditions (age, height) · top 12% for the role's look."*

In a development run on real data, 82 actor–role pairs took about a minute: 80 model-written sentences, all distinct, and 2 template fallbacks.

## What this doesn't solve yet

- **One sentence carries the whole role.** The caption is a single English sentence. Whatever it leaves out of a long description doesn't count toward the match.
- **A percentile is only as good as its pool.** With ten eligible actors, one actor joining or leaving moves everyone's percentile noticeably. The minimum pool size limits this but doesn't remove it.
- **Percentiles are relative.** The top actor for a role nobody fits well still gets a high percentile. The median cutoff stops the low end from being shown, not the top end from being overstated.
- **Up to ten minutes stale.** A new role or a new photo appears after the next sync.
- **We haven't measured fit yet.** The natural test is whether actors apply more often to roles near the top of their list. We'll report it when we have data we trust.

## Next

Across this series, the same idea keeps coming back: decide what a model is allowed to answer, and then read its answer carefully. We'll keep writing about the parts of Castar where that idea gets tested.
