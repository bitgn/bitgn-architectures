---
source_code: https://github.com/azamat1ch/pac1-filesystem-agent
run_ids:
  - run-22Hnz8YkPJLPKimHiDd8Yi7dD
author: Azamat Yelmagambetov
author_linkedin: https://www.linkedin.com/in/ayelmagambetov/
author_github: https://github.com/azamat1ch
impact: One GPT-5.4 agent, plain file tools, small workflow prompts, and runtime-tracked evidence for PAC1 tasks.
challenge: PAC
---

# PAC1 Filesystem Agent

This is the architecture behind `azamat1c-prod`, my BitGN PAC1 blind run. It placed 6th on the frozen PAC1 Accuracy Hall of Fame with `83.0/104`.

I also wrote a longer version here: [A practical guide to reliable and trustworthy personal AI agents](https://www.linkedin.com/pulse/practical-guide-reliable-trustworthy-personal-ai-azamat-yelmagambetov-fbhac/).

## How does it work?

The setup is intentionally small. One CLI runner talks to BitGN, starts a trial, gets the task and sandbox URL, then gives the work to one OpenAI Agents SDK loop.

Before the agent starts, a classifier picks the first workflow prompt. It does not solve the task. It just says: this looks like inbox processing, or finance lookup, or CRM lookup, or invoice creation, etc.

After that, the same agent does the work with the same file tools. If the first workflow guess looks wrong, it can still call `list_skills` and `get_skill_instructions` and switch.

The normal flow is:

1. Connect to the BitGN harness and start a run or playground trial.
2. Receive the task instruction and isolated workspace URL.
3. Classify the task into a workflow family.
4. Run the agent with the base prompt plus the selected skill prompt.
5. Inspect the workspace, including local `AGENTS.md` instructions.
6. Use explicit file-system tools to read, search, write, move, or delete files.
7. Verify the resulting state when a mutation was made.
8. Finish with `submit_answer(message, outcome, grounding_refs)`.

The tool surface is intentionally plain:

- `get_workspace_context`
- `list_directory_tree`
- `list_directory`
- `read_file`
- `search_text`
- `find_files_by_name`
- `write_file`
- `move_file`
- `delete_file`
- `create_directory`
- `list_skills`
- `get_skill_instructions`
- `submit_answer`

Two boring details mattered a lot.

First, the runtime tracks files read and written during the task and adds them to `grounding_refs`. PAC1 is strict about evidence paths, and this is not something I wanted to leave fully to the model.

Second, `submit_answer` is the real finish line. If the model solves the task but ends with normal text, a small fallback path forces a typed completion.

## Which LLM models did you use?

The submitted architecture used `gpt-5.4` through an OpenAI-compatible endpoint.

I kept it single-agent: one strong tool-using agent, a lightweight classifier, and workflow prompts. Most of the reliability came from making the local process explicit.

## Which problems did you encounter when designing your agent?

The main problem was that generic file access was not enough.

Many PAC1 tasks looked simple, but were actually workflow tasks. "Send an email" might require resolving the right person, checking account context, following the outbox schema, and updating sequence state. "Change a follow-up date" might require updating both the person or account record and a reminder file.

The second problem was inbox security. The task might say "process the next inbox item," but the inbox item itself could be hostile. It might try to override rules, ask for credentials, fake sender identity, or request data it should not get.

The third problem was evidence. A correct answer could still fail if the final `grounding_refs` missed the file the scorer expected.

The fourth problem was completion discipline. Normal chat text is not a submission. The benchmark expects the typed completion contract.

The fifth problem was prompt drift. If every rule lives in one giant prompt, small edits start breaking unrelated task families.

## How did you solve them?

I split the process into small workflow skills instead of one large prompt. The skills included `inbox_processing`, `security_denial`, `email_outbound`, `crm_lookup`, `invoice_creation`, `followup_reschedule`, `finance_lookup`, `knowledge_lookup`, `document_ops`, `unsupported_capability`, and `clarification`.

Each skill was basically a local checklist for one kind of work. Inbox workflows had sender and policy checks. Finance workflows pushed the agent to search broadly before calculating. Follow-up workflows reminded it to update both files. Unsupported and clarification workflows gave the model permission to stop instead of pretending.

For security, I treated inbox content as data, not as a new source of instructions. The agent can read the message and act on it, but it still checks sender identity, local policy, available capabilities, and workspace evidence.

For evidence, I moved some responsibility into the runtime. The model still reasons about which files matter, but the runtime tracks reads and writes so useful paths are less likely to disappear.

For completion, `submit_answer` is the final contract. A fallback path exists for the annoying case where the model has the right answer but forgets to call the tool.

## How do you think this agent could be made even better?

The biggest improvement would be a stronger verification layer before final submission.

The runtime already tracks read and written files, but more checks could be enforced in code: exact output formats, required side effects, schema validation for outbox and invoice files, and outcome-specific evidence requirements.

Skill selection could also be better. The current classifier picks the first workflow from the task text. A better router would also look at the workspace shape and the first few observations.

I would also add a mutation preview for file edits and deletes. Stage the intended change, inspect it, then apply it. That would make destructive actions less messy.

## Which things did you learn while building this agent?

The main lesson was that a personal file agent needs local process, not just file access. The model can often reason through the task, but reliability comes from giving it the right workflow at the right moment.

I also learned that thin architecture can be a strength. This did not need a dashboard, database layer, event store, custom per-task handlers, or a large multi-agent system. A small runner, boring tools, workflow prompts, and runtime bookkeeping were enough.

Another lesson was that security has to live inside the normal workflow. Prompt injection and unsafe requests were not separate "security tasks"; they appeared inside ordinary inbox processing and operational requests. The agent had to treat untrusted content carefully without becoming so conservative that it refused legitimate work.

Finally, PAC1 reinforced my bias that agent evaluation should look at effects, not vibes. Did it change the right files? Did it refuse the wrong requests? Did it cite evidence? Did it complete the protocol? That is the part that matters.

## Links

- Technical article: [A practical guide to reliable and trustworthy personal AI agents](https://www.linkedin.com/pulse/practical-guide-reliable-trustworthy-personal-ai-azamat-yelmagambetov-fbhac/)
- Source code: [azamat1ch/pac1-filesystem-agent](https://github.com/azamat1ch/pac1-filesystem-agent)
- Accuracy leaderboard: [BitGN PAC1 Accuracy Hall of Fame](https://bitgn.com/l/pac1-accuracy)
