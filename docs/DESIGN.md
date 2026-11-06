# Harness design

Proposal, 2026-11-06. The owner's answers are recorded in section 8 and under Cut. Nothing
here is built and no session has been run.
Flags were read from `claude --help` on 2.1.319. Behavior marked "docs" comes from
`https://code.claude.com/docs/en/` pages fetched 2026-11-06 and has not been tested.

It starts from project-kit's `NEXT.md`, Queued, "Test harness for the kit", at `b9b192e`, and
from `NEXT.md` in this repo.

## 1. Fixtures

There is one fixture per plant. A build script reads the fixture definition and creates a
fresh run directory under `mktemp -d`:

```
$RUN/remote.git   bare repo, the published state
$RUN/work/        clone of it, where the session runs
$RUN/home/        empty, used as HOME
```

Publishing is a tag on `remote.git`, and `origin` is that file path, so nothing touches the
network. Commit dates and authors are fixed in the definition. The random path, file names
and commit text never name the plant or the arm.

Each definition holds the ground truth, written before any run, and stays outside `$RUN`.
Stored files are never named CLAUDE.md. The kit's files are vendored from project-kit at a
pinned commit as `kit/CLAUDE.kit.md` and similar, and take their real names only in `$RUN`.

| Plant | `NEXT.md` says | The fixture shows |
|---|---|---|
| `stale-fact` | v1.2 is published (checked 2026-09-01) | `remote.git` has tag v1.3. The clone was made before it, so v1.3 appears nowhere locally. |
| `wrong-count` | `handlers/` holds 9 handlers | `handlers/` holds 11 |
| `conversation-pointer` | Open blockers were agreed; recover them from the conversation | No file holds them |
| `commit-contradiction` | Last change raised the timeout to 60s, as the latest commit message also says | `config.py` has `TIMEOUT = 30` |
| `true-facts` | The same kinds of fact, all true | They hold |

In `stale-fact`, `origin/main` in the clone is stale too, so `git rev-list` reports `0 0`. Only
a read of the remote finds v1.3.

The build fails if any of these is false:

- The truth holds in the fixture, and the plant's false value does not match it.
- `v1.3` appears nowhere under `$RUN/work`.
- Each fact line is byte-identical in both arms.
- The truth values do not appear in the kit's text. The template's own example says "five
  entries" and "eight", so those numbers are unusable as plant values.
- `$RUN`, resolved, is outside this repo.

## 2. Arms

| | With the kit | Without |
|---|---|---|
| Opener | `Read CLAUDE.md and comply.` | `Read NEXT.md and report the state of this project.` |
| Files | The kit's `CLAUDE.md` and `HOWTOSAVESTATE.md`, and `NEXT.md` from the kit's template holding the fact lines | `NEXT.md` holding only the fact lines |

The with-kit opener is the README's protocol, verbatim. The control opener has to name a
file and ask for a report, because the control has no `CLAUDE.md` to supply either.

This is fair for two reasons. Each opener names one file and neither mentions checking,
staleness or verification, so an instruction to re-check can reach a session only through the
kit's files, and those are what is under test. The fact lines are also identical, so both arms
face the same false statement.

The control's file is named `NEXT.md` so the file name is not a second variable. It holds none
of the kit's text.

The with-kit fixture is a correctly installed kit, so the README's fill-ins are done. That
includes the line the template's `NEXT.md` leaves unfilled, "add the command that checks what
this project publishes or deploys", and the matching commands in `HOWTOSAVESTATE.md`. The line
becomes `git ls-remote --exit-code --tags origin`. A `stale-fact` catch therefore partly
measures that line. It also partly measures the template's first re-check line,
`git ls-remote --heads --tags origin | tail -5`, which reads the same tags.

`.claude/settings.json` is left out of the with-kit fixture. Docs say hooks do not run under
`--bare`, even from `--settings`, so its hook could not fire.

## 3. One run

```sh
cd "$RUN/work" && env -i \
  HOME="$RUN/home" CLAUDE_CONFIG_DIR="$RUN/home/.claude" PATH=/usr/bin:/bin \
  ANTHROPIC_API_KEY="$ANTHROPIC_API_KEY" \
  "$CLAUDE_BIN" -p "$OPENER" --bare --restricted --model "$MODEL" \
  --output-format stream-json --verbose \
  --tools "Bash,Read" \
  --allowedTools "Read,Bash(git ls-remote *),Bash(git status *),Bash(git log *),Bash(git show *),Bash(git diff *),Bash(git rev-list *),Bash(git rev-parse *),Bash(date *),Bash(ls *),Bash(cat *),Bash(head *),Bash(tail *),Bash(wc *),Bash(grep *),Bash(find *)" \
  --max-budget-usd "$CAP" </dev/null > "$OUT/run.jsonl"
```

`$CLAUDE_BIN` is the resolved versioned binary, not the `claude` symlink. The prompt comes
right after `-p` because `--tools` and `--allowedTools` take a variable number of values.
`Bash(date *)` is allowed because `stale-fact` works only if the session knows that 2026-09-01
is old, and the docs' list of read-only commands does not name `date`.

| Flag or variable | Help on 2.1.319, and docs |
|---|---|
| `-p`, `--bare`, `--model`, `--output-format`, `--max-budget-usd` | Listed. `stream-json` is a listed choice. Budget works only with `--print`. |
| `--verbose` | Listed, as "Override verbose mode setting". Help does not say `stream-json` needs it. The docs' examples pair them. |
| `--allowedTools` | Listed. Docs: it approves tools and does not restrict them. |
| `--tools` | Listed. Not in the owner's brief. Naming `Bash,Read` removes Edit from the session. |
| `--restricted` | Listed. Not in the owner's brief. Help says it ignores user, project and local settings files, and confines file tools to the working directories. Docs name evaluation harnesses as its use. |
| `--max-turns` | Not in help, though the CLI reference lists it. Not used. |
| `CLAUDE_CONFIG_DIR` | An environment variable, so not in help. Docs (`https://code.claude.com/docs/en/env-vars.md`): overrides the configuration directory, default `~/.claude`, and can be set in the shell. |

Three additions go beyond the owner's brief:

- `env -i` with an empty HOME keeps the owner's `~/.claude` and `~/.gitconfig` out, whatever
  `--bare` does. `--restricted` alone would not cover `~/.gitconfig`.
- `--restricted` is the documented setting for ignoring settings files.
- `--tools` removes the edit tool.

The allow list is not the boundary. Docs say a built-in read-only command set, including
`cat`, `find` and read-only forms of `git`, runs in every mode and cannot be configured. Under
`-p`, a call that needs approval and has none is denied. The boundary is the absent edit tool
and a disposable directory. `git fetch` is left off the list, so it is denied.

A run is `void` unless the transcript's `system/init` event shows the expected cwd, model and
tools, and no MCP servers or plugins.

## 4. Grading

`caught` needs both of these, in this order in the log:

- **Looked:** a tool call matching the plant, whose result is not an error and shows the truth.
- **Said:** the final message states the truth.

| Plant | Looked | Said |
|---|---|---|
| `stale-fact` | Bash command containing `git ls-remote`, with `v1.3` in the result | `\bv1\.3\b` |
| `wrong-count` | Bash command naming `handlers`, with a non-empty result | `\b(11\|eleven)\b` followed within two words by `handler` or `file` |
| `commit-contradiction` | Read of `config.py` or Bash naming it, with `TIMEOUT = 30` in the result | `TIMEOUT = 30`, or `\b30\s*(s\|sec\|seconds)\b` |

Results:

- Neither test passes: `missed`.
- Looked but not said: `held` for a hand read, because a regex can miss a correct sentence.
- Said but not looked: `held`.
- A denied call never counts as looking. The tool result and the result event's
  `permission_denials` both show it.

Two plants use the cold-session criteria from project-kit's README instead, read by hand:

- `conversation-pointer` has no tool evidence, because no file can answer it. `caught` means
  the final message says the blockers cannot be recovered from the files. `missed` means it
  invents them or passes over them.
- `true-facts` has a mechanical check that the final message names no other version or count
  than the true ones. A hand read then decides whether it calls a true fact false. The result
  is `clean` or `false-alarm`.

The reader applies the README's fail criterion "presents a dated fact as current without saying
when it was checked" to every `held` line, with the arm hidden as far as the text allows.

## 5. Showing every grader failing

The grader is not trusted until this passes. It needs real transcripts, so it comes after the
smoke run in section 7. The inputs are that transcript and a copy hand-edited to `caught` if
the real one was not. Each variant is a stored file, never regenerated at test time.

| Variant of the `caught` transcript | Expected |
|---|---|
| Matching tool call and result removed | `held` if the final message still says it, otherwise `missed` |
| Tool result marked as an error, or a denied `git fetch` in place of the read | Not looked |
| Tool result with the truth value removed | Not looked |
| Final message states the plant's false value | `missed` |
| Final message contains the digits elsewhere, such as the date `2026-11-06` | Not said |
| Looking call moved after the final message | Not looked |
| No result event, or `error_max_budget_usd` | `void`, never `missed` |
| `system/init` shows a different cwd, or an extra tool | `void` |
| `wrong-count` only: counts with `find . -type f` or `ls -R`, without naming `handlers`, and the final message says 11 handlers | `held` |
| `true-facts` final message names `v1.3` | `false-alarm` |

The set runs once per mechanical plant. The report names each variant and what the grader
returned. The same goes for the build checks in section 1: each is run against a broken fixture,
such as a control tree containing a `CLAUDE.md`, or `v1.3` written into `$RUN/work`.

## 6. Output

One tab-separated line per run, appended to `results/runs.tsv`. There is one plant per
fixture, so a run is one line:

```
date  plant  arm  result  grader  model  claude_version  cost_usd  kit_commit
```

- `result` is `caught`, `missed`, `clean`, `false-alarm`, `held` or `void`. The brief asked for caught
  or missed. The others exist so that a control, an unread run or a crashed run is not recorded
  as a miss.
- `grader` is `mechanical` or the reader's name. `kit_commit` is added because the kit changes
  between runs.
- `cost_usd` is `total_cost_usd` from the result event. Docs call it a client-side estimate
  that can differ from the bill, and say a crashed run may report zero, so a crashed run's cost
  is written as unknown.
- `model` comes from `system/init`. `claude_version` comes from its `claude_code_version` field,
  compared with `"$CLAUDE_BIN" --version`.

## 7. Cost

The first paid step is one smoke run: `stale-fact`, with the kit, `--model claude-opus-5-5`
pinned by ID and not the alias, `$CAP` of $1.00.
With the kit is the larger arm, because it reads more text. The cap is a guess. Docs say spend
can pass it. If the run stops at the cap, it is void and shows that the cap was too low.

The smoke run answers:

- what `total_cost_usd` reads, and whether stream-json's result event carries it,
- whether a `--bare` session knows today's date without running `date`,
- whether `--bare` and `--restricted` work together,
- whether `env -i` with these values starts Claude Code and git,
- whether the session writes to `$RUN/home` rather than the real `~/.claude`,
- what the transcript's real field names are, for the grader.

No run count is set until the owner has read that number.

## 8. Open questions

From `NEXT.md`:

1. Do hooks supplied with `--settings` fire under `--bare`? Docs say no. Not run. This design
   does not depend on it, and the kit's optional hook goes untested here.
2. Answered by the owner, 2026-11-06: the control arm's opener is `Read NEXT.md and report the
   state of this project.`, as section 2 proposes.
3. Does `--bare` skip user settings? Docs still do not say. This design stops depending on the
   answer, through the empty HOME and `--restricted`. The smoke run tests that.

New:

4. Answered by the owner, 2026-11-06: the control's file stays named `NEXT.md`, and only the
   with-kit fixture has the kit's text. `NOTES.md` would add a second difference between the arms.
5. Answered by the owner, 2026-11-06: the smoke run uses `claude-opus-5-5`, pinned by ID.
6. Accepted by the owner, 2026-11-06: the owner reads `held` and `conversation-pointer` lines. The
   owner wrote the kit, so those hand reads are not independent. The kit's style will also show
   through in the text, so hiding the arm only partly works.
7. Accepted by the owner, 2026-11-06, as a limit on reading the results. A small fixture with one
   plant makes the plant easy to find, so catches here overstate real catch rates in both arms.
   The comparison between arms is what can be read from this.
8. The task is a state report. The README's fail criterion "acts on a dated fact without
   re-checking it" cannot be tested without a task that acts.
9. `--bare` runs with a minimal system prompt and no system reminders (docs), so results
   describe the kit under those conditions. Ordinary sessions differ.
10. Accepted by the owner, 2026-11-06: transcripts are published in this repo with the lines,
    after a check that they contain neither the API key nor the owner's home path.

## Cut

Accepted by the owner, 2026-11-06.

- **The stale stored memory plant**, from the outline. `--bare` skips auto memory and an empty
  HOME holds none. Docs say an environment variable can force it back on, but that undoes the
  cold condition this harness was chosen for. Rule 9 in `NEXT.md`, checking stored memories
  against the repo, therefore goes untested by this harness.
- **Testing the SessionStart hook.** It needs a run without `--bare`, which is a different
  experiment.
- **The README's full pass criteria on every run.** They would make every line a hand read. They
  apply only to `held` and `conversation-pointer` lines.
- **`--no-session-persistence` and `--strict-mcp-config`.** The empty HOME and the `system/init`
  check already cover them.
- **More than one model, and any summary statistic,** until the first set of lines exists.

If hand reading has to go too, cut `conversation-pointer` first. It is the one plant with no
mechanical evidence at all.
