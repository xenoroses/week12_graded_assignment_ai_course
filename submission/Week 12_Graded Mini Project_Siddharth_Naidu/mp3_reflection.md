# MP3 Reflection

## Grounding & abstention

I started with the given `RETRIEVAL_THRESHOLD = 0.30` and printed the top retrieval score for every question before trusting it. The two out-of-corpus questions that sounded believable scored a lot higher than I expected. The Moriarty question got 0.482 (the top chunk was a "criminal gets caught" passage from the Red-Headed League capture scene), and my own abstain question about Toby the tracking dog got 0.480. Both are well above 0.30, so with the default they would have been answered from outside knowledge instead of abstaining. The six answerable questions scored between 0.528 and 0.689.

So I raised the threshold to 0.50, which sits in the small gap between those two groups. A truly unrelated question ("What is the boiling point of water?") only scored 0.144, so the threshold really only mattered for the two tricky ones that sound like Sherlock Holmes but aren't in the corpus.

With that, abstention accuracy was 1.0 on all three should-abstain questions and hallucination rate was 0.0. Nothing got answered from outside knowledge.

The more interesting problem went the other way. Three of my six answerable questions (q1, q2, q5) also abstained, even though retrieval pulled chunks from the right story each time. For q1 I printed the retrieved text and saw that the top 3 chunks only mentioned the alias "Vincent Spaulding", not the real identity "John Clay". That reveal is in a later chunk ("The Capture") that ranked 6th, outside k=3. So the model said "I don't know" instead of guessing. This isn't a bug in `should_abstain`, since the score gate passed (scores were 0.55 to 0.69). The model itself refused because the evidence it was given didn't have the answer. That's correct behaviour, but it means my setup can't tell the difference between "this isn't in the corpus" and "the right document is there but my top-k missed the paragraph."

## Citations

`validate_citations` never caught a fake citation across the nine live questions. gpt-4o-mini always stayed within the numbers it was given (q3 cited [1][2] from the two "The Photograph" chunks, and q6 cited [3] for the James Ryder confession). To check that the validator actually works, I used `selftest`, which passes in an out-of-range marker (`validate_citations("An outside fact [3].", 2)` should flag `invalid == [3]`). That passes, so I trust it would catch a real one.

Since the model abstained on 3 of 9 questions, I think the system prompt ("If the excerpts do not contain the answer, reply with exactly...") made it lean toward under-answering instead of making things up. That's the safer way to fail for this kind of assistant.

## Reading the eval table

```
Retrieval hit rate    : 1.0   (6 answerable questions)
Grounded & cited rate : 0.5   (KPI: grounded response rate)
Abstention accuracy   : 1.0   (3 should-abstain questions)
Hallucination rate    : 0.0   (KPI: hallucination rate)
```

Retrieval hit rate and abstention accuracy are both perfect, and hallucination rate is 0.0, which is the best it can be. On this small corpus, retrieval reliably finds the right document and the tuned threshold stops the system from guessing on out-of-corpus questions.

The weak number is grounded & cited rate at 0.5. Half of the answerable questions were answered and cited properly (q3, q4, q6), and the other half abstained even though the right document was retrieved (q1, q2, q5). In each of those cases the fact needed for the answer was outside the top 3.

If I improved one thing, I'd look at k and chunking. Raising k alone might not be enough, because for q5 the missing fact only shows up at rank 6, so even k=5 would miss it. A better fix is probably chunking so the setup and the resolution of a story aren't split so far apart. Right now a 500-character target breaks one Holmes case into about 13 pieces, and the reveal chunk scores lower against the question because the question uses setup wording, not reveal wording.

## One surprise

I expected the abstention failures to look like hallucination, like the model confidently answering the Moriarty question from what it already knows. The real problem was the opposite and harder to notice. The system abstained too often on real, in-corpus questions when the retrieved chunks didn't have the exact fact. In the eval table, "abstained because it's not in the corpus" and "abstained because top-3 missed the paragraph" look exactly the same (both show `abstained: ✓`). I only found the difference by printing the retrieved chunks for each failing question by hand. A better eval would need a third category, something like "retrieved but incomplete," so you could tell these apart without checking manually.