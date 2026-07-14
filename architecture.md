# Architecture

This document walks through the design decisions behind the BrightCart lead qualification bot, not just what each module does, but why the scenario is built this way.

## The problem this solves

Inbound leads from a contact form all look the same at a glance: a name, an email, a short message. Someone on the team has to read each one and decide who's worth chasing immediately versus who can wait. At low volume that's fine. At higher volume, it either eats someone's whole day or hot leads sit unread in an inbox next to leads that were never going to convert.

This scenario automates the triage step, not the sales conversation itself, that stays human. It just makes sure the right leads get human attention fast, and every lead (hot or not) is logged somewhere permanent.

## Flow

```
Webhook (form submission)
        │
        ▼
OpenAI — Generate a Completion
   (scores the lead 0-100, assigns Hot/Warm/Cold, drafts a next action)
        │
        ▼
JSON — Parse JSON
   (converts the AI's text response into structured fields)
        │
        ▼
Airtable — Create a Record
   (logs every lead, regardless of score)
        │
        ▼
     Router
     ┌──────────────┴──────────────┐
     ▼                              ▼
Hot Leads → Slack alert      (unbuilt — see below)
```

## Why each step is built this way

**Webhook as the trigger, not a scheduled poll.** Leads should get scored the moment they come in, not on a 15-minute delay. A webhook means the moment the form submits, the scenario starts.

**OpenAI does the judgment call, structured as plain text.** The prompt asks for a fixed three-line format (Score / Category / Next Action) rather than JSON directly from the model. In practice, asking a completion model to return raw JSON inline is more likely to break on edge cases (extra commentary, malformed brackets) than asking for a simple, predictable text format and parsing it downstream. That's a small design choice, but it's the difference between a scenario that silently fails on a weird input and one that doesn't.

**JSON Parse as its own step, not folded into the AI call.** Separating "get the AI's opinion" from "turn that opinion into usable fields" keeps each module doing one job. If the prompt format ever changes, only this step needs adjusting, the Airtable and Router modules downstream don't care how the text was produced, only that score/category/next_action exist by the time they run.

**Airtable logs everything, before the Router filters anything.** This ordering matters. If Airtable came after the Router's hot-lead filter, only hot leads would get logged, and every warm or cold lead would disappear the moment the scenario finished running. Logging first means nothing is lost even if the Router path never gets built out further.

**Router splits on category, not a manual filter guess.** Only leads scored "Hot" continue to Slack. This is a deliberate noise-reduction decision, a team that gets pinged for every single form submission will eventually start ignoring the channel entirely. Reserving the interruption for leads that are actually worth an immediate reply keeps the alert meaningful.

**The second Router path is intentionally left open.** In this demo it's unbuilt. In a production version, this is where Warm and Cold leads would typically route into a nurture sequence (an automated follow-up email, a CRM tag for later outreach) instead of just sitting in Airtable. Left visible and undocumented-as-finished on purpose, rather than quietly filled with a placeholder, because a real client engagement would scope that decision with the client rather than assume it.

## What I'd change for a real client deployment

This is a demo built around a fictional brand (BrightCart) to show the pattern, not a finished product. Adapting it for a real business would typically include:

- Replacing the fixed Airtable base with the client's actual CRM (or adding both, log-everything to Airtable/Sheets as a backup even if the primary record lives in a CRM)
- Building out the second Router path for Warm/Cold leads based on how the client actually wants to follow up
- Adding error handling for malformed webhook payloads (missing fields, empty messages)
- Tuning the OpenAI prompt against the client's actual lead volume and messaging style, a generic prompt scores differently than one calibrated against real examples
