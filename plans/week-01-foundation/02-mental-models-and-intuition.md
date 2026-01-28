# 02. Three Mental Models & Design Intuition

## Problem Overview
- Capture reusable thinking tools that guide rapid system design decisions before deep dives.
- Package models so new engineers can diagnose trade-offs under time pressure.

## Functional Requirements
- Document three canonical models (e.g., Backpressure Triangle, CAP switches, Hot-path slicing) with triggers for use.
- Provide facilitation script for 30-minute workshops that rehearse each model using historical incidents.
- Offer checklists that attach the models to the rest of the curriculum’s problems.

## Non-Functional Goals
- Materials must be portable (Markdown/PDF) and easily embedded into retros or design docs.
- Encourage debate: each model includes prompts, anti-patterns, and metrics to monitor.

## Architecture Overview
- Knowledge base stored in Git-backed docs; surfaced via internal handbook site.
- Companion exercises stored as issue templates that students can duplicate.
- Optional automation: Slack bot that serves a random model when `/design-check` command is invoked.

## Data & Templates
- Template fields: `Scenario`, `Tension`, `Indicators`, `Interventions`, `Escalation path`.
- Backpressure model uses queue depth + latency graphs; CAP model uses matrix heatmaps; Hot-path model overlays dependency graphs.

## Implementation Plan
1. Interview senior engineers to extract tacit heuristics and map them to incidents.
2. Draft canonical doc per model following the template and review with peers.
3. Build lightweight facilitation deck + worksheets for breakout sessions.
4. Automate reminders or Slack bot integration to keep models top-of-mind.
5. Measure adoption via retro templates referencing the models.

## Testing & Validation
- Pilot sessions with small cohorts, collect feedback surveys, and iterate wording/examples.
- Track whether new design docs cite at least one model (target >70%).

## Operational Considerations
- Keep models updated after major architecture shifts or on-call learnings.
- Archive outdated heuristics and annotate when assumptions change (e.g., new datastore replaces old CAP profile).
