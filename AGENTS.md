# Agent Instructions for MINGW-packages

Git for Windows is built on top of MSYS2, which provides the Unix
build environment (bash, make, autotools, Perl, etc.) and the
package management infrastructure (pacman, makepkg). The Git for
Windows SDK is a full MSYS2 installation with additional tools for
building Git for Windows itself.

MSYS2 has two classes of packages: MSYS packages (which depend on
the MSYS2 runtime, `msys-2.0.dll`, a Cygwin fork) and MINGW
packages (which produce native Win32/Win64 binaries with no MSYS2
runtime dependency). This repository contains MINGW packages. The
build tool is `makepkg-mingw`, not `makepkg`.

## Building packages

MINGW builds need the target toolchain on PATH. In PowerShell:

    $env:MSYSTEM = "MINGW64"
    $env:PATH = "<MSYS-root>\mingw64\bin;<MSYS-root>\usr\bin;$env:PATH"
    $env:HOME = $env:USERPROFILE
    bash -c "cd /path/to/package && makepkg-mingw -s 2>&1"

Do not use `bash -l` or `bash --login`; login shells may hang in
Git for Windows SDK environments.

Use `makepkg-mingw -s` to install missing dependencies
automatically. Use `--nocheck` to skip the test suite, `--cleanbuild`
to force a clean rebuild, and `--skippgpcheck` to skip GPG signature
verification. Use `makepkg-mingw -o` (extract and prepare only)
to validate that patches apply cleanly without waiting for a full
compile.

When a package uses git sources, `updpkgsums` must be run with
`core.autoCRLF=false` in the Git configuration. Otherwise the
generated `.tar` archive is checksummed with CRLF line endings
and CI (which uses LF) will fail the validity check. Set the
environment variable in PowerShell before invoking bash:

    $env:GIT_CONFIG_PARAMETERS = "'core.autoCRLF=false'"
    bash -c "cd /path/to/package && updpkgsums"

## Running tests without rebuilding

After a successful build (even with `--nocheck`), the build directory
at `src/build-<MSYSTEM>/` contains all compiled binaries and test
infrastructure. Run individual tests or the full suite directly:

    cd src/build-MINGW64
    make VERBOSE_FAILURE=1 TESTS='test_ocsp_cert_chain test_store' test

To run the full suite (excluding a specific test):

    HARNESS_JOBS=8 make VERBOSE_FAILURE=1 TESTS=-test_symbol_presence test

`HARNESS_JOBS` controls Perl Test::Harness parallelism. Without it,
tests run sequentially, which can take over an hour for large suites
like OpenSSL. A value of 8 is a reasonable default.

## Perl $^O on Git for Windows

Git for Windows recently stopped building its own Perl and now uses
the MSYS2 version. Because MSYS2's runtime is a Cygwin fork,
`/usr/bin/perl` reports `$^O` as `cygwin`, not `msys`. Any test
skip guard that checks for `msys` (e.g. `skip if $^O =~ /msys/`)
must also check for `cygwin`, or it will stop firing and the test
will run unexpectedly.

This is particularly dangerous for tests that open listening sockets
(OCSP responders, HTTP servers, TFO listeners), because Windows
Firewall will prompt for permission. On interactive machines this
causes a modal dialog; on CI agents there is nobody to click
"Allow", so the test hangs until the job times out.

## The playground repository

When upgrading a package to a new version, patches often fail to
apply because the upstream context has changed. Rather than hand-
editing hunks, create a standalone Git repository at `src/playground`
inside the package directory and use it to rebase the patches:

1. Import the old upstream tarball with Git's
   `contrib/fast-import/import-tars.perl` (run via
   `<MSYS-root>\usr\bin\perl`; the script can be downloaded from
   https://github.com/git/git/blob/HEAD/contrib/fast-import/import-tars.perl).
2. Apply the existing `.patch` files as individual commits using
   `git am`. For patch files that lack a `git am`-format header,
   start `git am` with a properly formatted copy to capture
   authorship and subject, then when it fails (because the diff
   content is stale), apply the correct patch via `patch -p1`,
   `git add -A`, and `git am --continue`. Check for stray `.orig`
   files and remove them before or as part of the amend.
   (`--committer-date-is-author-date` is not needed; step 5 forces
   committer=author via fast-export/fast-import regardless.)
3. Import the new upstream tarball into the same repo.
4. `git rebase --onto <new-tag> <old-tag> <branch>` to replay the
   patches onto the new version. Resolve conflicts.
5. Before exporting, verify committer matches author in every
   commit (to keep `.patch` files stable across Git versions):

       git log --format='%aN <%aE> %ai%n%cN <%cE> %ci' HEAD | uniq -u

   Each commit produces two consecutive lines (author, then
   committer). When they match, `uniq` collapses them and
   `uniq -u` suppresses the result, so empty output means all
   OIDs are stable. Use `HEAD` (not a range like
   `<base-tag>..HEAD`) so the import commit is checked too; a
   mismatched committer date there changes the OID of every
   descendant. If any lines appear, fix them with:

       git fast-export --no-data HEAD | \
         awk '/^author /{a=$0} /^committer /{$0="committer " substr(a,8)} 1' | \
         git fast-import --force --quiet

   Then update the tag and branch ref (`git checkout <branch>`).
6. `git format-patch --no-signature -o ../.. <base-tag>` to export
   back to the PKGBUILD directory.
7. Verify the round-trip: re-export `format-patch` from the
   playground and `git diff -I'^@@'` against the originals. Only
   commit OIDs, index lines, and `@@` offsets should differ.

`patch -p1` supports fuzz matching that `git apply` does not. When
`git am` or `git apply` fails on context lines, fall back to
`patch -p1` and stage the result.

Keep the playground around for the duration of the upgrade work. New
fixes and patch adjustments are committed here, then re-exported with
`git format-patch --no-signature -o ../.. <base-tag>`. Deleting and
recreating it means re-importing tarballs, re-applying all patches,
and re-resolving any conflicts from scratch.

When an existing patch needs updating (e.g. a patch that introduces
MSYS-specific handling needs to cover a new platform variant), amend
the original commit in the playground rather than layering a new
patch on top. Use `amend!` commits and
`git rebase -i --autosquash` to fold the fix into the right commit,
then re-export all patches. This keeps the patch series clean and
makes the intent of each patch self-contained.

## General workflow

Use `git worktree add` with sparse checkout for testing dependent
packages. Commit fixes in the worktree and `git cherry-pick` them
into the main branch rather than copying files manually.
