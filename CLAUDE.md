# Handoff: terminal reflow-on-resize feature

## What this branch is

Working on vim's bundled `:terminal` (`src/terminal.c`), branch `reflow`
(currently identical to `master`, both at commit `fc9012d46`, 2 commits
ahead of `origin/master`, **not pushed**). Goal: make `:terminal`
scrollback re-wrap when the window is resized, retrying the intent of
closed upstream PR #19863 but on top of the libvterm already bundled by
patch 9.2.0336. Full history/reasoning is in the local plan file
`~/.claude/plans/hazy-brewing-glacier.md` if more background is needed.

## Commits so far (local only, both new commits, neither amended)

- `a420ee7a7` — "terminal: reflow scrollback when the terminal is resized"
  The feature itself: `vterm_screen_enable_reflow()` turned on in
  `create_vterm()`, `handle_resize()` now calls new `reflow_scrollback()`
  (+ helpers `reflow_group()`, `reflow_flush_fragment()`) which rewraps
  the *persistent* `tl_scrollback` fragments (`[0, tl_scrollback_scrolled)`)
  to the new column width. Buffer text/line count/cursor/folds are never
  touched — reflow only rebuilds `sb_line_T` fragment metadata, because
  continuation-joined wrapped lines are already one physical buffer line
  (patch 9.2.0336). Includes one genuinely-minimal libvterm fix (a
  one-line bound-check change in `src/libvterm/src/screen.c`'s
  `resize_buffer()`, found via ASAN) and new tests in
  `src/testdir/test_terminal.vim` (`Test_terminal_reflow_*`).
- `fc9012d46` — "terminal: fix E315/E340 crash on resize after
  Terminal-Normal round-trip" — see below.

## The E315/E340 bug (fixed, but read this if it resurfaces)

User torture-tested by compiling vim inside `:terminal ++curwin` while
resizing GVIM. Trigger: enter Terminal-Normal mode (`Ctrl-\ Ctrl-N`),
scroll back, return to insert (`i`), *then* resize. Root cause:
`handle_postponed_scrollback()` (flushes `tl_scrollback_postponed` —
output that arrived while in Terminal-Normal mode — into `tl_scrollback`
when normal mode is left) copied every `sb_line_T` field **except**
`continuation`. That flag marks a fragment as a wrapped continuation of
the previous buffer line; losing it desyncs the fragment-to-buffer-line
grouping that `reflow_scrollback()` (and pre-existing `limit_scrollback()`)
rely on, eventually asking `ml_get_buf()` for a line number beyond
`ml_line_count` → `E315`/`E340`.

Fix is one line: `line->continuation = pp_line->continuation;` in
`handle_postponed_scrollback()` (`src/terminal.c`). Regression test:
`Test_terminal_reflow_postponed_continuation` in `test_terminal.vim` —
verified it fails deterministically without the fix (3/3 runs) and
passes with it.

**If E315/E340 resurfaces**: first confirm which binary produced it —
this session repeatedly got confused by stale/system vim binaries and by
the user's own concurrent builds in this same working tree clobbering
`src/vim` mid-investigation. Rebuild (`make -j$(nproc)` in `src/`) and
retest before assuming it's a new bug. The debugging technique that
actually worked (after an earlier misplaced breakpoint gave a *wrong*
backtrace by snapping to a function entry line): break directly on the
message function with a condition matching the exact format string,
e.g.:
```
gdb -q -batch -ex "break siemsg if \$_streq(s, \"E315: ml_get: Invalid lnum: %ld\")" \
    -ex "run -f -u NONE -N --not-a-term -S repro.vim" -ex bt ./vim
```
A misplaced `break file.c:LINE` can silently snap to the wrong line and
catch unrelated calls — cross-check any suspicious backtrace against a
script that logs its own state (e.g. a resize counter) before trusting it.

## Testing

Run the full terminal suite from `src/testdir/`:
```
make test_terminal VIMPROG=../vim
```
One test, `Test_aa_terminal_focus_events`, fails/crashes vim in this
sandboxed environment (confirmed pre-existing on `a420ee7a7` before any
of this session's changes — not a regression, likely missing real-tty/
screendump support here). Skip it to see the rest:
```
TEST_SKIP_PAT='Test_aa_terminal_focus_events' VIMRUNTIME=../../runtime \
  ../vim -f -u util/unix.vim --gui-dialog-file guidialog -U NONE --noplugin \
  --not-a-term -S runtest.vim test_terminal.vim \
  --cmd 'au SwapExists * let v:swapchoice = "e"'
```
All other 106 (107 including the new regression test) pass. `test.log`
is only created on failure — its absence means a clean run. Clean up
`test.log`, `messages`, `test_terminal.res`, `guidialog` after manual
runs (not gitignored as build artifacts in every case — check `git
status` before committing).

Performance was benchmarked earlier in this session (2-6ms per resize
with 4000-8000 lines of scrollback) — no regression vs. the "painfully
slow" behavior the user reported from an earlier, different reflow
attempt (Cimbali's pre-merge patch). Re-benchmark if `reflow_scrollback`
changes.

## Standalone libvterm branch (separate repo, for upstreaming)

`~/libvterm/trunk` is a `brz` (Bazaar/Breezy) stacked branch of
`https://bazaar.leonerd.org.uk/c/libvterm/` (revno 844).
`~/libvterm/fix-reflow-row-underflow` (revno 845) contains **only** the
one-line `resize_buffer()` fix, ported to current upstream (which no
longer has the `REFLOW` macro vim's bundled copy still has — upstream
inlined it, use `screen->reflow` there instead). Verified against the
full 43-test upstream libvterm suite. This branch is ready for the user
to submit upstream; not yet pushed/proposed anywhere by me. Building
standalone libvterm requires generating gitignored `.inc` files first:
```
perl -CSD tbl2inc_c.pl src/encoding/X.tbl > src/encoding/X.inc   # per encoding table
perl find-wide-chars.pl > src/fullwidth.inc
```
then compile `t/harness.c src/*.c` directly with `gcc` (no `libtool` in
this environment) and loop `perl t/run-test.pl -e "$HARNESS" "$f"` over
`t/*.test` (its runner only accepts one file at a time).

## Working-tree hygiene notes

- The user works in this same `/home/staufk/vim` tree concurrently —
  `src/vim` and other build artifacts have gone missing/changed
  mid-session before. Don't assume the binary on disk matches the last
  build you did; rebuild before trusting a repro.
- User's stated preference this session: **new commits only, never
  amend**, and **commit locally, do not push**, confirmed twice via
  explicit prompts. Don't push to `origin` or squash without asking again.
- User explicitly asked to keep libvterm-side changes to an
  absolute minimum to maximize odds of upstream acceptance — this
  shaped the whole design (one enable call + one bugfix line in
  libvterm; everything else is vim-side).

## Not yet done / open threads

- The libvterm fix branch (`~/libvterm/fix-reflow-row-underflow`) has
  not been proposed/emailed upstream — that's on the user.
- No further torture-testing has been requested beyond what's recorded
  above; if the user reports another crash, get their exact repro
  sequence and confirmed binary/build state before diving into gdb —
  this session burned significant time on stale-binary and
  wrong-breakpoint false leads.
