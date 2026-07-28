# Lifeform

A single HTML file that runs Conway's Game of Life, stores the entire grid inside its own source code, and reproduces by writing a new copy of itself containing the next generation.

Open it. It decodes its genome, advances one generation, and hands you a file. That file is the offspring. Open the offspring and it does the same. The file is not running the simulation — the file **is** the simulation, and the save-and-open cycle is its metabolism.

**[Live demo](https://kaymyg.github.io/lifeform/)** — every visitor starts from the same genesis ancestor.

No dependencies, no build step, no network. 17 KB.

---

## How the self-copy works

A page loaded from `file://` cannot `fetch` its own source, which is why most HTML quines resort to storing an escaped copy of themselves in a string. This one doesn't need to. The first statement in the script is:

```js
const DNA = "";
const SELF_SRC = "<!DOCTYPE html>\n" + document.documentElement.outerHTML;
```

The browser has already parsed the document, so it will happily hand the organism its own body — as long as you ask before touching the DOM. Reproduction is then just a string splice on `SELF_SRC`: find the genome locus, replace it with the child's genome, wrap it in a `Blob`, and download.

The whole trick rests on serialization being a **fixpoint**: the file, when parsed and re-serialized by the browser, must come out byte-identical, or each generation would accumulate drift until the lineage corrupted itself. This one does. See [Verification](#verification).

## The two-genome bug

The first working version was sterile, and the reason is too good not to document.

The splice looked for `const DNA = "..."` with this regex:

```js
/const DNA = "[^"]*";/
```

But the reproduction function itself contained the line that builds the replacement:

```js
return SELF_SRC.replace(RE, () => 'const DNA = "' + dna + '";');
```

Read that as plain text and the pattern matches it: `const DNA = "` followed by `' + dna + '` (no quotes in there) followed by `";`. The file had **two** genome loci, and `String.replace` took the first one. Every child was born with its parent's reproductive machinery overwritten by a genome, and no genome where the machinery used to be.

The fix is that the reproduction function never spells the key out. It assembles it at runtime, so it cannot match itself:

```js
const KEY = "const" + " DNA";
```

There is also a guard that counts loci before splicing and refuses to reproduce unless it finds exactly one:

```js
if (hits.length !== 1) throw new Error("expected 1 genome locus, found " + hits.length);
```

Sterility caused by the organism mistaking its own reproductive machinery for a gene. Make of that what you will.

## Genome format

The DNA string is one line of ASCII, quote-free so it can live inside a double-quoted JS literal:

```
version;generation;width;height;rule;mutationRate;id;parentId;lineage~<base64>
```

The payload after `~` is the grid: one bit per cell, packed into bytes row-major, run-length encoded as `(value, count)` pairs, then base64. A 96x64 grid is 6144 bits, and typical Life states are sparse and streaky, so the RLE earns its keep — genesis encodes to **1003 characters**.

`lineage` is a comma-separated ancestor log of `gen:population:id`, capped at 32 entries. Without the cap the genome grows without bound and the file bloats a few bytes every generation forever, which is a fun failure mode but a failure mode.

The identifier is a 24-bit FNV-style hash of the cell array and generation number, rendered as 6 hex digits. Two organisms with the same id at the same generation are the same organism.

## What you can do to it

| Control | Effect |
| --- | --- |
| **Spawn offspring** | Writes the child file. Fires automatically after a 3s countdown; any click or keypress cancels it. |
| **Re-roll mutations** | Recomputes this generation with a different random seed. |
| **Mutate rule** | Flips one digit in the B/S rulestring and **recomputes the current generation under the new physics**. The child inherits the new rule. `B3/S23` is only the starting condition. |
| **Mutation rate** | Per-cell probability of a random bit flip after each step. Heritable. Above roughly 0.005, lineages tend to go extinct fast. |
| **Preview forward / Adopt** | Run the simulation forward without committing, then adopt the result — the offspring is born that many generations older, with the jump recorded in its ancestry. |
| **Reseed soup** | Randomizes the grid. Resurrection for extinct lineages. |
| **Click the grid** | Toggle individual cells. Hand-edited genomes are inherited like any other. |

Everything except *Spawn* is a mutagen. Nothing is validated for "good" outcomes, because there aren't any — a rule mutation to `B0.../S...` will fill the grid with static, and that lineage is as alive as any other.

## Verification

The interesting failure mode is silent corruption over many generations, so the tests target that:

- **Codec round-trip** — genesis encodes and decodes to a bit-identical grid, including exotic rulestrings like `B36/S125678`.
- **40 save-and-open cycles**, splicing the real file text each time and re-decoding the result as a browser would. No locus drift, no genome corruption, file size stable at ~18 KB (the 32-entry lineage cap holding). Population drifted 643 → 387 over those 40 generations; the Gosper glider gun in the genesis state is what keeps it from flatlining.
- **Serializer fixpoint**, checked with `parse5` (the same HTML serialization spec `outerHTML` implements): the committed file is **byte-identical** to its own serialization — 0-byte delta. Script body preserved verbatim, no entity mangling. A child therefore differs from its parent in exactly one place: the DNA string.
- **Runtime**, under a DOM stub: full boot path plus every control handler exercised, and the locus guard confirmed to fail loudly rather than silently producing a sterile child.

Getting the seed file to be a perfect fixpoint took two fixes: `<html lang="en"><head>` with no newline between them (the parser discards whitespace in the "before head" insertion mode), and ending the file `</body></html>` with no trailing newline (trailing whitespace gets reparented into `<body>`). Without those the file still replicates fine, it just reaches its fixpoint at generation 1 instead of generation 0.

## Caveats

Chrome usually blocks programmatic downloads on `file://` without a user gesture, so the countdown may quietly do nothing — the **Spawn offspring** button always works. Served over HTTP (including the Pages demo) the auto-download generally goes through.

Each open produces one download. A lineage accumulates in your Downloads folder as `lifeform-g42-a3f1c2.html`. The `.gitignore` here excludes `lifeform-g*.html` so you don't commit an entire family tree by reflex.

## Running it

Open `index.html`. That's the whole procedure.

To host your own genesis, edit the `genesis()` function — the seed is a Gosper glider gun over a deterministic soup (`mulberry32(0x5eed1e)`, 24% density), so every fresh copy of the unmodified file starts life as the same organism. Change the seed and you found a different phylum.

## License

MIT
