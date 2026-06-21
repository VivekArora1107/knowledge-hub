# 🧠 Knowledge Hub — an agentic layer over Notion

A personal knowledge system that fights note rot. It turns a pile of write-only notes into a daily, focus-aware revision habit — using a scheduled AI agent on top of Notion, with **no custom app, no frontend, and no separate database.**

> **Status:** v1 working. Daily agent live; note migration in progress.

---

## The problem

I learn something almost every day and take a lot of notes — but I rarely revisit them, so they decay into write-only storage. I wanted two things:

1. **Spaced revision** — old notes resurface automatically so they don't rot.
2. **Focus-aware retrieval** — I write a plain-language statement of what I'm working on (e.g. *"prepping for PM interviews, emphasis on metrics + stakeholder management"*) and the system pulls the most relevant notes for a quick daily review.

## What it does

Every morning, a scheduled agent:

1. **Interprets my focus** — reads a free-text "Current Focus" line I can edit anytime.
2. **Ranks notes** — scores every note against that focus, blended with a **spacing boost** so nothing goes un-revisited indefinitely.
3. **Synthesizes a digest** — for the top notes, writes a 2–3 line refresher and *why it's relevant today* (not just a link).
4. **Generates an active-recall prompt** — one sharp question per note, because retrieval practice beats passive re-reading.
5. **Stamps each surfaced note** with the date, so spacing works over time.

The output lands as a dated page in Notion each morning.

## Architecture

A deliberately **thin layer**, not a custom app:

- **Store:** Notes live in Notion (a single `Notes` database + section pages). No custom storage, no vector DB in v1 — at hundreds of notes the LLM ranks titles and snippets directly.
- **Intelligence:** A scheduled agent reads, ranks, and writes back via the Notion API.
- **Single-user, personal:** no auth, no multi-tenant.

```
Current Focus (editable)        Notes DB (rows: title, section, dates,
        │                         Last surfaced, Times surfaced)
        ▼                                   │
   ┌─────────────────── daily agent ────────────────────┐
   │  interpret focus → rank (relevance + spacing) →     │
   │  refresher + "why today" + recall question →        │
   │  write digest page → stamp Last surfaced            │
   └─────────────────────────────────────────────────────┘
        ▼
  📬 Daily Digests → "Daily Digest — <date>"
```

## Design decisions (the interesting part)

**Source migration: OneNote → Notion.** Hundreds of notes with images and attachments. Path: `onenote-md-exporter` → markdown + Word + a full-notebook PDF → import to Notion.

**Image handling — the load-bearing constraint.** Notion's API *cannot* embed local images (it only takes public URLs), and even after import, Notion's hosted image URLs are time-signed and expire in ~1 hour — so per-note pages can't be rebuilt with images via the API. Resolution:
- Word files import into Notion **with** images (Notion's importer uploads them) → kept as a **visual archive** at the section level.
- Each note becomes a **row in the Notes DB** holding clean, searchable text, with a link to its section page for the original images.
- Most notes turned out to be *text-as-images* (tables, slides, bullet lists), so a transcription/vision pass recovers the content as searchable text where it matters.

**Granularity.** Word import produces section-level pages; the agent needs note-level units for spacing and retrieval — so the section pages are split into per-note DB rows.

**Spacing logic.** Never-surfaced notes get the largest boost; otherwise the boost grows with days since last surfaced, blended with focus relevance, with a cooldown so nothing repeats within a week unless highly relevant.

## The agent

The full scheduled prompt lives in [`agent-prompt.md`](./agent-prompt.md). It's self-contained — it runs fresh each morning with only the Notion connector and the IDs it needs.

## Roadmap

- Finish bulk migration of remaining notes (scripted transform once tooling allows).
- v1.1 vision pass: caption/transcribe image-heavy notes at ingestion so they rank on content.
- Tune focus-vs-spacing weighting from real usage.

## Stack

Notion (store + delivery) · scheduled LLM agent (interpret/rank/synthesize) · OneNote export tooling · no frontend, no custom backend.
