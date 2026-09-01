# Euodia

**euodia.app** · *yoo-OH-dee-ah* · good road / prosperous way

A public product that takes a job seeker from **hunt → tailored resume → interview stories → offer decision / negotiate**. Facts in. No fabricated experience. Structure first, render second.

This repo is empty on purpose. The brief below is the contract.

---

## Product

| | |
|---|---|
| **Who** | Public from day one |
| **What** | Full pipeline (vision). First shipped slice is resume. |
| **Shape** | One repo: agent-agnostic **engine** + **Vercel / Next** app |
| **Hosts** | Interchangeable. Not a Hermes or Claude skill pack. Those can be clients later. |

**Engine does not import Next. The app imports the engine.** If the host disappeared, the engine still runs.

```
euodia/
  packages/engine/    # schemas, prompts, pipelines
  apps/web/           # Next.js + Vercel AI SDK
  adapters/           # optional later (Hermes / Claude)
```

## Rules

1. Never fabricate. Every title, bullet, and number comes from the user.
2. Ask only what changes the answer.
3. Structure first, render second (one schema, many skins).
4. Never apply on the user’s behalf.

## Inspiration

Patterns from [offer-toolkit-skill](https://github.com/yanliudesign/offer-toolkit-skill) (MIT). Copy a file only after it has been used and earned its keep. Retain upstream MIT notices on anything substantial taken from that work.

## Not this

- A Hermes profile, `AGENTS.md`, or lane-A job-search pipeline
- A rename of Offer Toolkit
- A stack demo whose value is Vercel or AI

## License

MIT © 2026 Layish Sieger
