---
id: platforms-environment-text-encoding-and-normalization
domain: platforms
category: environment
applies_to: [general]
confidence: verified
sources:
  - https://www.unicode.org/reports/tr15/
  - https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/APFS_Guide/FAQ/FAQ.html
  - https://pubs.opengroup.org/onlinepubs/9699919799/utilities/grep.html
last_verified: 2026-07-30
related: [platforms-environment-timezone-and-locale, platforms-tools-bsd-vs-gnu-cli, testing-quality-document-verification-gates]
---

# Matching Non-ASCII Text When the Same String Has Several Binary Forms

## When this applies

You are grepping, comparing, deduplicating, or key-ing on text that contains
non-ASCII characters (Korean/Japanese/Chinese, accented Latin, emoji) — in file
contents, filenames, or user input — and a match that should hit returns nothing.

## Do this

The same visible string can have more than one binary representation, and equality
is byte equality unless you normalize first. Decide both sides explicitly:

| Case | Do |
|------|----|
| Matching a substring of syllabic text (Hangul, and any precomposed script) | Match the syllables exactly as written in the target. `아니` is not a substring of `아닌` in composed form: `아닌` is U+C544 U+B2CC and `아니` is U+C544 U+B2C8 — the second syllable is a different code point, not a prefix plus a letter |
| The stem's ending inflects (`아닌`/`아니다`/`아니라`) | Enumerate the alternatives explicitly (`아닌\|아니다\|아니라`); a stem prefix that works in alphabetic scripts silently matches zero here |
| Comparing or matching two strings from different origins | Normalize both to the same form (NFC for storage and wire formats) before comparing; a pattern in NFC never matches text stored in NFD and vice versa |
| Storing or emitting text your own checks will later match | Normalize on input to one form for the whole system, so pattern and target agree by construction |
| Building map keys, dedup sets, or diffs from user or filesystem text | Normalize before use; unnormalized keys produce two entries that render identically |
| A pattern must run on both macOS and Linux userlands | Regex-feature differences (`grep -P`, `\b`) → [platforms-tools-bsd-vs-gnu-cli] |

Reproduce the failure before theorizing: print the code points of both sides
(`python3 -c "print([hex(ord(c)) for c in open(f).read()])"`). A zero-hit grep on
text that is visibly present is a form mismatch until the code points say otherwise.

## Edge cases

| Case | Then |
|------|------|
| Filenames on macOS | Finder writes NFD, a shell writes NFC. APFS is *normalization-insensitive for lookups* — either form opens the file — but it "preserves the normalization of the filename", so `ls`/`readdir` returns the stored form and a grep over that listing can miss it. Normalize the listing before matching |
| Text arrives from an HFS+ era archive or `readdir` and must go to a Linux/Windows system | Convert with `iconv -f UTF-8-MAC -t UTF-8` (or normalize to NFC in code) before writing it out; the two systems disagree on the stored form, not on the text |
| Match must be normalization-agnostic and you cannot control the input | Normalize the input stream into the matcher (`python3`/`perl` filter emitting NFC) instead of widening the regex; alternations over both forms grow past review |
| Comparing for equality where the scripts differ in case rules too | Locale-sensitive casing is a separate hidden input → [platforms-environment-timezone-and-locale] |
| Byte sequences that are not valid characters in the current locale (arbitrary pathnames) | Set `LC_CTYPE`/`LC_COLLATE` to C/POSIX for that command — POSIX leaves the behavior undefined otherwise |

## Instead of

| If you are about to | Do this instead | Why |
|---------------------|-----------------|-----|
| Write `grep '대상이 아니'` to catch every inflection of a Korean stem | Match the exact syllables and enumerate the inflected alternatives | The composed syllable `닌` does not contain `니`, so the stem pattern returns 0 hits on text that plainly contains the phrase |
| Conclude "the tool's regex engine is broken with multibyte input" after a zero-hit grep | Print the code points of the pattern and the target first | The usual cause is a normalization or syllable mismatch, which looks identical on screen |
| Widen a failing pattern until it matches | Normalize both sides to one form, then keep the exact pattern | A pattern widened to absorb form differences also matches things the check was meant to reject |

## Sources

- https://www.unicode.org/reports/tr15/ — UAX #15: canonical decomposition/composition (Hangul handled algorithmically); "when implementations keep strings in a normalized form, they can be assured that equivalent strings have a unique binary representation"; higher-level processes that compare strings must respect canonical equivalence; normalize on input to the whole system
- https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/APFS_Guide/FAQ/FAQ.html — "APFS preserves the normalization of the filename and uses hashes of the normalized form of the filename to provide normalization insensitivity", whereas "HFS+ stores the normalized form of the filename on disk"
- https://pubs.opengroup.org/onlinepubs/9699919799/utilities/grep.html — `LC_CTYPE` determines interpretation of byte sequences as single- vs multibyte characters; set it to POSIX/C for pathnames, whose byte sequences may not form valid characters in some locales
