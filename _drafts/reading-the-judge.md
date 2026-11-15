---
lang: en
translation_key: reading-the-judge
title: "Reading the Judge: Teaching Appearance Search What “Cute” Looks Like"
description: "Zero-shot image–text search reads impression words poorly. A vision-language model judges them far better, but too slowly to run per query. We studied how to turn its judgments into training signal, and found that the way an answer is read matters more than its format."
summary: "A VLM can judge impressions like “cute” or “well-groomed” much better than zero-shot search. Descriptions waste that judgment; ratings and rankings keep it, as long as you read the answer properly."
---

<!-- AUTHOR NOTES (delete before publishing)
- Numbers come from the team's ARR Oct 2026 submission (Tables 1, 7, 8). Re-check against the PDF before publishing.
- All experiments use One Million Impressions (OMI, CC BY-NC-SA). Do NOT add OMI face images to this post. Diagrams only.
- The post deliberately makes NO claim about which concepts the production adapter uses. Do not add one unless it is checked against the deployed adapter.
- Before publishing: check which concepts the production adapter was trained on, and get the team's and the supervising professor's agreement (it is published under the company name and reports a bias audit).
-->

Our [last post]({{ '/blog/2026/09/from-a-casting-call-to-search-conditions/' | relative_url }}) ended with half a casting call left over. A call such as *"a sweet-faced woman in her twenties"* splits cleanly into conditions a profile field can answer (female, 20–29) and a residue that no field can: *sweet-faced*. That residue goes to an image–text channel, which ranks the actors who passed the filter by how well their photos match it.

This post is about that channel, and about a question we spent most of the autumn on: **how do you teach a search model what an impression word means, when the only reliable judge is too slow to put in the search path?**

The short version: a vision-language model (VLM) can judge impressions far better than zero-shot image–text search, and its judgments can be distilled offline into something cheap enough to run on every query. But the *form* in which we ask for those judgments matters less than we expected. What matters is **how the answer is read**.

Every experiment in this post was run on a public research dataset of synthetic faces, not on Castar actors. Details are at the end.

## The information is in the image, not in the query

Image–text models such as SigLIP put photos and sentences in one embedding space, so a query can be scored against every photo with a single dot product. They work well for things the web describes literally: *short hair*, *glasses*, *smiling*. Impression words are different. "Cute," "dorky" or "trustworthy" are matters of taste and degree, and the text encoder learned them from how often they co-occur with images in web captions, not from anyone deciding which face is cuter.

We measured how big the gap is. On a dataset where hundreds of human raters scored each face for dozens of impressions, a zero-shot text query reaches **NDCG@10 of .625** on phrasings it has never seen (random ordering gives .521). That is not because the image embeddings lack the information: a simple linear probe trained on the *same frozen image embeddings* with human ratings reaches **.935**. The faces are separable. The query side just doesn't know where to look.

So the job is not to build a better image model. It is to teach the query side what humans mean, and we don't have human ratings for every word a producer might type.

## A judge that is too slow to ask

A VLM can look at a face and answer "how cute is this person?" much like a human rater. The problem is speed. Reranking 40 candidates per query with a VLM took about **10.5 seconds** in our setup; Castar's search returns in about one second. A judge that good cannot sit in the search path.

That leaves the obvious plan: ask the judge **offline**, once, over a fixed set of faces and impressions, and train the fast model on its answers. The open question is what to ask for.

## Three ways to ask

We compared three forms of the same question, all posed to the same judge (Gemma 4 12B, temperature 0):

- **Describe.** "Describe this person's appearance and impression." Then search over the description.
- **Rate.** "How *cute* is this person, from 1 to 5?"
- **Rank.** Show ten faces of the same gender and ask the judge to order them by *cute*. Each group is asked forward and reversed to cancel position bias.

Each answer then trains the same small model, and we score it on phrasings held out from training.

<figure class="figure" role="group" aria-labelledby="fig1-cap">
  <div class="figure__frame">
    <svg class="fig-capped" viewBox="0 0 440 268" role="img" aria-labelledby="f1-title f1-desc">
      <title id="f1-title">Ranking quality by supervision signal</title>
      <desc id="f1-desc">Horizontal bars of NDCG at 10 on held-out phrasings. Zero-shot query .625. Free description .680. Rating, greedy digit .781. Rating, expected score .815. Ranking ten faces .810. Human rankings, reference, .848. Random ordering is .521.</desc>
      <g font-size="10.5" font-weight="600" letter-spacing="1.6" fill="#8e8e88">
        <text x="0" y="14">SIGNAL</text>
        <text x="440" y="14" text-anchor="end">NDCG@10, HELD-OUT PHRASINGS</text>
      </g>
      <!-- axis: x = 190 + (v-0.5)*440 -->
      <g stroke="#3d3d3a" stroke-width="1">
        <line x1="199" y1="26" x2="199" y2="236" stroke-dasharray="3 3"/>
        <line x1="234" y1="26" x2="234" y2="236"/>
        <line x1="322" y1="26" x2="322" y2="236"/>
        <line x1="410" y1="26" x2="410" y2="236"/>
      </g>
      <g font-family="'IBM Plex Mono', ui-monospace, monospace" font-size="11" fill="#8e8e88" text-anchor="middle">
        <text x="199" y="254">.52</text><text x="234" y="254">.60</text><text x="322" y="254">.80</text><text x="410" y="254">1.0</text>
      </g>
      <g font-size="13" fill="#b4b3ac">
        <text x="0" y="52">Zero-shot query</text>
        <text x="0" y="86">Describe</text>
        <text x="0" y="120">Rate, greedy digit</text>
        <text x="0" y="154" fill="#f2f1ec" font-weight="600">Rate, expected score</text>
        <text x="0" y="188" fill="#f2f1ec" font-weight="600">Rank ten faces</text>
        <text x="0" y="222">Human rankings (ref.)</text>
      </g>
      <g>
        <rect x="190" y="38" width="55" height="20" rx="3" fill="#2b2b29" stroke="#8e8e88"/>
        <rect x="190" y="72" width="79" height="20" rx="3" fill="#2b2b29" stroke="#8e8e88"/>
        <rect x="190" y="106" width="124" height="20" rx="3" fill="#2b2b29" stroke="#8e8e88"/>
        <rect x="190" y="140" width="139" height="20" rx="3" fill="#f2f1ec"/>
        <rect x="190" y="174" width="136" height="20" rx="3" fill="#f2f1ec"/>
        <rect x="190" y="208" width="153" height="20" rx="3" fill="none" stroke="#b4b3ac" stroke-dasharray="4 3"/>
      </g>
      <g font-family="'IBM Plex Mono', ui-monospace, monospace" font-size="12" fill="#f2f1ec">
        <text x="251" y="53">.625</text>
        <text x="275" y="87">.680</text>
        <text x="320" y="121">.781</text>
        <text x="335" y="155">.815</text>
        <text x="332" y="189">.810</text>
        <text x="349" y="223">.848</text>
      </g>
    </svg>
  </div>
  <figcaption id="fig1-cap"><b>Figure 1.</b> The same judge asked three ways. Descriptions recover little of what the judge knows. Ratings and rankings recover most of it, but only if the rating is read as an expected score rather than a single digit. The dashed line at .52 is random ordering. Numbers are from our experiments on a public synthetic-face dataset.</figcaption>
</figure>

### Descriptions throw the judgment away

Descriptions were the weakest signal, at .680. Reading them, the reason is clear: the judge describes almost everyone the same way. The word *approachable* appeared in **92%** of descriptions. A description answers "what is this face like?", not "how much of *this* impression does it have compared with the next face?", and the second question is the one search needs. Even when we let a language model read the descriptions and score them, the result (.736) stayed below a plain 1–5 rating.

### Ratings work, if you read them properly

A 1–5 rating, taken at face value, reached .781. Taking it at face value is the problem. When the judge answers with a single digit, most faces get a 3 or a 4, and **51.6%** of pairs within a group came out tied. A tie carries no ranking information at all.

But the judge doesn't really believe in one digit. Its output is a probability distribution over "1" to "5", and a face it calls "4" might be 60% "4" and 40% "5". Reading the **expected score**, Σ k·p(k), instead of the most likely digit turns those ties into an ordering. Same judge, same prompt, same number of calls: .781 becomes **.815**.

### Rankings work just as well, and need less

Ranking ten faces at a time reached **.810**, statistically indistinguishable from the expected-score rating (the difference was −.004, with a 95% interval of −.047 to .038). Rankings have a practical edge: they need only text output, not token probabilities, and fewer calls. One round of rankings over our 716 faces took 1,008 calls, about eight minutes of serving. Eight rounds of fresh groups added only +.003.

So the honest headline is not "compare, don't rate." It is: **once the question names the impression, how the answer is read matters more than its format.** A ranking is read in full by construction. A rating has to be read as a distribution to keep the same information. A description is read by a second model, and most of the signal is gone by then.

## Storing the judge's answer

The judgments still have to live somewhere fast. We tried two places:

- **A LoRA adapter on the text tower.** A small low-rank update (1.34M trainable parameters, about 17 seconds to train per adapter) that changes how queries are embedded. The image tower stays frozen, so the photo index never has to be rebuilt. On held-out phrasings it reached .816.
- **A ridge head per impression on the image embeddings**, with a language model routing each query to the right head. This kept more of the judgment: **.884** on held-out phrasings and **.874** on real casting-call phrasings, up from .625 and .686 zero-shot.

For reference, the same heads trained on *human* rankings instead of the judge's reach .936. Human labels still lead by about .05. A judge gets most of the way there with no human labeling for the impressions it was asked about.

Put together with the filters from our last post, the full pipeline on casting-call queries went from .520 with zero-shot ranking to .680.

## What this doesn't solve yet

- **One judge wrote part of its own exam.** The held-out phrasings were generated by the same model family that judged the faces. We also tested on 84 queries written independently; the ordering of methods held (zero-shot .665, expected score .802, rankings .797, human .816), but the gap to human labels was smaller and the test is small.
- **Ratings versus rankings is still open.** Seven impressions are not enough to tell the two apart, and the adapter's hyperparameters were tuned on rankings, which may favor them.
- **It doesn't generalize to impressions nobody asked about.** Improvements on untrained impressions were mostly a side effect of learning *smiling*. Impressions the judge never saw moved from .534 to .547, which is nothing.
- **Not every impression improved.** *Dorky* got worse after training (.811 → .778).
- **One dataset, two mid-sized judges.** The faces are synthetic, rated mostly by US raters. Taste in a Korean casting context may differ.
- **Accuracy hides exposure.** In our research runs, the trained adapter's top results skewed toward slimmer faces and showed darker skin tones less often than before training, even as its accuracy went up. An accuracy number alone would never have shown this.

## The judge will answer questions it shouldn't

The most uncomfortable result of this work wasn't a number in the main table. Asked to rate faces one at a time, the judge declined to rate 57% of women for *attractive* and 14% for *smart*. Asked to rank ten faces for the same words, it declined nothing. Changing the framing was enough to get past its safeguards.

That matters for anyone building on this idea, us included. A judge that will rank faces by any word you give it doesn't tell you which words it should be given. Some words describe what a photo shows: *smiling*, *long hair*, *well-groomed*. Others, like *trustworthy* or *smart*, describe a person, and a face is not evidence of them. In between sit words like *attractive*, which are about taste and are exactly where a dataset's raters, and their biases, decide the answer.

We don't think a model's willingness to answer should settle which of these belong in search, and we don't think accuracy should either. It's a question about what a casting platform should let people filter on, and casting is close to hiring. These findings are an input to that question, not an answer to it. They are also a strong argument for auditing exposure next to accuracy before any adapter reaches users.

## Details

Experiments used One Million Impressions (Peterson et al., 2022): 1,004 synthetic StyleGAN2 faces, each rated by at least 30 people on 34 attributes. We used the 716 adult faces, grouped into five gender × age cells, with relevance taken as the mean human rating within a cell and five-fold splits over faces. The image–text model was SigLIP-B/16; the judges were Gemma 4 12B and, as a check, Qwen3.5-9B. All judging took about nine GPU-hours. No Castar actor photos were used, and none are shown here.

## Next

This post covered how a query learns what an impression looks like. The next one covers what happens after search: how Castar takes the shortlist for a role and has a language model write criteria for that role, then asks a VLM to score each candidate against them, using the same lesson about reading the judge's answer.
