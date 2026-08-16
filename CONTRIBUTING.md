# Contributing

Contributions welcome — in English or 中文.

## The bar

An entry must teach something about **building the scaffolding around a model**. That's the whole filter.

**In scope:** context management, tool and interface design, agent loops, memory systems, orchestration, sandboxing and permissions, prompt-injection defense, progressive disclosure, trajectory evaluation, and case studies of real harnesses.

**Out of scope:** model training and fine-tuning, prompt tricks for chat interfaces, product announcements, "top 10 AI tools" roundups, and frameworks with no accompanying explanation of *why* they're built that way.

A useful test: if the post would be equally true about a chatbot with no tools, it doesn't belong here.

## Format

One line per entry, in the section it fits:

```markdown
- [Title](https://example.com/post) — Author or org. One sentence on what you actually learn from it.
```

That sentence is required and it should be specific. "A great article on agents" tells a reader nothing. "Names four ways long contexts fail and gives a remedy for each" tells them whether to click.

## Submitting

1. Fork and branch.
2. Add your entry, keeping sections alphabetical only where they already are — most sections are ordered by usefulness, so place accordingly.
3. Check the link resolves.
4. Open a PR describing why the entry meets the bar.

Bilingual note: adding to only one README is fine. Say so in the PR and a maintainer will mirror it, or open a follow-up yourself.

## Longer notes

The `docs/` folder holds synthesized notes rather than link lists. If you want to write one — the agent loop, memory, multi-agent, permissions, and evaluation are all open — open an issue first so we can agree on scope. Follow the existing shape: a one-sentence thesis, the substance, honest open questions, and sources.

## Removals

PRs that remove dead links, superseded material, or entries that no longer meet the bar are as welcome as additions.
