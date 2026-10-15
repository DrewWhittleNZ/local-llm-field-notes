# From Faster Answers to Better Decisions

## A plain-language account of what we learned building and testing a local AI investment-analysis system

**Status:** Exploratory plain-language companion  
**Research began:** 23 September 2026  
**Prepared by:** Codex  
**Technical source:** [*From Tokens per Second to Decision Quality*](from-tokens-per-second-to-decision-quality.md)  
**Public-safe evidence extract:** [Measurements and provenance](evidence-and-provenance.md)  
**Last updated:** 16 October 2026

> “ELI5” here means explain it with ordinary words and useful pictures in your head. It does not mean talking down to the reader, removing important warnings, or pretending the problem is simpler than it is.

## Who this is for

This is for somebody who is not an AI specialist but has an idea and is trying to make it work.

You might know how to install software, follow instructions, edit a configuration file, or ask an AI coding assistant for help. You may not know how a language model uses memory, what a KV cache is, or how a formal AI speed test is designed. That is fine. We did not begin this work because we wanted to become AI performance specialists. We began because we wanted a useful investment-analysis system and it was slow, sometimes wrong, and difficult to understand.

The useful lesson is not “learn all the AI terminology first.” It is:

> Start with one real thing you want the system to do, keep examples of correct and incorrect answers, measure what actually happens, and change one part at a time.

You can skip the number tables on a first read. They show that the claims were measured; the surrounding explanation says why the measurements matter.

If you know somebody with deeper experience in local AI systems, the linked technical report is the version to give them. It includes enough method and measurement detail for them to challenge our explanation, point out something we missed, or propose a better test. This version and that one use different language, but they are required to keep the same facts and conclusions.

## A few words you need before we start

You do not need a machine-learning background. These plain definitions are enough:

| Word | What it means here |
|---|---|
| **Model** | The AI “brain” producing the words |
| **Weights** | The many numbers the model learned during training. They are stored as part of the model itself, unlike the company documents we give it to read. A larger model usually has more of them. |
| **Prompt** | The question and instructions we give it |
| **Context** | Everything the AI can currently see: instructions, facts, tool results, earlier work, and its growing answer |
| **Token** | A small piece of text; computers count these more easily than words |
| **Reasoning** | The AI's hidden scratch work before or around its final answer |
| **Ollama** | The program on our Mac that loads the AI model, runs it locally, and passes its answers back to our project |
| **Runtime** | The software doing that work; in our tests, Ollama and the engine it selected |

One more technical term appears later: **KV cache**. For now, think of it as the model's working-memory cards. We will explain it when it matters.

**Two Macs, not two AI models:** M1 and M6 are the names of the Apple chips in the Mac minis we used. We started on an older M1 Mac mini with 8 GB of memory, then moved to a newer M6 Mac mini with 32 GB. That memory is shared by the computer and its graphics processor, which runs the AI. When a result differs between the two Macs, the chip is not necessarily the only cause: the model, Ollama version, and other settings may have changed too. The names *4B*, *9B*, and *27B* later in this report refer to AI model sizes, not Mac models.

## The very short version

We built a local AI system that reads company information and helps write investment reports.

It was taking a very long time.

At first, we thought the obvious fixes were:

- make the AI write less;
- give it less information;
- turn off its hidden thinking; or
- buy a faster computer.

The real answer was more interesting:

- The AI was doing lots of invisible scratch work before showing its answer.
- A recorded short test found that turning off that scratch work made it much faster but produced a wrong financial comparison; its original raw answers were not saved.
- Giving it more pages to remember made every new word slower, not just the initial reading.
- On the old M1 setup, a memory-saving mode called Q8_0 made that slowdown much worse than F16, and a second controlled run reproduced it.
- The new 32 GB M6 did **not** reproduce that F16 advantage on the tested llama.cpp path. Q8_0 and F16 ran at practically the same speed there, so Q8_0 remained the memory-efficient choice for that path; the current MLX runner has not confirmed which cache precision it actually applies.
- Sometimes the AI looked weak because we had not actually given it the facts needed to answer the question.
- Our carefully shortened prompt ultimately failed its safety test: it saved only 6.3% of the time and still omitted required facts.
- On one dense job, forcing hidden reasoning made the AI think until the safety limit without ever answering. Turning reasoning off made that job finish, even though an earlier financial calculation had needed reasoning to be correct.
- The M6 made the 4B model much faster, but a fully loaded 27B model was still slow. It also repeated a factual mistake made by 4B, while 9B avoided it. A bigger model was therefore not automatically a better model.
- An Ollama update silently changed the model files and engine under the same name. The completed repeated tests show that restarting the newer MLX model at the right boundaries prevents a large memory slowdown. They also show changing scores and occasional malformed reports on the *same saved facts*, so they do not declare a quality winner.

So we did not simply choose the fastest setting.

We kept the known-safe setup and built a fair test that asked two questions at once:

1. Is it faster?
2. Does it still reach the correct financial decision?

That second question is the important one.

## If you remember only five rules

1. **Save one real example and its correct answer.** That becomes your test case.
2. **Check that the AI received the needed facts.** Missing evidence is not proof of a weak model.
3. **Change one thing at a time.** Otherwise you will not know what caused the result.
4. **Verify what the software actually used.** A setting shown in your code may have been ignored by the running service.
5. **A speed improvement only counts if the answer remains good enough for the job.** Faster wrong answers are not progress.

Everything that follows is the real project story behind those five rules.

## What we were trying to build

Imagine ten helpers preparing one company report.

One helper finds company facts. Another reads announcements. Another checks insider buying and selling. Others look at value, risk, market mood, forecasts, and the original investment idea. The final helper reads everybody else's work and recommends what to do.

These helpers are not ten separate brains. They are ten jobs given, one after another, to the same small local AI model.

**CrewAI** is the software framework that organises those jobs, gives each helper its instructions, and passes the work along. **Ollama** is the separate program that runs the AI model on the Mac. CrewAI coordinates the work; Ollama supplies the model's answers.

The project began with Qwen3.5-4B running through Ollama on an 8 GB M1 Mac mini. It later moved to Qwen3.6 on a 32 GB M6. Both run locally rather than sending the investment data to a paid cloud model.

The final helper often receives a large pile of notes from the other helpers. That pile is called the **context**.

This is important for a beginner because the model is only one part of the product. Our result also depends on:

- which company documents we collected;
- what ordinary Python code extracted from them;
- which tools worked or failed;
- how much material we put in the prompt;
- which settings the runtime truly used; and
- whether our checker looked for correct meaning or only tidy formatting.

When an AI application gives a bad answer, “the model is bad” is only one possible cause.

## Surprise number one: the AI was writing lots of invisible scratch work

Imagine asking somebody to solve a maths problem.

They hand you a one-line answer, but their desk contains six pages of calculations. You only see the final line, yet all six pages took time to write.

The AI was doing something similar.

For one important final-decision job, the computer reported an average of 8,878 generated tokens. A token is roughly a small piece of a word. But the visible answer contained only 4,305 characters.

Across 321 normal text responses, a rough characters-to-tokens calculation estimated that about **67.7% of generated tokens were not visible in the final answer**. A separate direct probe showed internal reasoning was an important cause; the 67.7% calculation did not count every hidden thought token directly.

This explained why telling the AI to make the final report shorter was not enough. Shortening the answer might remove one page while leaving most of the hidden notebook untouched.

### What this means for your project

Do not assume the visible answer is all the work the model performed. If your runtime reports input and output token counts, save them. If the token count is enormous but the visible answer is short, find out whether hidden reasoning, retries, or tool loops are consuming the time before rewriting the final answer format.

## The tempting button: “stop thinking”

The software gave us a switch that could turn the hidden reasoning off.

That sounded wonderful. A note made during one small test records:

| Setting | Work produced | Time |
|---|---:|---:|
| Thinking on | More than 1,600 hidden/output tokens | About 100 seconds or more |
| Thinking off | About 60 tokens | About 4 seconds |

That is a huge speed difference.

But speed was not the real test.

We asked a question based on a mistake the production system had already made. QFIN had paid several dividends, and the AI needed to compare the matching year-on-year payments.

The correct comparison was approximately:

> 0.76 down to 0.46, or about a 39.5% fall.

The tempting but wrong comparison was:

> 0.78 down to 0.46, or about a 41% fall.

The recorded thinking-on answers chose the correct matching periods.

The recorded thinking-off answers both failed. One reproduced the wrong 41% comparison seen in the real report. The original responses were not saved, so this is a documented case, not a test an outside reader can independently rescore from raw output.

Leaving reasoning on was not a complete cure. A later check of 359 saved report versions found three QFIN reports making a wrong dividend comparison despite having the matching-year numbers and reasoning switched on. One even called a fall from 0.78 to 0.46 a 57% reduction. The 15 October report used the correct roughly 39% figure. This is why an ordinary code check of which payments were compared and whether the arithmetic works is more durable than relying on the thinking setting alone.

So the “fast” button was like telling someone:

> Do not show your working. In fact, do not do the working. Just answer quickly.

That may be fine for an easy question. It was not safe for this financial comparison.

**Decision:** do not turn reasoning off for the whole system.

A later test added an important qualification. On a much denser KPG.NZ decision prompt, forcing reasoning on made the model fill all 8,192 allowed output tokens with invisible checking and never show an answer. Turning reasoning off made both the full and shortened versions finish and agree on the same decision, although they still omitted required fields. The lesson is therefore not “reasoning always helps” or “reasoning always hurts.” It is that the switch changes behaviour, and each real job needs its own quality test.

### What this means for your project

A setting that makes the model dramatically faster can also change how it answers. Test it on a real problem that your system previously got wrong. If the fast version fails that known case, it is not an optimization yet; it is a different and less capable behaviour.

## Surprise number two: a bigger pile of notes made every new word slower

Imagine writing a report while sitting beside a stack of paper.

With ten pages, it is easy to glance back and find a fact.

With hundreds of pages, every time you write a new sentence you may need to scan a much larger pile. The first reading takes longer, but so does every later lookup.

An AI behaves in a related way. It first reads the prompt. This is sometimes called **prefill**. Then it generates the answer one token at a time. This is called **decode**.

Our early measurements accidentally mixed those two jobs together. We fixed the experiment so the computer reported them separately.

Under the production Q8_0 memory setting, the AI generated roughly:

| Starting pile of notes | Writing speed |
|---:|---:|
| About 150 tokens | About 16.8 tokens each second |
| About 7,250 tokens | About 11.3 tokens each second |
| About 14,450 tokens | About 8.6 tokens each second |
| About 21,770 tokens | About 6.9 tokens each second |

At the largest tested pile, it was writing at only about 41% of its short-prompt speed.

We repeated the test four times and got a similar curve.

## The bookmark trap

Our first test had another mistake.

Each larger prompt began with the exact same text as the smaller prompt. The software remembered that shared beginning, like leaving bookmarks in a textbook. It reused earlier work and made some large prompts appear strangely fast to read.

We added a unique marker to every prompt so the software could not quietly reuse the old beginning.

The strange result disappeared.

The lesson is:

> If you do not control the bookmarks, you may think you are measuring reading speed when you are really measuring remembered reading.

### What this means for your project

Repeat important tests, change their order, and make each test input unique enough that the software cannot quietly reuse a previous result. A surprising speed improvement may be a cache, a retry disappearing, or a different input—not the setting you meant to test.

## The answer also fills the desk

The AI's memory desk must hold both:

- the notes it was given; and
- the answer and scratch work it has generated so far.

We tested this by asking it to generate 6,000 tokens from three different starting pile sizes.

As the answer grew, the AI slowed down in almost the same way as it did when we made the original prompt larger.

In simple terms:

> A page already supplied and a page the AI just wrote both make the next page more expensive.

This matters because the system's hidden reasoning can be thousands of tokens long. Looking only at the starting prompt does not tell us whether the whole job fits on the desk.

## Would a smaller pile save time?

We used the measured speed curve to estimate what would happen if the final decision helper received about 7,200 tokens instead of 14,400.

The calculation suggested a saving of about **24% for that one job** under the M1/Q8_0 setup.

That was useful as a prediction, but it was not a finished result.

We later ran the same saved job with full and carefully shortened evidence. The shorter input was 26.5% smaller, but the completed direct-answer comparison was only 6.3% faster. It also scored 17/20, or 85%, below our required 90%, and omitted a required thesis-status field.

So 24% means:

> Our measured curve says this is a reasonable estimate.

It never meant:

> We have proved that the real report is 24% faster and equally good.

The direct test therefore rejected this particular shortcut. We kept the 24% calculation in the record because it explains why the idea was worth testing, but we did not change the acceptance rules to make the disappointing result pass.

### What this means for your project

Do not shorten a prompt by deleting whatever looks long. First list the facts the answer truly needs. Keep those facts, their dates, units, and sources. Then compare the full and shorter versions using the same saved evidence.

## Surprise number three: the “small memory” cards were slow to use

The AI keeps special memory cards so it can look back at earlier words. These are called the **KV cache**.

On an 8 GB computer, space is precious. We used a compressed kind of card called Q8_0. Think of it as writing the notes in tiny shorthand:

- more cards fit in the box;
- but the computer may have extra work to do when reading the shorthand.

We then tested a larger, less-compressed card format called F16.

At short prompts, Q8_0 and F16 were close. As the prompt grew, F16 pulled further ahead:

| Prompt size | Q8_0 speed | F16 speed |
|---:|---:|---:|
| About 150 tokens | 16.57 tokens/s | 17.14 tokens/s |
| About 7,250 tokens | 11.21 tokens/s | 16.17 tokens/s |
| About 14,450 tokens | 8.64 tokens/s | 15.09 tokens/s |
| About 21,770 tokens | 7.09 tokens/s | 14.25 tokens/s |

At the largest point, F16 was about twice as fast as Q8_0.

We repeated the F16 arm with the running server's settings independently recorded. At the largest matched prompt it produced 14.09 tokens per second versus 14.25 the first time. Five of six points were within 2.5%; the remaining point differed by 6.3%. The speed pattern was therefore real on that M1 setup. But we did **not** switch the M1's normal runs from Q8_0 to F16: with only 8 GB of memory, the F16 setup needed a smaller context limit (35,840 rather than 64K tokens), which might cut off evidence in a long company report.

This changed our explanation.

We had correctly measured that longer contexts were slow under Q8_0. But we had been too ready to treat the steep slowdown as a general rule about long context.

The better explanation is:

- longer context still costs time;
- F16 also slows a little as context grows;
- Q8_0 made the slowdown much steeper on this particular M1, Ollama, and Apple's Metal graphics-computing setup.

The Q8_0 measurements were real. Our first explanation of why they looked that way was incomplete, and the later M6 result narrowed it again.

### The bigger M6 gave a different answer

We repeated the comparison on the new 32 GB M6 with both the small 4B model and the large 27B model. We first made sure every model stayed completely in the fast GPU part of memory.

| Model | Memory cards | Speed at the start | Speed around 21.8K prompt tokens |
|---|---|---:|---:|
| 4B | Q8_0 | 45.33 tokens/s | 40.12 tokens/s |
| 4B | F16 | 45.60 tokens/s | 38.95 tokens/s |
| 27B | Q8_0 | 8.98 tokens/s | 7.90 tokens/s |
| 27B | F16 | 9.05 tokens/s | 7.87 tokens/s |

This time F16 was not meaningfully faster. These endpoint numbers correct an earlier working table that called the prompt 24K and used different 4B rates; we checked the saved run records again. We should not say the old experiment was false: we repeated it and saw the same M1 result twice. The better conclusion is that the advantage belonged to that particular combination of computer, runtime and memory format. It was not a universal F16 rule.

**Decision for the tested M6 llama.cpp path:** use Q8_0 because it needs less memory and we detected no speed penalty there. Ollama was configured for Q8_0 during the later MLX tests, but we could not verify that MLX actually used it.

### What this means for your project

Model name and computer name are not enough to describe a test. Runtime version, context size, memory settings, how tightly the model's own numbers are stored, and whether the model stayed in fast memory can change the result. Record them even if you do not yet understand every field; somebody can help interpret them later.

## Why we did not change everything to F16

Because the larger memory cards need a larger box.

The original M1 Mac had only 8 GB of shared memory. We found an F16 setup that could hold 35,840 tokens without spilling work into a slower place.

But the 35,840-token desk has to hold the question, the supplied facts, the answer, and the hidden reasoning. Future answers do not come with a guaranteed size label.

Company filing vaults also grow over time. A job that fits this week may not fit after more documents arrive.

The software can sometimes keep going by moving old information out of the active window. That means a finished answer is not proof that the AI still remembered the first important fact.

So the smaller F16 setup is like a very fast desk that is sometimes too small. The current 64K/Q8_0 setup is slower, but it gives the general production jobs more room.

That was the M1 decision. The later M6 test reached the same final configuration choice for a different reason: F16 fit, but it was not faster.

That is our answer for our hardware and workload, not a configuration recipe for everybody else. Another model, runtime, computer, or task can produce a different result.

### What this means for your project

“It loaded” and “it finished” do not prove the full job fit safely. Leave room for the answer as well as the question. If the software can silently discard old context, add a test whose correct answer depends on information near the beginning.

## A real M1 company run gave us an encouraging clue, not a portable rule

We ran KPG.NZ through the real pipeline using the smaller F16 setup.

Across ten comparable job names, the middle speed improvement was about **1.74 times** the historical Q8_0 speed. That was very close to the roughly 1.75-times improvement predicted by the controlled test at production-sized contexts.

That was encouraging for the M1 investigation, but it was not a portable rule for the M6.

The new run and old runs did not receive identical inputs. The comparison also mixed reading time, writing time, and framework overhead. Both the memory-card format and the maximum desk size changed.

So we call it **corroboration**:

> The real-world result points in the same direction as the clean experiment.

We do not call it:

> A perfectly controlled proof that F16 alone caused a 1.74-times speedup everywhere.

## The M6 made the small model faster, but not the large model practical

The 4B, 9B and 27B models in this first comparison are called **dense** models. The “B” means billions of learned numbers, or weights. Roughly speaking, a dense model uses its whole set of weights to produce each new piece of text. A 27B model therefore has much more to move and calculate than a 4B model, even when both fit in fast memory. Later we tried a different design, called **Mixture-of-Experts (MoE)**: it keeps many specialist sets of weights but uses only some of them for each piece of text. The other sets still take up memory, even when they are not being used for that piece.

At the same 64K desk size and Q8_0 memory setting, the retained tests measured:

| Model | Speed on the M6 | Fast-memory placement |
|---|---:|---:|
| 4B | 43.8 tokens/s | 100% |
| 9B | 27.9 tokens/s | 100% |
| 27B | 9.1 tokens/s | 100% |

The old M1 4B result was 16.9 tokens/s, so the new machine made that model about 2.6 times faster at generating text. It also read the test prompt about 4.3 times faster. Those are whole-system comparisons rather than a perfect chip-only race because the operating system and Ollama version changed too.

The surprising part was the 27B model. It fitted completely in fast memory, yet the computer still had to move its much larger collection of learned numbers—the model's **weights**—to produce each new piece of text. Its roughly 18 GB size and the M6's memory bandwidth predict a ceiling close to the measured nine tokens per second. Giving it more GPU memory could prevent a slow spill, but could not make the memory bus carry the model several times faster.

We then ran one frozen company case through all three models:

- 4B took 27.8 minutes.
- 9B took 68.6 minutes.
- 27B took 168.6 minutes.

The 27B model did not buy a better checked result. It made the same net-debt mistake as 4B, while 9B got that calculation right. All three still reached the same final Hold decision. This was one deliberately difficult company and one run per model, so it was enough to reject an obviously poor speed-versus-quality trade but not enough to declare 9B the universal winner. The later three-company panel confirmed 9B as the stronger dense model before the MoE comparison changed the production choice again.

We also tried Ollama's MLX versions *at that point*. They were slightly slower for all three dense sizes in that test, including 8.8 rather than 9.1 tokens/s for 27B. That did not predict the later Ollama update: the newer MLX setup was faster for our tested production work once we controlled its growing memory use, as explained below.

We also learned not to read one memory number by itself. On the old M1, Ollama's displayed size suddenly jumped at exactly the point where it stopped keeping the whole model in the GPU and began splitting work with the CPU. The jump was a change in placement strategy, not proof that one small increase in context had suddenly created a huge pile of memory cards.

There are also two different kinds of compression in this story. Q8_0 versus F16 describes the working-memory cards. Q4_K_M describes how tightly the model's learned weights are stored. Changing model size while every model uses Q4_K_M does not tell us how much of a quality difference came from size and how much came from weight precision.

One clock also found something another clock missed. The application's internal timer recorded 168.6 minutes for 27B, but the experiment wrapper covered 428.5 minutes. We ruled out a duplicated AI task, a logged crash and the local document reader. We initially thought a power-log check had also ruled out sleep; it had not looked for the specific event we later learned mattered.

A later overnight run really did pause for about seven hours because the Mac went to sleep. The job was still safely running inside `screen`, but `screen` did not keep the computer awake after the SSH login ended. When Drew reconnected, network traffic woke the Mac and the job continued. The permanent sleep setting was then disabled for unattended operation.

An exact-window check of the older 27B experiment was recorded as finding repeated sleep-and-wake cycling. The filtered power-log output was not kept, so the exact event count and first-event time should not be treated as publicly checkable measurements.

That supports the conclusion that the outer 428.5-minute clock did not represent continuous AI work. We therefore use the application's 168.6-minute timer for the model comparison. We did not add up the exact minutes asleep, so the difference between the clocks is not fully accounted for.

The next idea was the **MoE** design described above. Imagine a large office with many specialists, but only a few work on each word. Qwen3.6-35B-A3B stores about 35 billion weights in total but activates roughly 3 billion for each piece of text. It still has to fit the whole office in memory; it need not put everyone to work on every piece.

It worked much better than the dense 27B model:

- 22 GB and completely in GPU memory;
- 72.3 tokens per second at the 64K screen;
- correct on the frozen KPG, SMI and IDX financial figures;
- clean tool calls; and
- about 68 minutes for all three companies, compared with about 274 minutes for dense 9B.

An NVFP4 version that outside evidence suggested might be faster reached only 49.0 tokens per second in that earlier screen, so we stopped that branch *then*. Later, a newer Ollama release and a memory-reset workaround made NVFP4/MLX faster in production-shaped tests. GPT-OSS 20B fitted and used the tools correctly, but turned $1.211 billion of net debt into $10.4 million of net cash. That was serious enough to stop without spending time on two more companies.

We then tested Qwen3.6 again at the real 80K production desk: 22 GB, completely in GPU memory and 70.6 tokens per second. We chose 80K because real jobs had previously approached 70K, not simply because the new computer had spare room. Production moved to `qwen3.6-35b-a3b-80k` only after that check.

## The same model name quietly became a different engine

When we updated Ollama and refreshed the model, its familiar name stayed the same, but the actual model files changed from Q4_K_M to NVFP4 and the engine changed from llama.cpp to MLX. We caught it because the before/after probes recorded the model format as well as its name. Our earlier finding that MLX was slower belonged to an older test; it was not a permanent rule.

The newer MLX engine started fast, but kept too much working memory from one company to the next. This is memory growth we observed **while Ollama was serving this model through MLX**; it is not a claim that the Mac's physical memory is faulty. It is not just our observation: Ollama's issue tracker has reports of [cache pressure on 32 GB Macs](https://github.com/ollama/ollama/issues/18131) and [extra memory retained after tool calls](https://github.com/ollama/ollama/issues/18620). A [fix for the latter issue](https://github.com/ollama/ollama/pull/18955) arrived after the Ollama 0.40.2 version used in these tests. The later 0.40.4 retest, described below, found lower memory use with restarts but did **not** make between-helper restarts unnecessary. We still cannot assign an exact share of the original slowdown to either upstream issue. On our 32 GB Mac, Ollama's reported memory rose from about 23 GiB to around 30-31 GiB by the second company, and writing speed fell. Restarting the model before each company takes about nine seconds; two earlier live three-company comparisons recorded roughly **10-15% faster generation** with restarts than the older Q4_K_M engine, but those live inputs changed. The completed frozen repeats below give a firmer test of the restart policy. MLX also behaved differently when prompts grew past the intended limit and did not repeat exactly at the same seed. Faster operation is not a clean replacement with no new costs.

The newer engine's reports were not clearly better in that earlier live comparison. Both versions used numbers that appeared in their inputs, but one MLX report mixed up a gold-price assumption with an AISC margin and had a valuation inconsistency. Each company was run only once per version, and one company received a new announcement between the two runs. We therefore kept the faster setup provisionally and ran the frozen-input repeats described below.

US companies exposed a second problem: one data-tool answer contained **471,073 characters**, making the model's question much bigger than its intended 80K-token limit. Capping each answer at 12,000 characters and explaining how to request the next piece brought the largest observed QFIN question down to about 33K tokens. QFIN then took 21.6 rather than 43.9 minutes; OGC took 31.1 rather than 45.1. Those before/after runs changed more than one thing, so we cannot credit the whole saving to that cap alone. Memory still grew inside a US company, so those jobs now restart the model between agents too. The restarts are explicitly temporary workarounds; the smaller, paged data answers and near-limit warning are lasting safeguards. QFIN's report now used the supplied valuation figures and calculated net cash in the right currency, but its buy/hold/sell-style recommendation still changed across runs. Better facts did not by themselves settle the decision.

An internal summary of the first whole-watchlist attempt with both kinds of restart records 220.8 minutes and a reported memory peak of 28.1 GiB, versus about 424 minutes for the preceding run. One company report was rejected because the AI omitted a required label, so this was **not** a complete set of published reports. Several things changed between those live runs. This is an operational observation that the combined setup ran without the earlier severe memory cliff, not a public call-level comparison, proof of how much time each change saved, or proof that the reports were equally good.

## A saved test needs more than saved company files

We found a way one test run could accidentally teach the next one: the second run could read the previous report that the first run had just written. We changed evaluation mode to read a saved, fixed previous report and isolated the other outputs and local inputs. US data needs its own frozen database for the planned comparison. Two versions may still choose different searches; that is part of their behaviour, so we will record what each looked for instead of forcing them to ask identical questions.

We also tested whether asking for the same random seed would give the same answer twice. On this Ollama version, Q4_K_M/llama.cpp did so exactly; NVFP4/MLX did not, through either API route tested. Setting temperature to zero made MLX only *nearly* repeatable and would change the normal production setup. The fair test therefore needs several runs per version and should compare how much each one's scores and recommendations move on the same facts.

## What the two overnight tests actually showed

We saved the company information, then ran each setup three times. The first test covered three New Zealand/Australian companies:

| Setup | Time for each three-company pass | Reports accepted |
|---|---|---:|
| Newer MLX engine, left loaded | About 71-76 minutes | 9 of 9 |
| Same MLX engine, restarted between companies | About 54-57 minutes | 9 of 9 |
| Older Q4_K_M build and different engine, restarted | About 60-61 minutes | 8 of 9 |

Restarting MLX also kept its reported working memory below 29 GiB. The older-build comparison changes both model files *and* engine, so it cannot identify which caused the difference. The [evidence table](evidence-and-provenance.md#e07--frozen-nzasx-and-us-batches-sanitised-extract) shows all three passes rather than just the ranges.

Claude, acting as an AI reviewer, scored these reports against the frozen rubric, assisted by an automated figure trace and checks against the saved source files. Drew did not independently re-score them. Claude found two wrong or mislabelled figures in the older-build arm and none confirmed in the two newer-build arms. Restarted MLX also had three keyword-detected completeness gaps versus none when left loaded; Claude read the flagged reports, but a keyword gap can sometimes reflect phrasing rather than absent analysis. Nine reports per setup, scored by one AI reviewer, cannot tell us that one version generally gives better answers. On the same saved facts, one company's risk score went 46, 69, then 58 across three runs. Another company's valuation was sometimes scored and sometimes treated as inapplicable.

The second test covered two US companies. Their data-search tools can make many calls inside *one* company, so merely restarting between companies did not reliably prevent memory growth:

| Restart policy | Three two-company pass times | Reports accepted | Peak working memory |
|---|---|---:|---|
| No restarts | 75.5, 74.6, 76.0 minutes | 6 of 6 | About 32-33 GiB |
| Between companies | 72.7, 81.6, 46.6 minutes | 5 of 6 | About 32-33 GiB |
| Between companies and agents | 51.1, 46.2, 46.4 minutes | 5 of 6 | Below 28 GiB in every pass |

All three setups used the same 12,000-character limit on each data-tool answer, so this test does not measure how much time that separate limit saved. The system could also choose different data-tool calls between passes. These are whole-run observations, not a guarantee that each future run will take the same time.

We also checked whether the US source stayed still. Its document tables did not change during the test, and all 41 data-tool questions repeated with exactly the same arguments returned identical answers. That does **not** mean each setup asked the same questions: only eight distinct calls were shared by all three arms. Those different information-seeking choices are part of what an agent system does and must be counted, not swept away. Five or six reports out of six were saved per arm, with a few factual or formatting problems and one unusual Sell recommendation where the other saved reports said Hold. Again, there is no trustworthy quality winner from this small sample.

The US test also caught a mistake **before** the AI answered. Python divided cash flow in Chinese yuan by dividends in US dollars without converting currencies, handing every setup an incorrect dividend-cover number. One report then misread even that wrong number. Giving all setups the same information makes the comparison controlled, but does not make the information true. The feeder was subsequently fixed: for cross-currency tickers it now uses trailing-year payments in the trading currency and converts cash flow using an FX rate, or says N/A when an input is missing. A later QFIN report showed 15.3x cover. The frozen tests retain the old wrong input because it is what those runs actually received. The fix also exposed a second lesson: Yahoo's per-share dividend fields did not identify currency and disagreed for QFIN, so simply converting the old number would not have made it trustworthy.

## The newer Ollama still needed the memory workaround

We updated Ollama from 0.40.2 to 0.40.4, which includes a fix for retained memory after tool calls, and repeated the two-company US comparison. New filings had arrived before we captured this test's company data. Its absolute times cannot fairly be compared with the previous overnight test; the two setups **within this new test** used the same saved data. Both kept the 12,000-character data-answer limit and restarted between companies. The planned difference was whether they also restarted between AI helpers.

| Restart policy on 0.40.4 | Three two-company pass times | Reports accepted | Peak reported working memory |
|---|---|---:|---|
| Between companies only | 62.2, 89.7, 49.1 minutes | 5 of 6 | 30.6-32.4 GiB |
| Between companies and helpers | 51.1, 50.4, 40.3 minutes | 6 of 6 | 24.0-26.2 GiB |

The second policy was quicker in all three passes and stayed well below the memory peaks of the first. Its peaks were also lower than those in the earlier 0.40.2 test, which is encouraging, but the changed company-data fixture prevents a precise claim about how much the software fix saved. We kept both kinds of restart. These are speed, memory and report-*acceptance* results, **not** a new assessment of whether the financial conclusions were right. [The per-pass measurements and source checks are in E09](evidence-and-provenance.md#e09--ollama-0404-us-unload-retest).

One report in the no-between-helper-restart setup leaked a thinking tag and process narration, and repeated a required facts block. The publication guard rejected it and retained the previous good report. This is the third observed case across the investigation and the first on 0.40.4. Because an earlier case happened without comparable memory pressure, we cannot call the leak a memory-only fault. It remains an open investigation, separate from whether the memory workaround improves speed.

## Giving the AI more room does not guarantee it sees the right page

Our vault used to cut long company documents down to 10,000 characters. An internal audit summary says 28 of 50 selected documents exceeded that limit; the cut versions kept a median of only 19% of their text. The per-document list was not saved, so those totals are not yet publicly reproducible. Nor does 19% of text retained tell us how much *useful* evidence survived. One important KPG gearing figure really was missing from its excerpt. Adding a few search words helped some topics while crowding out others.

Raising the cap to 20,000 characters let that KPG covenant information reach the model. Three live company runs were somewhat slower and used more memory, but did not fall off the earlier severe MLX speed cliff; KPG's new report correctly used the newly available covenant limit. The lesson is not "always send twice as much." It is to measure which facts a company question needs, select the relevant document sections, and test that the answer improves without swamping the model.

We then tested a smarter section picker. The existing method looks for keywords; the new one ranks document sections separately for each AI helper. In an offline check, the new picker lost fewer probe terms (47 instead of 60) while sending roughly 10-17% less text. It could include KPG's 33.3% *pro forma* gearing paragraph within 10,000 characters; the keyword picker needed 20,000. That showed better coverage of this particular passage, not yet a better report.

So we froze the same three companies and ran each picker three times with the same 20,000-character allowance:

| Document picker | Reports accepted | Checked wrong or mislabelled facts | Three-company pass times |
|---|---:|---:|---|
| Existing keyword picker | 9 of 9 | 0 | 58.2, 57.3, 53.6 minutes |
| Per-helper section picker | 9 of 9 | 2 minor label errors | 58.7, 54.9, 53.5 minutes |

Neither picker led any KPG report to use the 33.3% pro forma figure. All six KPG reports used the current 37.4% gearing figure already supplied by Python. Claude scored the reports as an AI reviewer; Drew did not independently re-score them. With only nine reports per method, this does not prove the section picker is worse, but it did not demonstrate a report-quality gain. It remains off. A smaller excerpt may still save memory, which would be a different test. One plausible reason the coverage gain did not show up in the checked answers is that code already supplied some decisive figures; this comparison does not isolate that explanation. [The measurements and limitations are in the evidence appendix](evidence-and-provenance.md#e08--frozen-vault-section-retrieval-ab-extract).

## A number can look right while saying the opposite thing

The last report from the old M1/4B setup said KPG's AFFO had fallen 7.7%. The first new M6/Qwen3.6 report said it had grown 7.9% to $100.2 million.

KPG's own results announcement said AFFO **increased 8% to $100.2 million**. The new report matched the company. The old report had retained almost the right percentage and the right dollar figure but reversed the direction.

This matters because a checker that looks only for nearby numbers could call both answers similar. They are not. “Up” versus “down,” “cash” versus “debt,” and positive versus negative can reverse the investment meaning while leaving familiar-looking digits behind.

One example does not prove the new model will always be right. We preserved the final old M1 reports so that every checkable new figure can be compared three ways: new report, old report and primary source.

## A fact can be on the page and still be too well hidden

The next checks found a problem on the new model's side. For CMO.NZ, the prompt contained the correct net debt of $144.6 million, but it was buried inside the working for a ratio as a raw number:

```text
Net Debt/EBITDA: 3.31x (Net Debt: 144576000, EBITDA: 43703000)
```

Nearby facts such as total cash and total debt had clear labels, commas and currency. The new model used a nearby 47.7% debt-to-equity ratio instead and said the dollar amount had not been provided. NZX.NZ then produced the same pattern.

The model was wrong to say the data was absent. It had received the number. But our prompt also made that number much harder to recognise and cite than the facts around it. The old model happened to notice this presentation; the new one missed it twice.

Think of a form where one important answer is printed clearly in its own box while another is written in tiny pencil inside somebody else's calculation. Both answers are physically on the page, but they are not equally usable.

The lesson is:

> “The text reached the model” is necessary, but it is not enough.

Important financial facts should be calculated by ordinary code and displayed once in a consistent form: clear label, currency, commas, date and source. Tests should inspect the actual rendered prompt and should be rerun when the production model changes. This is not wording designed specially for Qwen3.6; it is a cleaner interface for any present or future model.

The next full batch showed why this needs more than one carefully chosen example. GNE.NZ made the same “not provided” mistake using a third wording. NTL.NZ made a different mistake: it called NZ$1,178,381 of cash “net cash” even while mentioning NZ$93,133 of liabilities beside it. Net cash requires subtracting the liability, giving NZ$1,085,248. The number was present, but its financial meaning was wrong.

Two other figures that first looked suspicious were actually correct. SMI.NZ and RGI.NZ had used clearly dated company filings, while the newer yfinance numbers did not carry the same visible date. That is a useful warning: a different number is not automatically a bad number. Check what it measures, when it was measured, and which source it came from.

## A long-running stopwatch can reveal what a short race misses

The project had quietly saved the duration of every company run. There were 358 runs from the old M1 and 4B-model period, not merely the handful used during the computer change.

The average old run was 111.5 minutes per company. The first ten real runs using the M6 and Qwen3.6 averaged 21.4 minutes. In ordinary production use, the new system was therefore about **5.2 times faster** across those recorded samples.

The older system had also slowed over time: its weekly average rose from 78.4 minutes to 136.9 minutes over roughly seven weeks. Growing announcement files and longer context are a reasonable explanation, but the stopwatch cannot prove that by itself because the code, data and jobs also changed.

This does not mean “the M6 chip alone is 5.2 times faster.” We changed the computer, model, model architecture, software state and prompt workload. It means the complete new working system processed these companies about 5.2 times faster than the complete old system. That is still highly useful operational evidence, as long as we name the comparison honestly.

## Sometimes the best compression is a calculator

One KPG job once used 37,190 total tokens. That appeared to prove the smaller F16 desk could not fit the work.

Then we checked when the call happened.

It came before a code change that stopped asking the AI to read a complicated five-year REIT table. Python now reads the table deterministically and gives the model the exact facts it needs.

The next prompt for that job fell from 33,589 input tokens to 7,118.

This was not originally a speed trick. It was a correctness fix. The AI had been reading the wrong year's column.

But the correctness fix also removed roughly 26,000 tokens of clutter.

That taught us an important lesson:

> The best way to shorten an AI prompt may be to stop asking the AI to do work that ordinary code can do exactly.

Instead of handing a helper the whole phone book, use a reliable program to hand over the one checked address.

### What this means for your project

Use the AI for judgement, explanation, and ambiguous language. Use normal code for counting, date matching, unit conversion, table selection, and arithmetic whenever you can define the rule. This often makes the system faster and more accurate at the same time.

## A bigger brain cannot see a missing page

We also compared a Qwen report with work from a larger model, Claude.

This was not an attempt to replace Qwen with Claude. The question was simply:

> Can a larger or more capable model do better?

Claude did some things better. It also missed issues that a separate Codex review later noticed. During the exercise, another important problem became clear: in some cases, Qwen had not been given the information needed to answer the question.

Imagine asking two people:

> What colour was the car on page seven?

If neither person receives page seven, the more capable reader still cannot know. They may only make a more convincing guess.

That means every model failure must be split into smaller questions:

1. Did we provide the needed facts?
2. Did the tools retrieve them correctly?
3. Did ordinary code preserve the right dates, units, and values?
4. Did the model choose and combine the right facts?
5. Did the checker verify the meaning, or merely check that the answer had the right shape?

The larger-model comparison was also not a fair general race. It covered one ticker and one run. The runs happened on adjacent weekend days when the underlying information had not materially changed, but Claude could see the earlier Qwen report. Claude said it did not use that report, yet the test was still not blind.

It was useful for finding problems. It was not enough to declare one model the winner.

### What this means for your project

Before buying access to a bigger model, ask whether the current model received the necessary evidence. A larger model may help, but it cannot recover a missing page. Compare models on the same saved input and score specific errors rather than choosing the answer that sounds most polished.

## How we made the fair test

Imagine asking two cooks to make the same cake.

If one cook gets fresh ingredients and the other gets yesterday's leftovers, the result does not tell you which recipe is better.

Our live US-data tool can return a changing set of documents. That makes it unsuitable for comparing a full prompt with a compact prompt.

The fair test:

1. Take one exact, saved set of evidence.
2. Build both the full and compact prompts from that same evidence.
3. Prevent either test from fetching new information.
4. Write down the correct answers before looking at what the AI produces.
5. Planned to run both versions several times with reasoning turned on.
6. Score speed and answer quality separately.

The final compact-context fixture used a saved KPG.NZ decision task. Its answer sheet explicitly recorded:

- the ticker identity;
- carried risk, valuation, FOMO and thesis values;
- debt, occupancy, NTA, dividend yield and tracked cost;
- required output blocks and ordering; and
- claims the answer must not invent.

QFIN's date and dividend trap remained a separate reasoning-mode test. Mixing it into the compaction fixture would have changed both the evidence and the question at once.

Returning no data would be repeatable, but it would test a different question: how the model behaves when facts are missing. That is useful too, but it must be a separate test.

### The test found problems before it found an answer

The first checklist confirmed that twelve important facts survived shortening, but it missed that the requested final answer needed a sentiment score which no earlier helper had supplied. A broader audit checked every required output field and found two more problems: the final template forgot to provide a place for thesis status, and the shortening code had thrown away a complete one-line result saying a mining analysis was “Not applicable.”

After repairing those problems, reasoning-on still did not finish. Both the full and shorter versions used all 8,192 allowed output tokens as invisible scratch work and showed no answer. The full version repeatedly worried about whether a required marker counted as a forbidden heading. The shorter version chose its values, then repeatedly checked the form instead of submitting it.

We then ran one clearly labelled diagnostic with hidden reasoning turned off. Both versions finished in under three minutes and reached the same Watchlist/Watch decision with identical headline scores. That showed that forced reasoning—not a full 64K desk—was causing the runaway on this task.

It still did not pass the compaction test. The shorter version saved only 6.3% of wall time, scored 85% against a 90% requirement, and omitted required evidence. We stopped instead of running the planned five repeats.

## A tidy answer can still be wrong

The system has checkers that can confirm whether a report contains the expected headings and fields.

That is like checking that a school worksheet has every box filled in.

It does not prove the numbers inside the boxes are correct.

A useful quality check must separately ask:

- Did the report keep every required fact?
- Did it match the correct date and period?
- Are the units, signs, scales, and calculations right?
- Did it invent anything?
- Did one section contradict another?
- Did it admit when evidence was missing?
- Does the final recommendation agree with the analysis?
- Does it give a similar answer when repeated?

“The report parsed successfully” and “the report is financially sound” are different claims.

### What this means for your project

Write two kinds of checks:

- **shape checks:** required headings, valid JSON, fields present; and
- **meaning checks:** correct dates, values, units, calculations, sources, and final action.

Shape checks are easier to automate, but they should never be reported as proof that the analysis is correct.

We saw both sides of this in one October test. Three reports contained substantial analysis but were rejected because the checker demanded a `##` heading even though the writing instruction said a heading was optional. After the checker was corrected, a retry saved all three. That retry used headings anyway, so the specific no-heading fix is demonstrated by automated tests, not by the retry alone.

One saved report then passed the shape check while inventing `RUG.NZ` in a paragraph about Rua Gold (`RGI.NZ`). A warning spotted the wrong symbol but still allowed publication. This was **not** proof that it had confused Rua Gold with the separate company Rua Bioscience (`RUA.NZ`); its source and other company details were correct. The similar names remain a useful separate test. The report also called one valuation measure unusable even though that measure had been supplied. A complete-looking report can still make a wrong claim.

So we need three different questions: Did we fetch facts about the right company? Did the answer follow the promised format? Are its claims actually supported? Passing one check cannot answer the other two.

One later report omitted a risk-category label but included the risk score that determines it exactly. We did not make the format checker ignore the missing line. Ordinary code inserted that one label, marked it as calculated because the AI had omitted it, and then sent the report through the same strict checker. That is a useful distinction: safely filling in a fact we can calculate is not the same as letting the AI publish an incomplete or invented answer.

A later full run showed that even a strict rejection can have a second bug. Two reports were rejected, correctly keeping their bad versions from replacing earlier reports. But those companies then vanished from the dashboard and the portfolio overview, despite still being held. That downstream bug was fixed. “Reject the bad report” also has to mean “keep showing the last good one everywhere.”

Even an exact list prepared by Python could be changed when the AI retold it in the portfolio overview: it reversed one company's “hurt” into “helped,” dropped the only offset, and called an assessed company unassessed. We now display that list directly from Python. Giving the AI the right facts is not enough if we then let it rewrite facts that should be copied exactly.

We found the opposite problem in the checkers too: some warnings were false alarms caused by wording such as “while” or “non-existential,” or by a financial label that was valid for US filings. We replayed the revised checks over saved reports to confirm that the false alarms disappeared without losing known real warnings. The guard itself needs testing, not just the AI's answers.

## Mistakes we made while investigating

The investigation was useful partly because several attractive explanations turned out to be incomplete.

### We called a mixed speed number “writing speed”

It included both reading and writing. We replaced it with the computer's separate measurements.

### We briefly called the 24% estimate “measured”

The engine behavior was measured, but the real full-versus-compact saving was not yet measured at that point. We corrected the label, then ran the missing test. It delivered only 6.3% and failed the quality gate.

### We checked twelve facts and still missed an unanswerable question

Our first safety checklist confirmed that twelve important facts survived shortening. It did not notice that the final form demanded a sentiment number no earlier helper had supplied. We learned to check every required answer field, not only the facts we happened to list first.

### We let a failed answer run for two hours

The first native test had no output limit. The model generated more than 33,000 tokens without finishing, and the request timed out before it could save a result. We added a fixed ceiling and recorded whether the model stopped normally or merely hit the limit.

### We accidentally measured remembered prompts

Shared prompt beginnings activated the software's prefix cache. Unique markers fixed the test.

### We thought changing a setting meant the server used it

Some environment changes never reached the running Ollama server. We learned to verify the server's own startup record and actual memory footprint.

### We compared only the question with the desk size

The answer and hidden reasoning also occupy the context window. The whole job must fit.

### We used an old oversized call as if it described the current code

A later deterministic parsing fix had dramatically reduced that job. Historical measurements must be tied to the code version that produced them.

These corrections are not something to hide. They are the main recipe:

> Make a guess, design a test that can prove it wrong, keep the raw result, narrow the claim, and say what remains unknown.

## What we know, what we suspect, and what we still need to test

### We know

- The model produces a large amount of invisible reasoning in important jobs.
- Turning that reasoning off can make a small test dramatically faster.
- The contemporaneous QFIN probe summary records a wrong answer with reasoning off; its raw September response was not retained.
- On the dense KPG decision task, forcing reasoning on made both full and compact prompts exhaust 8,192 hidden tokens without answering; direct-answer mode finished but still missed required fields.
- Under Q8_0, longer total sequences made generation much slower.
- Generated tokens added the same kind of later cost as supplied prompt tokens.
- The independently repeated M1 F16 run stayed much faster than Q8_0 at larger contexts.
- The M6 did not reproduce that advantage; Q8_0 and F16 were effectively tied there.
- At 64K/Q8_0, the M6 measured 43.8, 27.9 and 9.1 tokens/s for 4B, 9B and 27B, all fully in GPU memory.
- The one-company screen found no checked quality advantage for 27B to justify its much longer runtime.
- The older Ollama MLX screen did not speed up any of the three dense models; later live NZ/ASX summaries recorded about 10-15% faster generation for NVFP4/MLX with per-ticker restarts. The subsequent three-pass frozen NZ/ASX panel measured the restart effect separately: 71-76 minutes per pass without it versus 54-57 with it.
- Dense 9B did better against the three frozen yfinance checks, but took about 274 minutes. One apparent 4B cash error was later found to be a real, older filing figure presented without its date and labelled as net cash; it was not a made-up number.
- Qwen3.6 MoE passed the same three financial checks and tool calls in about 68 minutes, and remained fully resident and fast at the production 80K context.
- GPT-OSS passed memory and tool compatibility but failed the financial gate by reversing net debt into net cash.
- The archived old report inverted KPG's AFFO direction even while preserving the approximate percentage and dollar amount; the new report matched the primary source.
- The old 4B model found buried net-debt values that Qwen3.6 missed on CMO.NZ and NZX.NZ, proving that models can react differently to the same uneven presentation.
- A later batch found a third wording of that failure on GNE.NZ and a separate NTL.NZ gross-cash-versus-net-cash mistake.
- Across the saved production log, 358 M1/4B runs averaged 111.5 minutes per ticker while the first ten M6/Qwen3.6 runs averaged 21.4 minutes, an observed whole-system improvement of about 5.2 times.
- The 27B run's outer clock included a long unexplained pause. Sleep/wake cycling is a plausible explanation from a contemporaneous check, but the filtered output was not retained and the time gap was not fully accounted for. A separate detached `screen` job demonstrably did not keep the M6 awake after SSH disconnected.
- The tested prompt compactor failed its predeclared quality and speed gates and is not going into production.
- An internal summary of the first full-watchlist attempt with the US data cap and both restart workarounds records 220.8 minutes without the earlier memory cliff, but one report was rejected; neither a complete published set nor each change's causal share is established by that run.
- Repeating the same prompt with the same seed was exact on the tested Q4_K_M runner but not on the tested NVFP4/MLX runner, so the planned comparison needs repeated runs.
- On the frozen US panel, per-company restarts alone were not consistently enough; restarting between agents kept the three two-company passes to 46-51 minutes and reported working memory under 28 GiB. Different setups made different data-tool calls, so these are end-to-end results rather than identical-answer tests.
- The frozen inputs also exposed large score swings and an input-currency bug, since fixed in the live feeder but deliberately retained in the frozen fixture. They do not establish a quality winner for either serving setup.
- An internal vault-audit summary recorded 28 of 50 selected documents cut under the old excerpt cap, without a saved per-document manifest; one important KPG passage was missing until the cap was raised and checked live.

### We have encouraging evidence, but not final proof

- The old KPG run's 1.74-times result supports the old M1 measurement, not a general F16 rule.
- Qwen3.6's early production evidence is encouraging, but the checked production reports have already exposed repeated prompt-salience and financial-attribution failures; there is still not enough evidence to estimate its general error rate.
- Compact, well-chosen evidence may still help other tasks, but our tested treatment did not justify deployment.

### Work that can continue after publication

- Compare more new M6 report figures with the archived M1 report and original company documents.
- Keep the fixed cross-currency dividend-cover calculation under regression test, including missing rates and currencies.
- Investigate score spread and the malformed-report cases using the saved prompts and source data. The completed frozen batches still cannot estimate a general accuracy rate.
- Test whether the measured section picker can use a smaller excerpt than the 20K-character baseline without losing answer quality. Its equal-budget test improved a coverage proxy, but not the checked reports.
- Keep checking the now-explicit classification and recommendation rules, including Sell for an owned position, against real reports.
- If another application timer disagrees with the outer clock, check sleep and pauses before calling the gap model time; sleep/wake remains a plausible but unclosed explanation for the older 27B gap.

## Why the M6 needed three races

Suppose we replace a small runner's shoes and also move them onto a better running track.

If they go faster, which change helped?

To answer that, we designed three comparisons:

1. M1 with Q8_0 against M6 with Q8_0: the computer change.
2. M6 with Q8_0 against M6 with F16: the memory-card change.
3. Today's real setup against the best safe M6 setup: the improvement the user actually feels.

Without the middle race, we would have mixed two causes together. The middle race produced the important correction: on the M6, F16 was not faster after all.

## The lesson in one story

Imagine a careful person doing a difficult worksheet at a small desk.

- They have a pile of source books.
- They write lots of scratch work you do not see.
- The more paper on the desk, the longer each lookup takes.
- Tiny shorthand cards saved space but were slow to read on the old setup.
- Bigger cards were faster there, but the new computer showed that this was not a universal rule.
- A larger brain can fit entirely on the desk and still be slow because every new word requires moving a much larger filing cabinet.
- Removing the scratch paper made one calculation fast but wrong; on another dense form, too much compulsory scratch work prevented any answer at all.
- Removing random source pages makes the desk cleaner but may remove the answer.
- A calculator can replace pages of error-prone arithmetic with one trusted result.
- A smarter person still cannot answer a question from a missing book.
- A neatly completed worksheet can still contain the wrong numbers.

So the goal is not “make the person write faster.”

The goal is:

> Give the person the right facts, enough room, useful tools, and time to think—then check that the answer is still right before celebrating the speed.

## If you are starting your own project tomorrow

You do not need our investment system, our model, or our hardware to use the method.

Start with this small loop:

1. **Choose one useful job.** Do not begin with “build an AI agent.” Begin with something you can recognize, such as “compare these two invoices” or “summarize this week's support problems.”
2. **Save one complete example.** Keep the exact input documents and the answer you believe is correct.
3. **Write down the traps.** Record the tempting wrong answer, missing-data case, wrong date, wrong unit, or other mistake you care about.
4. **Run the simplest version.** Save the prompt, output, duration, errors, and model/runtime settings.
5. **Check meaning by hand.** A fluent answer is not evidence of a correct answer.
6. **Change one thing.** Try a different model, shorter prompt, memory setting, or tool—but not all at once.
7. **Use the same saved example again.** Otherwise you cannot tell whether the change or the input caused the difference.
8. **Repeat it.** Models vary between runs. One good answer may be luck.
9. **Automate the checks you now understand.** Add ordinary code for exact facts and calculations before adding more model instructions.
10. **Keep the failed attempts.** They become your best future test cases.

You do not need a formal laboratory. A folder containing the saved input, expected facts, outputs, settings, and a short notes file is enough to begin. The important part is that next month's “improvement” can be tested against the same evidence as today's version.

## Questions to ask when something goes wrong

Work through these in order instead of immediately changing models:

1. Did the system receive the information needed to answer?
2. Did the tool or document reader return the right material?
3. Did ordinary code choose the correct date, row, unit, and value?
4. Did the prompt clearly ask the intended question?
5. Did the model have enough context space for both input and output?
6. Did the runtime actually use the setting we requested?
7. Did the model reason incorrectly despite having the right evidence?
8. Did the checker miss the error because the output looked tidy?

Only the seventh question is purely “the AI brain got it wrong.” In a real project, the other layers matter just as much.

## Our current answer

We have not found one magic switch.

We have found a safer way to improve the system:

- measure reading and writing separately;
- verify the computer used the settings we requested;
- use ordinary code for facts and calculations it can handle exactly;
- give the model compact, relevant evidence rather than raw clutter;
- compare treatments using identical saved inputs;
- choose reasoning mode from tests of the actual job rather than treating either setting as universally safe; and
- accept a speed improvement only after the financial answer still passes.

On the M6, that now means Q8_0 remains the memory-efficient choice on the tested llama.cpp path, dense 27B is eliminated, dense 9B remains the strongest tested dense baseline, and `qwen3.6-35b-a3b-80k` is the production model. The tag currently resolves to NVFP4/MLX, whose applied KV precision is unverified. The repeated NZ/Australian test found it much faster when restarted between companies; the US test found that restarting between agents mattered too. That serving-build choice is provisional on answer quality. The restarts are workarounds for the memory growth observed in Ollama's tested MLX path; we would remove them only after a later Ollama setup proves stable without them. The new tests do not give it a free pass on facts or decisions. They also found a Python currency mistake, since fixed in the live feeder, that illustrates why supplied facts need checks as well.

That is slower than changing one setting.

It is also much less likely to make a faster wrong-answer machine.

<!--
PAIRED REPORT MAINTENANCE CHECK
Before treating a change to this report as complete, review
from-tokens-per-second-to-decision-quality.md for the same:
- headline measurements and corrected measurements;
- measured/calculated/corroborative/planned status;
- production configuration and current decision;
- quality-gate outcome;
- limitations and open work; and
- model-comparison and missing-evidence framing.
Update both Last paired review dates together.
-->
