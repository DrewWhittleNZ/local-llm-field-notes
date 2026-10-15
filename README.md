# Local LLM field notes

This is the story of trying to make a local AI system genuinely useful, not just impressive in a quick demonstration. It is also the story of learning how these systems actually behave by building one.

I work in cybersecurity, where part of my work now involves understanding and managing my organisation's exposure to AI-related threats. I wanted to know more about AI than how to use Claude and Codex. Building something at home gave me a way to learn through real problems, failures, and tests. I am not an AI researcher or a software developer by trade. But working with code on a personal project has been fascinating, and I do not plan to stop.

I chose investment analysis because I already had investments, so the questions mattered to me. Reading company documents, checking financial figures, and deciding what evidence is sufficient also made it complex enough to be an interesting test of AI. The goal was to use a local system to help read that information and produce reports—not to hand financial decisions to a model. Claude and Codex helped me build, test, challenge, and document the work. I chose what to investigate and whether a result was good enough.

The analyzer uses **CrewAI** to organise several specialist AI agents, each responsible for part of a company report; **Ollama** runs the local model they use. The project began on an older M1 Mac mini. I was not a Mac person, but I bought the later 32 GB M6 Mac mini specifically for this work. Learning the machine and its software became part of the project too.

I knew the questions I wanted the system to answer, but had to learn how much of each answer depends on the information supplied, the code around the model, and the checks applied afterward. This project has been one way of upskilling, not a claim to be an AI expert.

Apologies for being so wordy about the background! I felt the "why" mattered as much as the numbers.

At first the problem seemed to be speed. Reports took too long to produce on a small Mac running a local model. Testing that problem uncovered others: an apparently sensible speed setting could make a financial comparison wrong; a larger model could miss a fact that a smaller one found; a tidy-looking report could contain a reversed or mislabelled number; and sometimes the model simply had not been given the data needed to answer. Later, a software update changed the model files and serving engine behind an unchanged name. Each discovery pushed us to save evidence, repeat tests, and narrow our claims.

These notes are for people like me who are trying to build something with AI without first becoming specialists in AI infrastructure. They are also open to technical criticism. We want someone more experienced to be able to point to a weak comparison, a better explanation, or an experiment we missed.

## Start here

- [*From Faster Answers to Better Decisions*](from-faster-answers-to-better-decisions.md) is the plain-language account. Start here if you do not have an AI background.
- [*From Tokens per Second to Decision Quality*](from-tokens-per-second-to-decision-quality.md) is the technical field report, with methods and limitations for readers who want to challenge the work.
- [Evidence and provenance](evidence-and-provenance.md) contains sanitised numerical extracts and identifies what still rests on internal records.
- [Changes to the field notes](CHANGELOG.md) records substantive additions and corrections after the first public version.

The plain-language and technical accounts are meant to stay in sync. Neither is investment advice or a recommendation to buy or sell any security. The point is how to test an AI-assisted analytical system without mistaking a faster or more polished answer for a trustworthy one.

## Publication status

This is the first public version of an ongoing field report, not a finished paper. The underlying analyzer, portfolio data, full reports, prompts, logs, and working history are not included. The reports distinguish reproduced measurements from results supported only by internal summaries, and I expect to correct or extend them as the work continues.

I intend to release the analyzer's code when I am happy with it and have reviewed what can safely be shared. It is not ready for that yet, and this document repository is not the code release. There is no promised date.

The shortest version of the lesson so far: **measure what actually ran, check what information the model received, test the financial meaning of its answer, and keep the corrections visible.**
