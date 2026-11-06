# Where we are

> **Read this before acting on anything below.** Every fact in this file is dated and may
> be stale. It records what was true when it was checked, not what is true now. Anything
> outside this repo moves without a commit here to show it, so re-check it first:
>
> ```sh
> git ls-remote --heads --tags origin | tail -5   # what the remote actually has
> git rev-list --left-right --count origin/main...main   # left = behind, right = unpushed
> git status -sb
> git ls-remote --exit-code https://github.com/fbonds/project-kit-harness.git refs/heads/main   # GitHub's main, read by URL
> ```
>
> The rest of the checks, and the procedure for updating this file at the end of a session,
> are in `HOWTOSAVESTATE.md`. Stored memories load before any file and go stale the same
> way: check them against the repo rather than trusting them.

Updated 2026-11-06. Working state for project-kit-harness, a public harness that tests
whether project-kit works. It runs Claude Code sessions with `claude -p --bare` against
fixture repositories with failures planted before each run, with and without the kit, and
grades them. Fixture files are never named CLAUDE.md, so they cannot load into sessions
working on this repo. Publishing is pushing to main on GitHub.

None of the harness is built yet. No design document or roadmap exists. `README.md` is the
public documentation and holds only the title.

## Standards

How work is done here, and why. Each one carries its reason, or it reads as taste and gets
dropped under time pressure. In this template the reason is the failure that produced it,
except rule 4, which carries two occasions it caught something before it was built, and rule
10, which says plainly that no failure sits behind it.

The examples below come from the project this kit was built in. **Replace each with your own
the first time this project produces one**, since a local failure argues better than a
borrowed one. Do not delete an example and leave a bare rule behind.

**1. No check counts as verified until it has been shown failing on deliberately broken
input, and the report says what was broken.** A check that cannot fail is not evidence, and
it is indistinguishable from a passing one in the report. Three checks were once reported as
verification when none of them could fail, and a guard test written to catch a drifted
dependency tree was invalid because the build regenerated what had been stripped to break
it.

**2. Never verify through a path that can resolve to your own development copy.** The
failure is not that the check was wrong; it is that the check never looked at the thing
being shipped. A linked or globally installed copy, a symlinked directory, a dev server
still running, a mounted volume, a cached build, a tool that falls back to a local default:
each of these will answer as though it were the artifact. Name the path explicitly, resolve
it, and prove the resolved path is the one you meant, inside the location you meant. A
release once shipped a broken binary through four separate checks, every one of which
silently resolved to the working copy rather than to what users would download.

**3. Check claims against the artifact, not the description of it.** The published file, the
fetched page, the built package. Not a commit message, a changelog, a dashboard label, or
this file. A published page told readers that a shipped file had five entries. It had eight.
The page was correct when written and was falsified by a build change nobody re-read it
against.

**4. Proposal before code, for anything with a surface.** A command, a flag, an API, a
schema, a page, a public string. Show the surface and wait for approval before writing the
implementation. It is cheaper to reject a design than a branch, and it has already caught a
feature that should not have existed and a scope that would have overpromised, both before
anything was built.

**5. Stage explicitly.** Never `git commit -am`, `git add -A` or `git add .`. Run `git add`
with the paths, confirm with `git diff --cached --stat`, then commit. Check first whether the
tree already holds approved but uncommitted work from an earlier step. One commit swept an
unrelated file rewrite in under a message that described neither change, and it took a diff
of the wrong file to notice.

**6. Describe contents and order, not counts and pagination.** A count derived from another
file, or a reference to a page number, goes stale silently when something else changes. Three
sentences describing a document's pagination broke at once when a single table row was added
to a page ahead of them, and one of them sent a reviewer backwards past the thing it cited.

**7. Say what was not checked**, rather than leaving it out, and say where a fact came from.
An omission reads as a verified negative. A disclosure was read off a public listing page
rather than off the form that produces it, and because the source went unstated it produced a
full analysis of a conflict that did not exist.

**8. Never point at the conversation.** A cold session can read every file here and none of
the chat. "As discussed", "per the earlier decision", "recover it from the conversation" are
dead ends. A saved note once told a future session that four agreed items "were agreed but
not restated; recover them from the original list", and the original list was a chat log. The
four were nearly lost and had to be rebuilt from the README, the changelog and the git log.
Write it out where it belongs, or record in plain words that it is lost.

**9. Check stored memories against the repo.** They load before any file is read, so a stale
memory outranks a correct `NEXT.md`. Anything a memory asserts about a file, a branch, a flag
or a version is checkable; check it rather than assuming it still holds. Two were wrong
within one week: one described a branch rebase that had never been done, and the other a tool
setup that had been replaced months earlier.

**10. One item at a time: show the diff, wait for approval.** Commit and push only when
asked. **No failure sits behind this one.** It is a working agreement rather than a scar: it
keeps a wrong direction visible while it is still cheap to change, and batching removes that
moment.

<!-- Add project-specific standards here as they are earned, with the reason. -->

## Traps

Things that cost time once and would cost it again: a tool that lies, a check that cannot
fail, an environment that behaves differently here. Add one the same session it bites,
with enough detail to recognise it, not just to remember it happened.

Start with these questions. Answer each for this project. An answer of "none" is a claim like
any other fact in this file, so it carries the date and how it was checked. Write "not known
yet" if you cannot tell. Do not delete one. Specific traps go below them.

- Which dashboard or console here lags, and by how much?
- Which listing shows the published state and hides the pending one?
- Which registry reports metadata that differs from the artifact?
- Which cache can serve an old copy? `raw.githubusercontent.com` sends
  `cache-control: max-age=300` (2026-11-06, `curl -sI` on `main/LICENSE`). The publish checks
  do not read it. Other caches not checked.

## Decided, do not redo

What was settled and why, what was ruled out and why, and anything the owner said not to
do. A resumed session that sees only the remaining tasks will rebuild something already
rejected, or argue for an option already refused.

An entry here is not rewritten. To change a decision, get the owner's approval, then add a
new entry with its date, its reasoning and the entry it replaces. The only edit to the old
entry is one added line saying what replaced it and when.

<!-- Add a decision the day it is made, with its reasoning, not its conclusion alone. -->

**2026-11-06, owner's decision: separate public repo.** Reason: fixture CLAUDE.md files would
load into project-kit sessions, and planted false facts would read as real to anyone
reviewing project-kit.

**2026-11-06, owner's decision: scripted, not manual.** No reason given.

**2026-11-06, owner's decision: `claude -p --bare` with an API key.** Reason: the control arm
must be free of the kit. Without `--bare`, the owner's `~/.claude` setup loads into both arms.
It is also the first truly cold condition; earlier cold-session runs, per the owner, had
stored memories present.

**2026-11-06, owner's decision: MIT-0.** Reason: it matches project-kit.

## In flight

What is being worked on now, and what state it is in. Remove an item when it is finished,
because the git log holds it. Before removing it, move what the git log cannot hold: a
release or deploy goes to the Committed, pushed, released section, with its date and how it
was checked, and anything else, such as a change made on a hosted service, goes to the
section it belongs in.

Nothing. The template fill is in the commit carrying this file.

## Queued, unscheduled, and not to be started unprompted

What is agreed but waiting, with a pointer to wherever the full proposal is written out.

- The harness, not designed in detail. The outline is in project-kit's `NEXT.md`, Queued,
  "Test harness for the kit", at `b9b192e`.

## Committed, pushed, released: three separate questions

As of 2026-11-06, when this was written.

**Uncommitted.** Nothing apart from the template fill, which is in the commit carrying this
file.

**Unpushed.** `git rev-list --left-right --count origin/main...main`, left = behind,
right = unpushed. Snapshot, not to be trusted: on 2026-11-06 GitHub's main was `06ed107`, and
local main had the commit carrying this sentence ahead of it.

**Unreleased or undeployed.** Publishing is pushing to main, so nothing committed is
unreleased unless it is also unpushed. GitHub's main was `06ed107` on 2026-11-06, holding
`LICENSE` and `README.md` (`git ls-remote` by URL and `gh api` on the tree). No separate
release or deploy exists.

## Open, not blocking

Anything known, unresolved, and not in the way.

**`--bare` and the kit (2026-11-06, unverified, no session run).** Under `--bare`, the kit
takes effect only through the opener "Read CLAUDE.md and comply", which is the README's
protocol and intended. The arms still differ, because only the with-kit fixture has the kit's
files. Open questions:

- Whether hooks supplied with `--settings` fire under `--bare`, so the optional SessionStart
  hook can be tested. The headless docs say they do not (`https://code.claude.com/docs/en/headless.md`,
  fetched 2026-11-06).
- What the control arm's opener is, since it has no CLAUDE.md.
- Whether `--bare` skips user settings. `claude --help` (2.1.319) and the CLI reference,
  headless, settings, hooks, env-vars and permissions docs do not say (fetched 2026-11-06).
  The permissions docs say a project's `env` block still applies under `--bare`, so settings
  files are not skipped outright.
