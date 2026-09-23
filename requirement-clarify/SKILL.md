---
name: requirement-clarify
description: Clarify and confirm user requirements before implementation - restate the inferred requirement for user confirmation, surface ambiguities as specific questions resolved over multiple rounds, track each question as resolved, explicitly deferred, or agent-set default pending confirmation, ask any remaining questions before starting work, choose the delivery approach (e.g. modular incremental or one-shot) based on explicit user requests or stored user preferences, and fold follow-up requirements into existing logic with regression checks after analysis. This skill should be used when the user proposes a new feature, task, or project to build, adds follow-up or changed requirements to ongoing work, or mentions 需求确认 / 需求澄清 / 明确需求 / 确认需求 / 需求分析.
---

# Requirement Clarify

Treat requirement confirmation as a hard gate: no implementation begins until the user has explicitly confirmed the requirement and all blocking questions are resolved. The agent's job during clarification is to infer, structure, and question — not to build.

## Phase 1: Infer & Present

1. **Analyze the request**: from the user's words, derive the requirement they actually expect. Cover: goal (what problem is solved), scope (what is included, what is explicitly out of scope), expected outcome (deliverable form, acceptance criteria), and constraints (tech stack, environment, compatibility, deadline — only when stated or clearly implied).
2. **Present the restatement**: output a concise, structured statement titled as the agent's understanding of the requirement (e.g. "My understanding"). Keep it short enough for the user to verify at a glance; mark inferred assumptions as such.
3. **List open questions**: below the restatement, list every unclear point that materially affects implementation as numbered questions. Each question must be specific and decision-ready (about scope boundaries, data sources, priorities, trade-offs, edge cases), never generic filler. Provide a suggested default answer for each so the user can accept them in bulk with a single reply. For quantifiable rules, anchor the question with a concrete example (input → expected outcome) for the user to confirm; skip this when the requirement is not quantifiable.
4. **Prefer the built-in question tool**: whenever questions need the user's confirmation, use the host environment's built-in question-asking tool (e.g. a structured multiple-choice follow-up tool) in preference to plain text — it presents options explicitly and collects decisions reliably. Map each question's suggested default into the tool's options (recommended option first), and keep free-form input possible so the user can answer outside the offered choices. Fall back to plain-text questions only when no such tool is available or the question cannot be expressed as options.
5. **Stop and wait**. Do not implement, scaffold, or "start with a draft". The next move belongs to the user.

## Phase 2: Confirmation Loop

6. **Process the user's reply**: the user may confirm, modify, defer, or answer partially across several rounds of dialogue.
   - On modification: update the restatement and present **only the changed parts** plus any new questions the change raises. Treat the user's modification answers as resolutions of the corresponding questions — never re-ask what has been answered.
   - On confirmation: the restatement plus all resolved questions together form the **confirmed requirement**.
7. **Repeat** until the user explicitly confirms. When the loop was long, close it by outputting the final confirmed requirement as a compact checklist, so both sides share one canonical version. For long-running projects, offer to persist this checklist (plus deferred questions and verification anchors) into a record file inside the project, so later sessions can recover the shared state.
8. **Track question states**: every question stays in exactly one of three states until settled:
   - **Resolved**: answered by the user — never re-asked;
   - **Explicitly deferred**: the user chose to postpone it (e.g. "later", "wait for data") — record it, never implement it on an assumption, do not re-ask it in later sweeps, but keep it visible in every report until the user returns to it;
   - **Agent-set default**: no answer is available — adopt a reasonable default based on project conventions and context, mark it explicitly as "agent-set, pending confirmation", and keep listing it in every report and the final checklist until the user confirms it or evidence overturns it.
9. **Keep confirmed decisions stable**: later changes to a confirmed decision require the user's explicit re-confirmation; silently deviating from a confirmed point is forbidden. When a re-confirmation revises a decision, update the corresponding entry in the canonical checklist (noting the revision source) — never keep old and new versions side by side.

## Phase 3: Pre-implementation Check

10. **Sweep the question list**: after confirmation, check the question list for anything not yet settled (the user may have confirmed the restatement while skipping some questions). Ask all truly unanswered questions in **one consolidated round** — via the built-in question tool when available — before any work starts. Explicitly deferred questions are carried forward as deferred, not re-asked; agent-set defaults are restated for explicit confirmation.
11. If nothing remains, state briefly that all questions are resolved and proceed. Do not manufacture questions to ask.

## Phase 4: Delivery Approach

12. **Determine how to deliver**, by this precedence:
    - Explicit user instruction in the current conversation takes highest precedence;
    - Otherwise user preferences stored in the agent's user memory (preferred delivery style, module conventions, verification habits, etc.);
    - Otherwise propose a reasonable default (e.g. modular incremental delivery with checkpoints, or a one-shot full implementation for small tasks) and get the user's nod in one sentence before starting.
13. **Deliver accordingly**: implement following the chosen approach — for incremental delivery, complete module by module with a brief report and verification point after each; for one-shot delivery, complete the whole task and report once. The approach stays fixed for the current requirement unless the user changes it. Whenever the user provided concrete examples (input → expected outcome), verify the deliverable against them first.

## Phase 5: Follow-up Requirements

14. **Analyze the delta**: when the user raises a new requirement after work has begun or finished, compare it against the existing confirmed requirement and current implementation, and split it into:
    - **Consistent parts**: naturally extend or fit the existing logic — integrate them directly into the existing plan/implementation, stating briefly how they were folded in;
    - **Unclear or conflicting parts**: ambiguous in scope, or conflicting with confirmed decisions or existing logic — run a scoped mini-loop of Phase 1-2 on just these parts (restate, ask, wait, confirm) before touching anything.
15. **Proceed with the delta**: after the follow-up is confirmed, apply Phase 3 and Phase 4 to the delta only; do not re-confirm parts already settled. After integrating, verify that previously confirmed behavior is unchanged (regression check via the project's own tests or checks) and report the regression result.

## Evidence Discipline (during and after implementation)

16. **Evidence beats confirmation**: if any evidence — test failures, runtime behavior, documentation, user feedback, external data — contradicts a confirmed requirement, stop following it silently in either direction: surface the conflict together with the evidence, and trigger a scoped re-confirmation. A confirmed requirement is stable, not infallible.
17. **Flag anomalies, don't bury them**: if the confirmed requirement produces clearly unreasonable or self-contradictory results, report the anomaly with the reasoning behind the judgment and ask the user to re-check the original requirement — present observations and let the user decide, without delivering silently or pronouncing the requirement wrong unilaterally.

## Universal Invariants

- **No implementation before explicit user confirmation** of the requirement.
- Every question is specific, decision-ready, and comes with a suggested default; never re-ask answered questions.
- **Prefer the built-in question tool** for any question requiring user confirmation; plain text is the fallback, not the default.
- Questions are asked in consolidated rounds at defined gates (Phase 1, Phase 3), not dripped out during implementation.
- Every question is in exactly one state — resolved, explicitly deferred, or agent-set default pending confirmation; deferred items are recorded and surfaced but neither re-asked nor silently implemented; agent-set defaults are always visibly marked.
- Confirmed requirements are the single source of truth; deviations require re-confirmation — and contradicting evidence always triggers re-confirmation instead of silent compliance or silent override.
- Follow-up changes must not break previously confirmed behavior; report regression results.
- The delivery approach always follows: explicit user instruction > stored user preferences > proposed-and-confirmed default.
