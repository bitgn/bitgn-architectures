# Instructions to submit architectures for BitGN ECOM1

<!-- AICODE-NOTE: Keep this frontmatter contract aligned with harness_hub/insights.NoteMetadata; ECOM1 uses challenge slug `ecom`, not the event slug `ECOM1`. -->

ECOM1 was the first BitGN Agentic E-commerce challenge. The blind PROD window ran on **May 30, 2026**, and the architecture writeups in this folder are meant to link participant systems to the frozen Hall of Fame leaderboards:

- [Accuracy at any costs](https://bitgn.com/l/ecom1-accuracy)
- [Speed](https://bitgn.com/l/ecom1-speed)
- [Ultimate](https://bitgn.com/l/ecom1-ultimate)

## Add your architecture

1. Write the architecture in English.
2. Add one markdown file in this folder named `ACCOUNTID_architecture-name.md`, for example `4jKiTS_checkout-ops-agent.md`.
3. Use your BitGN account token as `ACCOUNTID`. This is the short public account id used on the leaderboards and in the existing architecture filenames.
4. Start from [TEMPLATE.md](TEMPLATE.md) if you want a fill-in structure.
5. Put YAML frontmatter at the top of the file. `source_code`, `run_ids`, `impact`, and `challenge` are the important fields. Author fields are optional; you can stay anonymous.
6. If the same architecture produced multiple nominated or winning runs, list all relevant ECOM1 PROD run ids in one file. If you used meaningfully different architectures, create separate files.
7. Open a PR with your new file, then ping Rinat.

Use `challenge: ecom` for ECOM1 entries. The benchmark id behind these runs is `bitgn/ecom1-prod`.

```md
---
source_code: https://github.com/example/ecom-agent
run_ids:
  - run-22ExampleAccuracyRun
  - run-22ExampleSpeedRun
author: Example Author
author_linkedin: https://www.linkedin.com/in/example/
author_github: https://github.com/example
company: Example Company
impact: Combines a commerce policy reader, deterministic state lookup, and a single tool-using agent for checkout, payment, return, and support tasks.
challenge: ecom
---

# Checkout Ops Agent for ECOM1

Short intro: what this architecture did, which ECOM1 run or leaderboard it belongs to, and why it is interesting.
```

You can find run ids in your BitGN run history or in the run URLs you submitted during the challenge. Only `bitgn/ecom1-prod` runs from the blind May 30 window are eligible for the frozen ECOM1 Hall of Fame boards. The Speed board additionally requires total thinking time up to 60 minutes.

## Description format

Please cover:

- How does the architecture work?
- Which LLM models did you use?
- How did the agent read and reason over the e-commerce OS: catalogue, inventory, customer records, payments, policies, tickets, returns, and audit trails?
- How did the agent decide when to act versus refuse or escalate?
- Which problems did you encounter when designing it?
- How did you solve those problems?
- What would you improve next?
- What did you learn from ECOM1?

## Images and diagrams

If you want to include images, put them in a `res` folder next to this README and link them from your markdown:

```md
![Architecture diagram](res/my-ecom1-agent-diagram.png)
```

Do not include private API keys, local secrets, or unpublished benchmark internals in the writeup. It is fine to describe your architecture, prompts, tools, evaluation loop, and lessons learned.
