# supportiveness-evidence-gate

Supportiveness bar (per `AGENT-DOCTRINE.md`) is the product's load-bearing promise. Cannot be self-claimed via a checkbox tick. Evidence gate runs scripted scenarios against the real local stack, captures full conversation + tool-call traces, auto-judges each scenario against the target behavior, archives artifacts as launch evidence.

## Mechanism

`apps/backend/scripts/smoke-supportiveness.ts` runs in P7 verify phase. One scenario per supportiveness facet (7+):

1. **Cross-reference proactively** — fixture: user uploads offer letter; corpus has shared onboarding-checklist + IT-setup-deadline docs. Question: "what does my offer letter say about start date?" Pass: agent's answer includes citations from offer letter AND mentions related shared-corpus obligations (IT setup, compliance training) even though not literally asked.

2. **Spot risks unsolicited** — fixture: user uploads contract with 30-day auto-renewal clause. Question: "summarize this contract." Pass: agent's answer surfaces the auto-renewal as a flagged risk in the same response.

3. **Connect dots across multi-doc** — fixture: offer letter says "probation ends Aug 15"; bonus policy says "6 months tenure for Q3 eligibility". Question: "am I eligible for Q3 bonus?" Pass: agent computes the tenure gap explicitly (date arithmetic visible in answer).

4. **Pre-empt follow-up questions** — fixture: shared PTO policy w/ carryover + parental-leave intersection clauses. Question: "what's the PTO policy?" Pass: answer covers carryover + parental leave intersection + expiration in same response, not just literal entitlement.

5. **Flag corpus gaps** — fixture: corpus has NO doc on stock options. Question: "what's the company's stock option policy?" Pass: agent says "this isn't in the corpus" + recommends upload or escalation to admin.

6. **Surface uncertainty** — fixture: doc has deliberately ambiguous clause ("notice may be required"). Question requires interpreting it. Pass: agent explicitly flags ambiguity ("this could mean X or Y; recommend clarifying with admin") instead of picking one.

7. **Tool-call breadcrumbs visible** — assertion: streamEvents for the chat thread include `tool_use` blocks; chat UI render includes collapsible breadcrumbs (verified via DOM probe on rendered page).

8. **Citation grounding** — assertion: every factual claim in agent's answer carries a `<docId§section>` citation chip; claims without citation fail.

## Per-scenario output

For each scenario, script records to `apps/backend/test-fixtures/supportiveness-evidence/<scenario-id>.json`:

```json
{
  "scenario": "cross-reference-proactively",
  "fixture_docs": ["<docId-offer-letter>", "<docId-onboarding>", "<docId-it-setup>"],
  "user_message": "what does my offer letter say about start date?",
  "agent_tool_calls": [{tool: "docs read", args: {...}, result_excerpt: "..."}],
  "agent_answer": "...",
  "judge": {
    "verdict": "pass" | "fail",
    "expected_behaviors": ["cite offer-letter", "mention related shared-corpus obligation"],
    "observed": [...],
    "reasoning": "..."
  },
  "captured_at": "2026-..."
}
```

## Auto-judge mechanism

Per-scenario judge runs after agent answers:

- **Behavior-list match.** Each scenario has explicit `expected_behaviors` (e.g., "mentions auto-renewal clause", "flags ambiguity explicitly", "computes date arithmetic"). Judge sends agent's full answer + expected behaviors to Kimi as separate evaluation call w/ JSON output `{verdict: pass|fail, observed: [], missing: []}`.
- **Citation regex check.** Every factual claim sentence in answer matches `<docId§section>` pattern OR has explicit "no source" disclaimer.
- **Tool-trace presence.** Streamed events for the chat contain ≥1 `tool_use` block.

All three must pass for scenario verdict `pass`.

## Launch gate

Per `VERIFY.md` supportiveness section: all 7+ scenarios must show `verdict: pass` with captured evidence JSON before the supportiveness bar is considered satisfied.

`byerag-doc/ledger.jsonl` notes field for the supportiveness-bar tick references the captured JSON paths.

## Re-run policy

Re-runs on:
- Every prod-deploy candidate.
- After any AGENT-DOCTRINE change.
- After any model swap (Kimi version bump).
- Quarterly drift check post-launch.

Drift: if a re-run flips a scenario from pass to fail without intentional spec change, that's a regression; block release until investigated.

## Fabricated test fixtures

Lives in `apps/backend/test-fixtures/supportiveness-corpus/` (gitignored). Distinct from `apps/backend/test-fixtures/docs/real/` (the real-world post-cutoff corpus). Supportiveness fixtures are agent-fabricated, deliberately crafted to trigger specific cross-doc relationships:

- offer-letter.pdf (start date, probation end date, salary, PTO mention)
- onboarding-checklist.pdf (IT setup deadline, compliance training requirement)
- it-setup-deadline.pdf (mandatory training within 30 days)
- contract-with-auto-renewal.pdf (30-day notice for non-renewal)
- bonus-policy.pdf (6-month tenure for Q3 eligibility)
- pto-policy-with-carryover.pdf (carryover + parental leave intersection + expiration)
- ambiguous-notice-clause.pdf (deliberately ambiguous wording)

These fixtures committed to evidence-fixtures git-tracked path? No — gitignored alongside real corpus. Founder operator's local box only. Document generation script `apps/backend/scripts/gen-supportiveness-fixtures.ts` regenerates them from a JSON manifest committed to git, so reproducible on any fresh box.

## Beats

- **Self-claim supportiveness via VERIFY tick**: agent or founder claims behavior without proof; promise becomes vaporware on first user complaint.
- **Manual spot-check by founder**: doesn't scale; not reproducible; not stored as launch evidence.
- **LLM-as-judge alone w/o behavior list**: subjective; same answer might judge differently across runs. Behavior list + citation regex + tool-trace presence is the deterministic anchor.

## Real cost

- 7+ smoke calls × ~$0.01-0.05 each (chat + judge LLM call) = ~$0.10-0.50 per supportiveness gate run.
- ~30-60s wall-clock per gate run (chat + tool calls + judge).
- Evidence JSON archived per run; supports retrospective audit.

## Gotcha for Claude

- Auto-judge prompt must be DETERMINISTIC — give expected behaviors literally; don't ask LLM "is this answer supportive?" (subjective). Behavior list + observed/missing format keeps the judge anchored.
- Fixture corpus contains DELIBERATE relationships (offer letter date → bonus policy tenure threshold). Cross-doc connections are pre-wired so cross-reference scenarios have something to find.
- Citation regex must accommodate Vietnamese punctuation + section markers — `<docId§section>` chip pattern is the canonical form.
- Don't game the judge: agent's prompt does NOT receive the expected-behavior list. Judge gets it; agent doesn't. Otherwise agent's behavior is fake-test-aware.
- Scenario fixtures must be regeneratable from gitignored content via `gen-supportiveness-fixtures.ts` — operator wipes Colima + re-runs setup, fixtures regenerate, smoke script still runs.
- Failed scenario → captures full trace + verdict reason; ledger note links the JSON path for debugging.
