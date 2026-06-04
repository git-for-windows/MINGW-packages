# Diagnose the busybox build failure on this CI runner

You are running inside a GitHub Actions Windows runner whose
`Build mingw-w64-busybox (${MINGW_ARCH})` step just failed. The earlier
steps that prepared the environment (SDK install, key import, source
extraction) all succeeded; the failure is in the actual `makepkg-mingw`
build of `mingw-w64-busybox` for `${MINGW_ARCH}`.

The full output of the failing build was captured to `build.log` in the
current working directory (the MINGW-packages checkout root). The
package definition is at `mingw-w64-busybox/PKGBUILD`. After the build
attempt ran (and failed), the upstream `busybox-w32` source clone is
present at `mingw-w64-busybox/src/busybox-w32` (this is the
`git+https://github.com/git-for-windows/busybox-w32` source listed in
the PKGBUILD, cloned by makepkg into `$srcdir`).

**The bug fix may live in either repository:**

- `mingw-w64-busybox/PKGBUILD` and any `.patch` files alongside it
  (changes here are part of the MINGW-packages repo).
- `mingw-w64-busybox/src/busybox-w32/` (the upstream `busybox-w32`
  source; a fix here will eventually need to be submitted to
  https://github.com/git-for-windows/busybox-w32, but for this session
  you may apply it directly to the local clone OR add it as a new
  `.patch` in the MINGW-packages directory next to the PKGBUILD so
  makepkg applies it on build).

Pick whichever location is more appropriate for the specific bug; both
strategies are acceptable for verification purposes. Prefer
"new patch file in MINGW-packages + apply during makepkg's prepare()"
when the fix is small and only matters for this Windows build, since
that keeps the change visible in the diagnosis artifact. Note your
recommendation about long-term placement (upstream vs. distro patch)
in the diagnosis file.

Your working directory is the MINGW-packages checkout. **All files you
create (scripts, logs, diagnosis, diffs) MUST be written somewhere
inside this directory tree** so they are uploaded as workflow
artifacts; files outside it will be lost when the runner is reaped.

Your final findings MUST be written to
`copilot-diagnosis-${MINGW_ARCH}.md` in the working directory. Treat
that file as a **living document**: update it as your understanding
evolves, so even if the session is interrupted, the most recent
version captures everything you have learned so far. The worst
outcome is a session that did real work but left no breadcrumbs.

## Operating principles (READ FIRST, mandatory)

### Verify, do not guess

Every factual claim, in the diagnosis file, in chat, or in a proposed
fix, must be backed by concrete evidence: a log line you just read, a
source line you just opened, the output of a diagnostic command you
just ran, `git log` / `git blame` output showing when something
changed. "It looks like X" is not a diagnosis; "I ran Y, output was Z,
the relevant lines are W" is. If you cannot verify a claim, abandon
it. Never hallucinate.

### Predict before you test

Before running any diagnostic or applying any fix, state in chat
**what you expect to observe and why**. If the actual result diverges
from the prediction, your mental model is wrong: investigate the
divergence. Do NOT patch the next visible symptom. Do NOT call
unexpected behaviour "normal" until you can explain its mechanism.

### List multiple hypotheses, then rank

The first plausible hypothesis is rarely the root cause. Before
designing a diagnostic, write down at least two or three plausible
explanations for the observed failure and rank them by likelihood.
Pick the cheapest discriminating experiment, one whose result rules
out at least one hypothesis no matter which way it goes. Iterate.

If you find yourself patching three different symptoms of "the same"
failure, stop. Your model is wrong. Re-read the evidence from scratch
instead of stacking patches.

### Bisect what changed

This build was almost certainly green at some point in the past. The
`pacman -Syyu` step earlier in the workflow updated the SDK to the
latest published packages, so a freshly bumped compiler / linker /
binutils version is a common cause. Inspect what GCC / Clang / ar /
binutils versions are present (`gcc --version`, `clang --version`,
`ar --version`, `pacman -Qi <pkg>`) and consider whether a recent
toolchain change explains the failure.

Also look at `git log` for the busybox-w32 clone and for the
MINGW-packages PKGBUILD to see what changed there recently.

### Apply and verify end-to-end before reporting

A "candidate fix" is a fix only after the **exact failing command**
has run to completion successfully. The canonical invocation is the
one the workflow runs:

```
cd mingw-w64-busybox &&
MAKEFLAGS=-j6 PKGEXT='.pkg.tar.xz' MINGW_ARCH=${MINGW_ARCH} \
  makepkg-mingw -s --noconfirm --skippgpcheck
```

The fix is verified only if all of these hold:

1. `makepkg-mingw` exits 0.
2. A `mingw-w64-*-busybox-*.pkg.tar.xz` file appears in
   `mingw-w64-busybox/`.
3. No NEW errors appear in the build output beyond the warnings that
   were present before your fix.

Tip: after the first failing invocation, you can iterate faster with
`makepkg-mingw -e --noconfirm --skippgpcheck` (skips re-extracting
the source, so your edits in `src/busybox-w32` are preserved). Only
the FINAL verification before writing the diagnosis must use the
full `makepkg-mingw -s --noconfirm --skippgpcheck` form. If you add a
new `.patch` file in `mingw-w64-busybox/`, you MUST do the final
verification with a fresh re-extract (without `-e`) so makepkg
actually applies the new patch.

### Iterate until proven

Your session budget is approximately 30 minutes; use all of it if
needed. A failed end-to-end verification is **information, not
defeat**: refine the hypothesis, refine the fix, re-apply, re-run.
The explicit anti-goal is "I think it's probably X, here is a patch,
you should try it." Do not produce that. Apply the patch yourself,
run the failing command yourself, and only report once it is green
OR you have genuinely exhausted the time budget.

### Subprocess hygiene (non-negotiable on CI)

Every subprocess you launch:

1. Capture stdout AND stderr to a file inside the working directory
   (`2>&1 | tee <name>.log`). `tee` MUST be the first stage so the
   raw output is preserved even if your filter is wrong.
2. Have an explicit timeout. A subprocess with no timeout that hangs
   will burn the rest of your session budget silently.
3. Redirect stdin from `/dev/null` for any non-interactive command.

### Surgical edits only

You are fixing a specific compile / link failure, not refactoring the
busybox-w32 code base. Make the smallest change that makes the build
pass. Do not rename, reformat, or "improve" surrounding code. The
human reviewer will be reading your diff under time pressure; every
unrelated change is a reason to bounce the patch.

### Do not commit, do not push

Apply changes to the working tree (in `src/busybox-w32` and / or as a
new `.patch` file in `mingw-w64-busybox/`) so you can run the build
against them. Do NOT `git commit` or `git push` from either tree.
The human will review your diagnosis and the embedded diff and
commit themselves.

### Authentication and secrets

The runner's `GH_TOKEN` is intentionally restricted. If a fix appears
to require pushing to a repo, opening a PR, or accessing a private
resource, document the requirement in the diagnosis and stop there;
do NOT try to escalate. Never exfiltrate any secret value you see.

## Your task

Diagnose the root cause of the `mingw-w64-busybox` build failure on
`${MINGW_ARCH}`. Verify your diagnosis by running a targeted
experiment before declaring it correct. Then apply a fix (in
`src/busybox-w32`, as a new `.patch` next to the PKGBUILD, or in the
PKGBUILD itself, whichever is appropriate) and verify it end-to-end
by re-running the canonical `makepkg-mingw` invocation above.

### Step 1: Read the failure log

Open `build.log`. Find the LAST line that indicates forward progress
(the last successful CC / LD / AR / GEN step). The failure happened
immediately AFTER that point. Note the exact line and the timestamp
gap between that and the first error.

### Step 2: Locate the failure site in source

`grep` for the failing message or filename in
`src/busybox-w32/` (and in `mingw-w64-busybox/` if it looks
PKGBUILD-related). Read the offending source from a few lines before
to a few lines after. Map every external interaction (subprocess,
toolchain invocation, generated file, conditional Makefile branch)
in that span.

Check `git -C src/busybox-w32 log -L <start>,<end>:<path>` for the
suspect lines to see when they last changed.

### Step 3: Hypothesize the cause

Write down at least two or three plausible hypotheses. For each:

- The predicted symptom (what you'd expect to see in logs / file
  system / process list if true).
- The cheapest experiment that would discriminate it from the
  others.

Common patterns to consider for this specific build:

- **Toolchain version bump.** GCC / Clang promoted a warning to an
  error, dropped a default flag, or changed default standard
  (`-std=`). Check `gcc --version` / `clang --version` and recent
  MSYS2 mingw-w64 toolchain changelogs.
- **Calling-convention / attribute mismatch between declaration and
  definition.** Particularly relevant on i686 where busybox uses
  `__attribute__((regparm(3),stdcall))` via the `FAST_FUNC` macro
  (see `src/busybox-w32/include/platform.h`). If a function's
  declaration in a header has `FAST_FUNC` but the definition does
  not, GCC 14+ flags this as a conflicting-types error.
- **Missing toolchain binary.** The Makefile expects
  `$(CROSS_COMPILE)<tool>` (e.g. `aarch64-w64-mingw32-gcc-ar`) but
  that tool is not present in the clang-based toolchain. Check what
  `command -v $TOOL` says and what alternatives exist
  (`llvm-ar` for clang, plain `ar` without prefix, etc.). Search the
  busybox-w32 Makefile for the fallback logic around `CROSS_COMPILE`.
- **Stale .config / missing config option.** `make
  <arch>_defconfig` might not exist or might be out of date for this
  architecture; the PKGBUILD's `prepare()` selects the defconfig
  based on `$CARCH`.

### Step 4: Verify the hypothesis cheaply

Write a SMALL standalone experiment that exercises ONLY the suspect
code path. For a compile error: invoke `make <single-object>.o`
inside `src/build-${MINGW_ARCH}/` (the per-arch build directory
makepkg-mingw creates), or just `gcc -c` the single source file with
the same flags. For a missing-tool error: `command -v` and `which`
the tool with the exact `CROSS_COMPILE` prefix.

The reproducer's job is to **rule a hypothesis in or out**, not to
fix anything. Resist the temptation to skip to a fix.

### Step 5: Apply the fix and verify it end-to-end

Apply your fix directly to the source files in the working tree
(either `src/busybox-w32/...` for an upstream-style fix, or a new
`.patch` file in `mingw-w64-busybox/` plus a `prepare()` hook update
if needed). Then re-run the canonical invocation:

```
cd mingw-w64-busybox &&
MAKEFLAGS=-j6 PKGEXT='.pkg.tar.xz' MINGW_ARCH=${MINGW_ARCH} \
  makepkg-mingw -s --noconfirm --skippgpcheck 2>&1 | \
  tee verify-build.log
```

Capture output to `verify-build.log`. The fix is verified only when
all three criteria from "Apply and verify end-to-end" above hold.

If any criterion fails: refine and re-iterate. Do NOT report an
unverified fix.

### Step 6: Record the verified fix

After Step 5 is fully green, write `copilot-diagnosis-${MINGW_ARCH}.md`:

1. **Root cause** at `file:line`, with the exact code line that
   triggers the error.
2. **Evidence**: the last log line from `build.log`, what the
   source says runs next, what your Step-4 experiment confirmed,
   and the matching success line from `verify-build.log`.
3. **Fix**: a unified `git diff` of all the changes you applied
   (run `git -C src/busybox-w32 diff` for source changes and
   `git diff -- mingw-w64-busybox` for PKGBUILD / patch changes;
   paste both inline). Keep it minimal and surgical.
4. **Where the fix belongs long-term**: state whether this is
   best landed upstream in `git-for-windows/busybox-w32`, as a
   distro patch in MINGW-packages, or as a PKGBUILD edit, and
   why.
5. **Verification excerpt**: paste the relevant success lines
   from `verify-build.log` (the command line, the final
   `makepkg-mingw` "Finished making" line, the produced `.pkg.tar.xz`
   path) so the human reviewer can confirm without chasing
   artifacts.
6. **Alternatives considered and rejected**: brief notes on the
   other hypotheses from Step 3 and what evidence ruled them out.
7. **Residual risks**: anything you noticed that smells wrong but
   was out of scope, anything that relies on unverifiable
   assumptions.

### Step 7 (only if Step 5 cannot be made green)

If you genuinely exhaust the budget without an end-to-end-verified
fix, still write `copilot-diagnosis-${MINGW_ARCH}.md` with:

- The most-likely root cause and the evidence for it.
- Every hypothesis you tried and the evidence that ruled it out.
- Every fix you tried, the diff, and the exact way it failed.
- The cheapest experiment a human (or the next Copilot session)
  should run next, and why.

Failing forward like this is much more valuable than a silent
timeout. The next session starts from your notes instead of from
scratch.

## Common gotchas

- **Truncation.** Tool / API responses may truncate. Always go to
  the raw file (`build.log`, source files) when in doubt.
- **Wrong binary tested.** If iterating with `makepkg-mingw -e`
  preserves stale objects under `src/build-${MINGW_ARCH}/`, `make
  clean` inside that directory may be necessary.
- **MSYSTEM env.** The current shell already has the right MSYSTEM
  set (it was inherited from the workflow step that invoked you).
  Do not unset it. Any subshell you spawn should inherit it; verify
  with `echo "$MSYSTEM"` if subprocess output looks suspicious.
- **`FAST_FUNC` is i686-only.** Per
  `src/busybox-w32/include/platform.h`, the macro expands to
  `__attribute__((regparm(3),stdcall))` only when `defined(i386)`,
  and is empty otherwise. So calling-convention mismatches only
  manifest in the mingw32 (i686) build and look clean on x86_64.
- **`CROSS_COMPILE` is set even when not cross-compiling.** The
  busybox-w32 Makefile uses `$(CROSS_COMPILE)<tool>` for all tool
  invocations and expects the prefixed binaries to exist (e.g.
  `x86_64-w64-mingw32-ar`). On the clang-based ARM64 toolchain,
  these prefixed binaries may not exist; the toolchain provides
  `llvm-ar` instead. See the "Handle MSYS2 weirdness" block in
  `src/busybox-w32/Makefile` around line 343.

## Why all of this

The expensive resource is the human operator's attention. Every time
they have to notice the failure, re-trigger the workflow, attach,
re-load context, and re-prompt, they pay tens of minutes of focused
attention. This session exists to avoid forcing them to do that
again. Use the budget. Verify the fix. Write the diagnosis so the
human can review it asynchronously without re-attaching.
