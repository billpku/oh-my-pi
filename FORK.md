# Fork: timed auto-accept for plan approval

This fork adds a countdown to the plan-approval overlay so an unattended session can run
**plan → implement** end to end instead of blocking forever on the approval gate.

Everything else is stock upstream [`can1357/oh-my-pi`](https://github.com/can1357/oh-my-pi).

- Upstream PR: [can1357/oh-my-pi#11166](https://github.com/can1357/oh-my-pi/pull/11166)
- Branch: [`plan-approval-timeout`](https://github.com/billpku/oh-my-pi/tree/plan-approval-timeout)

---

## The problem

Upstream, both places the agent waits for a human block indefinitely:

1. The **plan-approval overlay** has no timeout at all.
2. **`ask` prompts in plan mode** have their timeout force-disabled, even though the countdown
   machinery already exists and works everywhere else:

   ```ts
   // src/tools/ask.ts — upstream
   const timeout = planModeEnabled ? null : settingsTimeout;
   ```

Step away after the model writes a plan and the session sits there. Come back an hour later to
find nothing happened.

## What this fork changes

| Setting | Type | Default | Effect |
| --- | --- | --- | --- |
| `plan.approvalTimeout` | number (seconds) | `0` | Countdown on the plan-review overlay. `0` waits indefinitely (upstream behavior). |
| `plan.approvalDefault` | `execute` \| `compact` \| `keep-context` | `execute` | Which option the expiry commits. |
| `ask.timeoutInPlanMode` | boolean | `false` | Apply `ask.timeout` during plan mode. |

**All three defaults reproduce upstream behavior exactly.** Nothing changes until you opt in.

### Behavior

- Remaining time renders on the prompt line: `Plan mode - next step (20s)`, ticking once a second.
- **Any keystroke restarts the full window** — including inside the annotation sub-mode. If you
  are present and reading, you are never rushed.
- On expiry the overlay commits `plan.approvalDefault`, the session leaves plan mode, and the
  execution turn dispatches exactly as if you had pressed Enter.
- The transcript records it: `Plan auto-approved after 600s with "Approve and execute".` It never
  reads as a deliberate choice you made.

### What it will never auto-select

- **`Refine plan`** — expiry means nobody is present; refining would loop the model against no one.
- **`Save and quit`** — would discard the session.
- **A disabled `keep-context` row.** Above the context-pressure threshold that option is dimmed;
  auto-picking it would drive an approval path the interactive branch refuses. It falls back to
  `Approve and execute`, because a fresh-context execute is always available and the alternative
  (skip the auto-select) reintroduces the exact hang this exists to remove.

### Relationship to `--plan-yolo`

Complementary, not a duplicate. `--plan-yolo` is startup-only, one-shot, unconditional, fires on
the model's *first* `xd://propose` with no review window, bypasses the overlay entirely, and
force-switches the execution model. This keeps the overlay fully interactive and only acts when
the window expires.

---

## Configure

```sh
omp config set plan.approvalTimeout 600     # 10 minutes; 0 disables
omp config set plan.approvalDefault execute # execute | compact | keep-context
omp config set ask.timeoutInPlanMode true   # also let plan-mode asks time out
omp config set ask.timeout 600              # the ask window itself (upstream setting)
```

Or in `~/.omp/agent/config.yml`:

```yaml
plan:
  approvalTimeout: 600
  approvalDefault: execute
ask:
  timeout: 600
  timeoutInPlanMode: true
```

Both are also exposed in `/settings` (Tasks → Modes, and Interaction → Notifications).

### Which default to choose

| Value | Use when |
| --- | --- |
| `execute` | Default. Clears context and implements — the cleanest unattended run. |
| `compact` | Long planning conversation you want summarized before implementation. |
| `keep-context` | Implementation needs the exploration history. Falls back to `execute` under context pressure. |

---

## Install and run

The released binary is a compiled ~135 MB Bun executable with no patchable JS, so this fork has
to run from source.

```sh
git clone https://github.com/billpku/oh-my-pi.git
cd oh-my-pi
bun install
```

The fork's `main` already carries the change; `plan-approval-timeout` is the same commit, kept as
the PR branch.

**Then stage the native addon.** It is gitignored and not in the repo, and without it every CLI
invocation — including `--version` and `config list` — dies with
`Failed to load pi_natives native addon`. Building it needs a Bazel toolchain, so unless you have
one, use the published prebuilt for your platform:

```sh
cd /tmp && npm pack @oh-my-pi/pi-natives-darwin-arm64@18.1.13
tar xzf oh-my-pi-pi-natives-darwin-arm64-18.1.13.tgz
cp package/pi_natives.darwin-arm64.node <repo>/packages/natives/native/
```

Swap the package name for your platform (`pi-natives-linux-x64`, etc.) and version-match it — a
stale addon fails later with `api().vcsDiscover is not a function`.

`packages/coding-agent/scripts/omp` is the supported dev launcher. Alias it — **do not overwrite
`~/.bun/bin/omp`**, so the released binary and `omp update` keep working:

```sh
ln -sf "$PWD/packages/coding-agent/scripts/omp" ~/.bun/bin/omp-dev
omp-dev --version   # omp/18.1.14
```

Both binaries read the same `~/.omp/agent/config.yml`. The released binary starts fine with the
three new keys present — it just ignores them — so you can switch back and forth freely. (Its
`config set` will reject the unknown key names; set them with `omp-dev config set` instead.)

---

## Try it end to end

```sh
cd /tmp && mkdir plan-demo && cd plan-demo && echo "export const x = 1;" > notes.ts
omp-dev config set plan.approvalTimeout 20
omp-dev
```

In the session: enter plan mode, ask for a small plan (`plan adding a hello() function to
notes.ts`), and when the approval overlay appears **type nothing**.

Expected: the prompt line reads `Plan mode - next step (20s)` and counts down; at zero the overlay
commits `Approve and execute`, a notice reports the auto-approval, and the session leaves plan mode
and starts implementing.

Press an arrow key mid-countdown and the timer jumps back to `20s` — proof the window restarts
whenever someone is actually there.

Reset when done: `omp-dev config set plan.approvalTimeout 600`.

---

## Tests

```sh
cd packages/coding-agent
bun run check                                              # oxlint + oxfmt + tsgo
bun test test/interactive-mode-plan-review.test.ts \
         test/modes/components/plan-review-overlay.test.ts \
         test/ask-timeout.test.ts \
         test/countdown-timer.test.ts                      # 120 pass
```

Coverage:

- **`plan-review-overlay.test.ts`** — countdown rendering, auto-pick, keypress restart,
  disabled-target refusal, dispose/cancel/manual-pick teardown, no-timeout inertness.
- **`interactive-mode-plan-review.test.ts`** — settings→index mapping for all three defaults, the
  disabled keep-context fallback, the auto-approval disclosure and its absence on a manual pick,
  plus an end-to-end case that mounts the **real** overlay, lets the countdown expire, and asserts
  the session leaves plan mode and dispatches the execution turn.
- **`ask-timeout.test.ts`** — plan mode still blocks by default; `ask.timeoutInPlanMode` restores
  the countdown.

The full `bun test test/` run has pre-existing failures unrelated to this change
(`SQLITE_IOERR_VNODE` in test tempdirs, plus network-dependent Exa/Jina cases). They reproduce
identically on the unmodified parent commit.

---

## Files touched

| File | Change |
| --- | --- |
| `src/config/settings-schema.ts` | The three new settings. |
| `src/tools/ask.ts` | Gate the plan-mode carve-out behind `ask.timeoutInPlanMode`. |
| `src/modes/components/plan-review-overlay.ts` | `CountdownTimer` wiring, prompt-line seconds, keypress reset, `dispose()`, disabled-target refusal. |
| `src/modes/interactive-mode.ts` | Settings→index mapping, `tui` handle, timer disposal in `#hidePlanReview`, auto-approval notice. |

The countdown reuses the existing `CountdownTimer` primitive with the same semantics `HookSelector`
already applies to `ask.timeout` — no new timer machinery.

## Staying current with upstream

```sh
git remote add upstream https://github.com/can1357/oh-my-pi.git
git fetch upstream
git rebase upstream/main   # on plan-approval-timeout
bun install
```

The change is four source files and additive, so conflicts should be rare. If the upstream PR
lands, drop the fork and go back to the released binary.

## License

MIT, same as upstream.
