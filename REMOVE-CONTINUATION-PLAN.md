# Plan: remove `sb_line_T.continuation` from `tl_scrollback`

Working notes for a fresh session. Base: `master` @ `832242d5c` (upstream 9.2.1125 plus
one unrelated display fix). All line numbers refer to that tree.

## Goal

Store **one `sb_line_T` per buffer line**, with a wrapped line's cells concatenated into
a single `sb_cells` array, and delete the `continuation` flag.

Today there is one `sb_line_T` per *terminal row*. Since patch 9.2.0336 a wrapped
terminal line is joined into one vim buffer line, and `continuation` marks each entry
that is the wrapped tail of the previous one. That leaves two coordinate systems —
fragment rows and buffer lines — and converting between them is what
`bufline_pos_in_scrollback()` and `scrollbackline_pos_in_buf()` exist to do.

### Why

Nearly every terminal-scrollback bug found recently traces to maintaining those two
coordinate systems in parallel:

- E315/E340 crash — `handle_postponed_scrollback()` failed to copy `continuation`.
- The ABA staleness bug in the sparse lookup index (on the `fast-scrollback` branch).
- The mid-group checkpoint bug in the same index.
- The `|| !i` row-0 special case in `limit_scrollback()`, and libvterm's matching
  `old_row >= 0` underflow in `resize_buffer()`.
- The `term_getscrolled()` identity being wrong for wrapped lines (see below).

One structural decision, five independent defects.

### Payoff

- `bufline_pos_in_scrollback()` becomes direct indexing, `tl_scrollback[lnum - 1]`.
- Re-wrapping stored scrollback on resize stops being necessary at all — the buffer line
  is soft-wrapped by vim at the current width, and nothing stores wrap points. (This is
  what `reflow_scrollback()` on the `fast-scrollback` branch was for.)
- A lookup index/cache for `bufline_pos_in_scrollback()` becomes unnecessary — that is
  the whole of the four `fast-scrollback` sparse-index commits.
- `limit_scrollback()` loses its wrapped-line loop and row-0 case.
- `tl_scrollback_scrolled` and `tl_buffer_scrolled` become the same number.

## What `continuation` is used for today

All sites in `src/terminal.c`:

| Line | Use |
|------|-----|
| 68 | field declaration in `sb_line_T` |
| 1424, 1432, 1443 | `scrollbackline_pos_in_buf()` — row → buffer line/offset |
| 1483, 1495, 1499 | `bufline_pos_in_scrollback()` — buffer line → row/column |
| 2292, 2303 | `update_snapshot()` — writes the flag for live-screen rows |
| 3671, 3679 | `limit_scrollback()` — one `ml_delete` per non-continuation entry |
| 3790, 3797, 3802 | `handle_pushline()` — writes the flag, drives `tl_buffer_scrolled` |
| 3845, 3857 | `handle_postponed_scrollback()` — same, for postponed output |
| 5685 | `read_dump_file()` — hardcodes `continuation = 0` |

`libvterm` defines it in `include/vterm.h:305` as "Line is a flow continuation of the
previous", i.e. it marks the *tail*, not "wraps onto the next line".

## Target shape

```c
typedef struct sb_line_S {
    int		sb_cols;	// can differ per line
    int		sb_bytes;	// length in bytes of text
    cellattr_T	*sb_cells;	// allocated
    cellattr_T	sb_fill_attr;	// for short line
    char_u	*sb_text;	// for tl_scrollback_postponed
} sb_line_T;
```

`sb_cols`/`sb_bytes` become the totals for the whole logical line. `sb_fill_attr` is the
fill colour past end-of-line; only the *last* row of a wrapped group has anything past
its end (every earlier row is full by definition of wrapping), so the group's fill attr
is the last row's. **Verify that assumption** — it is the one place concatenation could
silently lose information.

## Consumer-by-consumer

### `term_get_attr()` (4435) — no change needed

```c
    if (term->tl_scrollback.ga_len)
	bufline_pos_in_scrollback(term, lnum, col, &sb_line, &sb_col);
    ...
	if (sb_col < 0 || sb_col >= line->sb_cols)
	    cellattr = &line->sb_fill_attr;
	else
	    cellattr = line->sb_cells + sb_col;
```

This already does exactly the right thing once `sb_cols`/`sb_cells` cover the whole
logical line. Only its helper changes. Note `col` here is already the cumulative offset
from the start of the joined line.

### `bufline_pos_in_scrollback()` (1467) — collapses

The three loops (1483, 1492, 1499) walk fragments to convert a buffer line + cumulative
column into a fragment row + column within that fragment. Under the new model the row is
`lnum - 1` for committed lines and the column passes through unchanged. Keep the
`lnum > tl_buffer_scrolled` branch in some form for the snapshot region (below).

### `limit_scrollback()` (3658) — simplifies

```c
	if (update_buffer && (!sb_lines[i].continuation || !i))
	    ml_delete(1);
	...
    // Continue until end of wrapped line
    for (; todo < gap->ga_len && sb_lines[todo].continuation; ++todo)
	vim_free(sb_lines[todo].sb_cells);
```

One entry == one buffer line, so it becomes one `ml_delete(1)` per entry removed. Both
the trailing loop and the `|| !i` case go away.

### `handle_pushline()` (3711) — the main producer change

Receives one row at a time with a `continuation` argument. When set, it must **append**
the row's cells to the previous entry rather than create a new one:
`ga_grow`-style realloc of `sb_cells`, `sb_cols += len`, `sb_bytes += text_len`,
`sb_fill_attr` overwritten by this row's. The buffer-text side already appends —
`add_scrollback_line_to_buffer(..., continuation)` passes it as the `append` flag (2058).
`tl_buffer_scrolled` is only bumped for non-continuation rows (3802), which is exactly the
"one per buffer line" rule, so that stays but becomes redundant with
`tl_scrollback_scrolled`.

### `handle_postponed_scrollback()` (3820) — same treatment

### `read_dump_file()` (5639) — just drop the `continuation = 0` line

It already produces one entry per buffer line. No other change.

## Hard problem 1: the snapshot region

`update_snapshot()` (2189) appends a transient copy of the **live screen** past
`tl_scrollback_scrolled`, and `cleanup_scrollback()` (2148) removes it again before the
next push. That removal is the difficult part:

```c
    while (term->tl_scrollback_snapshot && gap->ga_len > 0)
    {
	line = (sb_line_T *)gap->ga_data + gap->ga_len - 1;
	if (line->sb_bytes < 0 || (size_t)line->sb_bytes > bufline_length)
	    break;
	bufline_length -= line->sb_bytes;
	if (!bufline_length)
	{
	    ml_delete(curbuf->b_ml.ml_line_count);
	    ...
	}
	vim_free(line->sb_cells);
	--gap->ga_len;
	--term->tl_scrollback_snapshot;
    }
    if (bufline_length < STRLEN(bufline))
    {
	char_u *shortened = vim_strnsave(bufline, bufline_length);
	ml_replace(curbuf->b_ml.ml_line_count, shortened, FALSE);
    }
```

It peels entries off the tail, subtracting each `sb_bytes` from the last buffer line's
length, deleting the buffer line when the length reaches zero and **truncating** it
otherwise. That `ml_replace` truncation is the tell: a snapshot row can be a
*continuation of an already-committed buffer line* — the live screen's row 0 is a
continuation when its head has already scrolled off. Removal relies on per-row `sb_bytes`
to know how much to peel back.

Under concatenation there is no per-row `sb_bytes` to subtract. At most **one** committed
entry can be partially extended (the last one, since the snapshot is always the tail), so
the fix is to record its pre-snapshot extent — e.g. `tl_snapshot_trunc_cols` /
`tl_snapshot_trunc_bytes` on `term_T` — and have `cleanup_scrollback()` shrink that entry
back rather than reconstruct the boundary. Decide this **before** writing any code; it is
the part most likely to go wrong, and the E315 crash lived in this neighbourhood.

## Hard problem 2: re-deriving rows for `term_getline()` / `term_scrape()`

These are the only consumers that address scrollback by *terminal row*, via
`sb_row = term->tl_scrollback_scrolled + row` (6509, 6781). Both take the history path
only when `tl_vterm == NULL` — a finished job. Confirmed empirically: with a live job,
`term_getline(buf, N)` returns an empty string for every N <= 0; once it exits the same
calls return history. So this is a cold, script-invoked path, never a redraw path.

`f_term_scrape()`'s history branch (6807):

```c
	    // vterm has finished, get the cell from scrollback
	    if (pos.col >= line->sb_cols)
		break;
	    cellattr = line->sb_cells + pos.col;
	    width = cellattr->width;
	    ...
	    mbslen = mb_ptr2len(p);
```

It bounds the loop by `line->sb_cols` and walks a text pointer `p` in step. Under
concatenation `sb_cols` is the whole logical line, so the loop would run past one row's
worth. Re-derivation must supply a (start cell, end cell, start byte) triple per row:
walk `sb_cells` summing `width` until reaching `term_cols`, and walk the text in lockstep
with `mb_ptr2len`.

**The risk is double-width characters at a re-derived boundary.** libvterm never splits
one across a wrap, so a double-width cell that would straddle the last column must push
to the next row. Check whether the stored `cellattr_T.width` data is sufficient to
reproduce that exactly.

**De-risk this before touching the data structure**: write the re-derivation function
standalone against the *current* row-granular data and assert it reproduces today's
`term_scrape()` output cell-for-cell, across several widths, with CJK and emoji in the
corpus. If that holds the rest is mechanical; if not, it is better to know while nothing
has been ripped out.

## Counters

`tl_scrollback_scrolled` (rows) and `tl_buffer_scrolled` (buffer lines) become equal.
Audit every use and collapse to one. Related, and worth fixing deliberately rather than
by accident: `:h term_getscrolled()` documents

```
	term_getline(buf, N) == getline(N + term_getscrolled(buf))
```

which is currently **false** for wrapped lines, because 9.2.0336 changed
`f_term_getscrolled()` to return `tl_buffer_scrolled` while `term_getline()` kept
addressing rows. Measured: 200 lines with line 50 wrapping ×3 at `term_cols=50` gives
`term_getscrolled()` == 191 against 193 rows actually scrolled off; lines before the wrap
need offset 193 and lines after need 191, so no single offset works. There is a failing
regression test for this on branch `bug-term_getscrolled` (`Test_terminal_scroll_wrapped`,
commit `31252fb71`). Decide what the identity should mean under the new model.

## Suggested order

1. Write and validate the row re-derivation helper against current data (hard problem 2).
2. Decide the snapshot truncation scheme (hard problem 1).
3. Change the producers: `handle_pushline()`, `handle_postponed_scrollback()`,
   `read_dump_file()`, `update_snapshot()`.
4. Rewrite `cleanup_scrollback()` for the new snapshot scheme.
5. Collapse `bufline_pos_in_scrollback()` and `scrollbackline_pos_in_buf()`.
6. Simplify `limit_scrollback()`.
7. Repoint `f_term_getline()` / `f_term_scrape()` at the re-derivation helper.
8. Delete the field; collapse the counters.

Build and run `make test_terminal` after each step, not at the end.

## Tests

Existing net: `make test_terminal` is 107 tests and passes clean on `832242d5c`
(`Test_terminal_eof_arg_win32_ctrl_z` skips, MS-Windows only). Also run
`test_textprop.vim`.

Not currently covered, and needed here:

- Double-width / CJK characters spanning a re-derived row boundary.
- A line wrapped across many rows (most existing tests wrap once, which hides
  accumulation bugs — that is how the per-row-reset behaviour of `vcol_off_co` stayed
  invisible).
- `'termwinscroll'` trimming that lands inside a wrapped line.
- Terminal dump load/diff round-trip of a wrapped line. Note `read_dump_file()` sets
  `continuation = 0` unconditionally today, so dumps already do not round-trip wrapped
  lines — a pre-existing inconsistency worth confirming rather than preserving.

The six `Test_terminal_reflow_*` tests are **not** on this branch; they were removed from
`832242d5c` because the reflow implementation they exercise never landed here. If the
reflow branch is revived they travel with it — but if this plan is carried out, most of
them describe behaviour that no longer needs to exist.

## Style

Match vim's comment density: one short line by default. Upstream `sb_line_T` field
comments are three to eight words (`// can differ per line`, `// for short line`); function
headers are two to four lines saying what the function does and what its parameters mean.
Reserve a longer comment for genuine subtlety — vim has a few, e.g. the "Special voodoo
required if 'wrap' is on" block in `drawline.c`. Put design rationale, rejected
alternatives and bug archaeology in the **commit message**, which is where vim's project
culture keeps it.
