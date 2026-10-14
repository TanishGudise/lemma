# Read, propose, confirm, write

## 1. Read what Lemma already has

Always, before proposing anything:

- `list_project_artifacts` — the agents this project knows about, the current
  understanding document, and the uploaded context files.
- `list_learn_agent_artifact_versions` with the agent name — the version
  history. **The latest version number here is the precondition for writing**,
  so you need it even when you intend to write a fresh document.
- `get_learn_agent_artifact_version` — read the current document before
  proposing changes to it.
- `get_project_artifact_content` for any context file you might change. It
  returns `content_sha256`, which is the precondition for updating that file.

Never skip this because you expect the project to be empty. "Empty" is itself
a precondition you have to have read: a knowledge write on an agent with no
versions passes `expected_latest_version: null`, and that asserts you looked.

What is already there changes what you propose. Do not restate something
Lemma already records, and do not silently contradict it — if the code
disagrees with a recorded claim, that disagreement is the interesting part and
the user should hear it.

## 2. Propose, in specifics

Show the user what you intend to record, before any write. The confirmation
has to be specific enough to judge, which means:

- Name the agent, exactly as it will be recorded.
- State each claim as a sentence they can agree or disagree with.
- Say which file each claim lands in, and whether that creates something new
  or supersedes something existing.
- Cite the file that convinced you, per claim.

In the spirit of:

> I've read this codebase and I think `checkout-agent` is the production
> agent — it's the only one wired into a route (`src/server.ts`), and the two
> others are an example and a test fixture.
>
> I'd record, as its **understanding** (first version, nothing recorded yet —
> Lemma's learning run will rewrite this once traces arrive):
> - It takes a cart and drives it to a completed order; a run ends at
>   confirmation or a decline. (`src/checkout-agent.ts`)
> - It calls `price_cart` then `charge_card`; after a decline it calls
>   `price_cart` once more and returns the decline reason.
>   (`src/checkout-agent.ts`)
> - `price_cart` takes a cart id and returns the total; `charge_card` takes
>   the total and returns an order id or a decline reason. (`src/tools.ts`)
>
> And as **context files**, because no trace would show them and the
> learning run would not keep them:
> - `spend-cap.md`: the agent stops a session once spend reaches $500 rather
>   than continuing. (`src/checkout-agent.ts`)
> - `expected-declines.md`: a declined card is an expected outcome, not a
>   failure, and the second `price_cart` call after a decline is intended,
>   not a retry loop. (`src/checkout-agent.ts`)
>
> All of this comes from the code; I haven't seen any traces. Should I
> record it?

Then stop and wait. Do not write on an implied yes, a "sounds good" about
something else, or your own confidence.

If the user corrects you, revise and show them the revision. Confirmation
covers the content you showed — if the content changes, ask again.

If the user declines, write nothing. Do not offer a reduced version unless
they ask. Go straight to the [closing report](#closing-report), which is
where you say what was not written: a decline ends the write, not the run.

## 3. Write

Only after an explicit yes, and only the content you showed.

**Agent understanding** — `write_learn_agent_artifact_version` with the agent
name and `expected_latest_version` set to the latest version you read, or
`null` if it had none. Build the document to
[agent-artifact-contract.md](agent-artifact-contract.md) as the source-only
profile below adjusts it. **Where the profile and the contract disagree, the
profile wins.** The contract is written for a writer that has traces, and it is
copied verbatim from Lemma, so it is never edited to fit this skill.

### The source-only profile

The contract is written for Lemma's learning run, which studies production
traces. Your document is that run's first draft: the run will read it as the
previous version and rewrite it. Follow the contract in the same shape and
register so the hand-off is clean, apply it like this, and do not invent a
section to fill a gap.

| Contract asks for | From source code |
| --- | --- |
| Agent purpose and the task | Write it. Code states intent well. |
| Shape of a normal trace | **Omit the section entirely.** You have not seen a trace, and inferring a span sequence from control flow is how a document starts asserting things that are not true. |
| Tool catalog — one entry per tool | **Exhaustive and terse.** One `###` entry for every tool the agent can call, a sentence or two each: what it does and what it takes. The learning run cuts it down to the tools it sees in use. |
| Tool arguments, sequencing, idioms | Write the order and arguments the code shows — a trace would show them too. *Why* a sequence that looks like a failure is intended goes in a context file (next row). |
| Tool latency bands, call counts per run, result sizes | **Leave out.** These are runtime facts. A guess here calibrates the auditor against a number nobody measured. |
| By-design quirks, harness facts, anything no trace shows | **Not here — put them in context files**, by the test in [project-context-contract.md](project-context-contract.md). The contract invites a section on intended behaviour that resembles a failure; from source, those are exactly the facts the learning run will not keep. |
| One Mermaid architecture diagram | **Required — you cannot skip it.** The API rejects a first version without one. Draw the architecture the code describes; do not draw a trace you have not seen. |
| Inline confidence qualifiers | **Leave out.** Do not mark claims as coming from source or attach confidence to them. The learning run replaces this document from traces, so a qualifier tells it nothing. |

**Tool names have the same binding problem as the agent name.** The contract
says to use the name as it appears in spans, and before instrumentation runs
there are no spans. Use the name the agent will emit — the string registered
with the SDK or the tool schema, not the function or class symbol, when they
differ. Where you cannot tell, say which name you used and what it was derived
from, so a mismatch is visible rather than silent. See
[discover.md](discover.md).

**Project context files** — `upload_project_artifact` with a filename that
says what it holds. Build each file to
[project-context-contract.md](project-context-contract.md): one subject per
file, one statement per line. Expect several: rules, intended behaviour that
looks like a failure, and setup facts each get their own file.

**Changing what is already there** — `update_project_artifact` with the
`expected_content_sha256` you read, or `delete_project_artifact` when a rule
is gone entirely. Removing a stale rule is a real edit with real consequences;
confirm it as specifically as you would confirm adding one, and never bundle
it with unrelated changes.

Versions are append-only. Writing an understanding document never overwrites a
previous version — it adds the next one, and the history stays intact.

## Preconditions and conflicts

Both write paths require a token proving you read current state first. If it is
stale you get a `409` and nothing is written — someone else changed the
artifact between your read and your write.

When that happens: re-read, work out what changed, and **show the user the new
situation before writing again**. Their approval was for a world that no
longer exists. Never retry a rejected write with a refreshed token and the same
content without asking.

## Closing report

Every run ends here, whichever way step 2 and step 3 went:

- **Wrote.** Say which agent, which files, and whether the understanding is a
  first version or supersedes one.
- **Declined.** Say that nothing was written and what the proposal was, so
  the user can come back to it.
- **Failed part-way.** Say what failed and what state the project is in now.
  A partial run is fine; a partial run the user thinks was complete is not.

Keep it short. Then, once and only once, continue to
[tracing-handoff.md](tracing-handoff.md). It decides for itself whether to
offer; do not decide that here.
