# Static export corruption — diagnosis

**Status: the committed export cannot be repaired. It must be rebuilt from the Next.js source.**

## Symptom

The site renders but is completely inert. React never hydrates, so every interactive
element does nothing: the "Watch Intro Reel" button, all five Work category filters,
the mobile hamburger menu, and the nav's scroll state. Chromium logs 19 page errors
on load, starting with `self.__next_f.push is not a function`.

## Root cause

A text transform was applied to the build output that matched pipe-delimited spans
and replaced each span — **including its contents** — with a spaced pipe (` | `).

The entry point fails first, in `index.html`:

    shipped:  (self.__next_f=self.__next_f | []).push([0])
    correct:  (self.__next_f=self.__next_f||[]).push([0])

`undefined | []` evaluates to `0`, so `self.__next_f` becomes the number `0` and every
subsequent `self.__next_f.push(...)` throws.

## Why this is not repairable by search-and-replace

Reversing ` | ` to `||` looks plausible but is wrong, because the transform **deleted
code**, it did not merely rewrite an operator. Proof, from the core-js percent-decoder
bundled into `_next/static/chunks/0cz1d0mv5g_q7.js`, checked against core-js 3.50.0
`internals/url-percent-coding.js`:

    upstream:  codePoint = (octets[0] & 0x0F) << 12 | (octets[1] & 0x3F) << 6 | (octets[2] & 0x3F)
    minified:  e=(15&t[0])<<12|(63&t[1])<<6|63&t[2]
    shipped:   e=(15&t[0])<<12 | 63&t[2]

The substring `|(63&t[1])<<6|` — 14 characters of real code — is gone. It cannot be
reconstructed from the shipped file. The same deletion hits `case 4` of the same switch.

Measured across the eight chunks with a JS lexer (see notes below):

| context | count | meaning |
|---|---|---|
| spaced ` \| ` in code | 1093 | corruption sites |
| tight `\|` in code | 175 | genuine bitwise ops (React lane/flag masks, UTF-8 packing) — untouched |
| pipes inside string / regex / template literals | 139 | must never be rewritten; 24 are inside regexes |
| `\|\|` remaining anywhere in the JS | **0** | against 3075 surviving `&&` |

Zero `||` alongside 3075 `&&` is not something a bundler emits. Every other operator
survived intact (`===`, `!==`, `++`, `>>`, `??`, `|=`, `&=`), so the transform targeted
pipes specifically. These files carry no spaces around any other binary operator
(` + `, ` && `, ` = `, ` < ` all occur zero times), which is why each space around a
pipe is itself a fingerprint of the tampering.

## Fix

Rebuild from the Next.js source and re-export, and do not run the transform over the
output. If the transform is a script that scrubs contact details, restrict it to text
nodes in `.html`/`.txt` and never let it touch `.js`.

If the source is unavailable, restore `_next/` plus `index.html` and `404.html` from an
earlier, un-transformed copy of the export.

## Content inventory

The site's own content survived the corruption intact. `recovered-content.json` in this
folder holds all 17 portfolio items across the 5 categories, plus the intro-reel video
id, extracted from the corrupted bundle. Use it to verify nothing is lost in the rebuild.

## Notes

Verified in Chromium against a local server: on the shipped export, clicking the
"Audition" filter leaves the Work grid unchanged at 3 cards.
