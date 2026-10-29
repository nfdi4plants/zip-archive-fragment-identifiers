# URI Fragment Identifiers for ZIP Archives

This document proposes a fragment selector syntax for identifying **files** and **directories (folders)** inside ZIP archives. It follows the structure and `key=value` style of the [spreadsheet fragment identifier draft](https://github.com/nfdi4plants/spreadsheet-fragment-identifiers). The goal is to let tools reference a data file or a folder within an archive using a single URI.

```url
https://example.com/dataset.zip#file=data/measurements.csv
https://example.com/dataset.zip#directory=data/
```

This is a **working draft**, not an established standard. The [IANA registration for `application/zip`](https://www.iana.org/assignments/media-types/application/zip) does not define fragment identifiers. [RFC 6839, Section 3.6](https://www.rfc-editor.org/rfc/rfc6839.html#section-3.6) also records that no fragment syntax was defined for `application/zip` at its publication. This proposal does not update either registration.

---

## 1. Introduction

### 1.1 Background: How ZIP Archives Work

A ZIP archive packages entries in one resource. Each file has a name, metadata, and a payload that may be compressed or stored without compression. Local headers accompany the entries; a **central directory** near the end indexes them and records where they can be found. This permits tools to locate an entry without decompressing every file. See the [PKWARE ZIP specification](https://pkware.cachefly.net/webdocs/casestudies/APPNOTE.TXT), especially Section 4.3.

Names such as `data/measurements.csv` express a folder hierarchy. This draft treats folders as logical paths: `data/` exists if an entry has that prefix, even without a separate folder entry. An explicit directory entry, with a name ending in `/`, can also represent an empty folder.

For example, an archive might contain:

```text
dataset.zip
├── README.txt
├── data/
│   ├── measurements.csv
│   └── raw/
│       └── run1.tsv
└── empty/
```

Here, `data/` and `data/raw/` can be inferred from file names; `empty/` requires an explicit entry.

Under [RFC 3986, Section 3.5](https://www.rfc-editor.org/rfc/rfc3986.html#section-3.5), fragment semantics depend on the representation's media type. The fragment is separated before dereferencing the resource. For an HTTP URI, the client retrieves the archive without sending the fragment to the server, then interprets the selector. A fragment does not itself provide server-side extraction or guarantee a partial download.

### 1.2 Goals

- Enable programmatic linking to an individual **file**, a **directory subtree**, or the **archive root**.
- Keep selectors readable and consistent with the spreadsheet identifier draft.
- Resolve paths predictably across operating systems and ZIP implementations.
- Distinguish a missing target from an invalid or ambiguous reference.

### 1.3 Scope

- Applies to ZIP representations using the media type `application/zip`, including ZIP64 archives supported by the reader.
- Defines identification, not an extraction API, a directory listing format, or a new archive format.
- Does not automatically apply to ZIP-based formats with their own media types, such as XLSX, ODS, EPUB, or other `+zip` types. Those formats need an explicit adoption or composition rule; their existing fragment semantics remain authoritative.
- The core draft selects one file or one directory. Selection inside a file, nested archives, wildcards, and multiple independent targets are future extensions.

The uppercase terms **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, **SHOULD NOT**, and **MAY** express requirements for implementations claiming conformance to this draft, following [BCP 14](https://www.rfc-editor.org/rfc/rfc8174.html).

---

## 2. Fragment Selector Types

This draft uses **one target selector per fragment**:

| Selector | Target |
| --- | --- |
| `file=<path>` | One non-directory archive entry at the exact path |
| `directory=<path>/` | A logical directory and its complete subtree |
| `directory=/` | The logical archive root and its complete subtree |

Paths are relative to the archive root. The root selector is the only use of a leading `/`. Selector keys are lowercase and case-sensitive.

### 2.1 File

Selects one entry by its complete archive-relative path, including its filename.

```url
https://example.com/dataset.zip#file=README.txt
https://example.com/dataset.zip#file=data/measurements.csv
https://example.com/dataset.zip#file=data/raw/run1.tsv
```

The target is the stored member's logical content after any required decryption and decompression, not its compressed byte range or local header. Selecting a zero-byte file is valid. A file path MUST NOT end in `/`.

### 2.2 Directory

Selects a logical directory, including all descendant files and directories at every depth.

```url
https://example.com/dataset.zip#directory=data/
https://example.com/dataset.zip#directory=data/raw/
https://example.com/dataset.zip#directory=empty/
```

- The trailing `/` is REQUIRED and establishes a path boundary: `data/` does not select `database/` or a file named `data`.
- A directory exists if there is an explicit entry with that name **or** at least one entry below that path.
- An explicit empty directory is a valid target; a nonexistent directory is not.
- Selection includes an explicit directory entry when present. Inferred directories have no stored payload or directory metadata.

The result is a logical subtree. This draft does not prescribe its display order, extraction layout, or serialization as a newly generated archive.

### 2.3 Archive Root

Selects the logical root directory, including every entry:

```url
https://example.com/dataset.zip#directory=/
```

The root exists even in an empty archive and does not require an entry named `/`. A URI with no fragment, or with an empty fragment (`dataset.zip#`), instead refers to the ZIP resource as a whole; it does not request a directory view.

> **No selector chaining in this draft.** `#file=data.csv&directory=data/` and `#file=book.xlsx&sheet=Data` are invalid. A literal `&` in a filename must be encoded as `%26`.

---

## 3. Syntax

The nonempty fragment consists of exactly one `key=value` selector. The grammar describes the fragment **after** the URI's `#`; `#` is not part of the grammar.

### 3.1 ABNF

The grammar uses [RFC 5234 ABNF](https://www.rfc-editor.org/rfc/rfc5234.html), including its `ALPHA`, `DIGIT`, and `HEXDIG` core rules, and the case-sensitive `%s` strings from [RFC 7405](https://www.rfc-editor.org/rfc/rfc7405.html).

```text
fragment            = file-selector / directory-selector

file-selector       = %s"file=" relative-path
directory-selector  = %s"directory=" ( "/" / relative-path "/" )

relative-path       = segment *( "/" segment )
segment             = 1*( unreserved / pct-encoded / name-delim )

unreserved          = ALPHA / DIGIT / "-" / "." / "_" / "~"
pct-encoded         = "%" HEXDIG HEXDIG
name-delim          = "!" / "$" / "'" / "(" / ")" / "*" / "+"
                    / "," / ";" / "=" / ":" / "@"
```

The decoded-path constraints below are also normative; satisfying the ABNF alone is insufficient.

### 3.2 Encoding and Path Rules

1. Parse the selector key and the first `=` before decoding the value. Further `=` characters are part of the name. Raw `&` and additional raw `#` characters are invalid.
2. Use literal `/` characters as path separators. Percent-decode each segment **exactly once**, interpreting its bytes as UTF-8. Malformed escapes and invalid UTF-8 are invalid selectors. `%2F` and `%2f` are invalid inside a segment; an encoded slash MUST NOT introduce a new path boundary.
3. Characters outside the grammar MUST be UTF-8 percent-encoded. Examples include space (`%20`), `&` (`%26`), `#` (`%23`), `%` (`%25`), and `?` (`%3F`). Hexadecimal digits in escapes are case-insensitive. `+` means a literal plus sign, never a space.
4. Decoded segments MUST NOT be empty, `.` or `..`, or contain `/`, `\`, U+0000-U+001F, or U+007F. Paths MUST NOT begin with `/`, except for `directory=/`, or with a drive-letter prefix such as `C:`. Repeated separators such as `data//run.csv` are invalid. These restrictions also apply when prohibited characters are percent-encoded.
5. Match the decoded path **case-sensitively by Unicode code point**, without Unicode normalization, case folding, dot-segment removal, or other filesystem normalization. `Data.csv` and `data.csv` are different names. Visually identical Unicode spellings may also be different names.
6. All name characters are literal. There are no wildcard, glob, regular-expression, or comma-separated-list semantics. For example, `#file=data/*.csv` selects only an entry literally named `data/*.csv`.

Producers SHOULD leave unreserved characters and separator slashes literal and use uppercase hexadecimal digits in percent-encodings. Equivalent encodings of allowed name characters, such as `A` and `%41`, identify the same name.

### 3.3 Encoding Examples

| Archive entry name | Fragment |
| --- | --- |
| `data/run 1.csv` | `#file=data/run%201.csv` |
| `data/a&b.csv` | `#file=data/a%26b.csv` |
| `data/result#1.csv` | `#file=data/result%231.csv` |
| `data/100%.csv` | `#file=data/100%25.csv` |
| `data/a+b.csv` | `#file=data/a+b.csv` |
| `data/café.csv` | `#file=data/caf%C3%A9.csv` |
| `data/literal%20.csv` | `#file=data/literal%2520.csv` |

The final example selects a filename containing the literal characters `%20`, not a space. ZIP entry names themselves are never URI-percent-decoded.

---

## 4. Processing Rules

### 4.1 Reading Archive Names

Resolvers MUST use the archive's **central directory** as the entry index. This draft chooses the following deterministic name-decoding profile, based on the mechanisms in [PKWARE APPNOTE, Section 4.6.9 and Appendix D](https://pkware.cachefly.net/webdocs/casestudies/APPNOTE.TXT):

1. If general-purpose flag bit 11 is set, decode the filename as UTF-8.
2. Otherwise, use a version 1 Unicode Path extra field (`0x7075`) if its filename CRC matches and its name is valid UTF-8.
3. Otherwise, decode the filename bytes as IBM Code Page 437 (CP437).

Invalid UTF-8 in a bit-11 filename is an archive error. Multiple otherwise usable Unicode Path extra fields with different names, or a usable Unicode Path extra field that disagrees with a bit-11 UTF-8 filename, also constitute an archive error. An unusable Unicode Path extra field is ignored in step 2. Resolvers MUST NOT silently substitute the machine's current locale. Archives produced using other unmarked legacy encodings may therefore have different names from those intended by their producer.

Decoded entry names MUST satisfy the structural path constraints in Section 3.2, item 4, allowing one final `/` for directory entries. ZIP entry names are not URI-encoded values and MUST NOT be percent-decoded; literal spaces, `%`, `&`, and Unicode characters are permitted. The root sentinel `/` is not a valid stored entry name.

This draft classifies names ending in `/` as directories. Explicit entry-type metadata that contradicts this classification is an unsupported archive condition: for example, a directory-marked name without a trailing `/`, or an explicitly regular-file or symbolic-link type on a name ending in `/`. Missing type metadata or an unset directory flag alone is not a contradiction.

Archives containing names outside this profile are outside the core draft: a resolver MUST report that limitation rather than repair names or silently omit entries to produce an apparently complete result.

### 4.2 Resolution

1. Separate the fragment from the archive URI. If absent or empty, use the complete archive resource.
2. Validate the selector syntax and decoded path.
3. Read the central directory and decode its names using Section 4.1.
4. Resolve the target:
   - **File:** match the exact path against non-directory entries. Zero matches means **no match**; one match succeeds; multiple matches mean **ambiguous target**.
   - **Directory:** resolve its explicit entry or infer it from descendants. Select the directory and every entry whose name begins with its complete path, including the trailing `/`.
   - **Root:** select the logical root and all entries.
5. If a directory or root selection contains multiple entries with the same decoded full name, report **ambiguous target**. A resolver MUST NOT silently choose an entry, merge payloads, or discard duplicates. Duplicate names outside the selected subtree do not make that target ambiguous.

A file named `data` and a directory `data/` are different targets and can coexist in this model. Inferring `data/` from its descendants does not add a duplicate of an explicit `data/` entry.

Resolution does not require writing files to disk. Resolvers MUST NOT follow symbolic links while matching paths or expand a link into its target. A selected symbolic-link entry identifies the entry itself, including its stored link data, not the referenced filesystem object.

Identifying a member and reading its content are separate operations. A reader may identify an encrypted entry by name but be unable to read it. If central-directory metadata is inaccessible, the reader cannot establish a match. Unsupported compression, missing decryption credentials, unavailable archive parts, or integrity failures MUST be reported as access/processing limitations, not as evidence that a named entry is absent. Readers MUST NOT present inconsistent or corrupt entry data as a successfully read target.

### 4.3 Errors

| Condition | Result |
| --- | --- |
| Unknown key, repeated selector, or selector chaining with `&` | Invalid/unsupported selector |
| Malformed percent escape, invalid UTF-8, or prohibited path | Invalid selector |
| `#file=`, `#directory=`, `#file=data/`, or `#directory=data` | Invalid selector |
| `#file=/data.csv`, `#file=../data.csv`, or `#file=data/%2E%2E/x.csv` | Invalid selector |
| `#file=data%2Frun.csv` | Invalid selector: encoded path separator |
| Valid file or directory selector with no matching target | No match |
| Duplicate decoded entry names within the selected target | Ambiguous target |
| Unreadable, malformed, or unsupported archive/name metadata | Archive/access error |
| Target identified, but its payload cannot be read or verified | Content/access error |

The result labels describe semantic outcomes; an API's exact error codes are outside this draft.

### 4.4 Fallback

- Tools unaware of this proposal may open or download the whole archive; support is not assumed in ordinary browsers or archive tools.
- A conforming resolver MUST NOT report the complete archive, a different entry, or a partial subtree as a successful resolution of a failed selector.
- A user interface MAY offer to open the whole archive after an error, provided it makes the unresolved selection clear.

---

## 5. Security and Privacy

- A fragment identifies content; it does not grant access. Sharing an archive URI may expose the entire archive to anyone already able to retrieve it, regardless of which file the fragment selects.
- Filenames in fragments can reveal sensitive information through copied links, browser history, or application logs, even though an HTTP request does not include the fragment.
- Implementations SHOULD limit archive size, entry count, decompressed size, and processing time to handle malicious or excessively compressed archives.
- Selection does not imply extraction or execution. If an application extracts selected content, it MUST independently enforce destination containment, handle platform-specific filename collisions, and prevent path traversal and symbolic-link escapes. Selector validation alone is not sufficient for safe extraction.
- Readers MUST bind any payload they return to the selected central-directory entry and reject inconsistent archive structures rather than switch to a different local-header entry.
- These references identify paths in a particular archive representation. Replacing the archive or changing a file at the same path can change the referenced data; this draft provides no content-authenticity or immutability guarantee.

---

## 6. Future Extensions and Discussion

The following are **design questions, not supported syntax** in this draft:

| Topic | Question for a later revision |
| --- | --- |
| Selection within a member | How should a ZIP file selector compose with a spreadsheet sheet/range selector, a CSV row/column selector, or another media type's fragment syntax? |
| Directory depth | Is selecting the complete subtree sufficient, or is a direct-children-only mode needed? |
| Multiple targets | Should one fragment select several exact paths, a filename pattern, or a filtered set of entries? |
| Nested archives | Should a reference be able to traverse an inner ZIP before selecting its member? |
| Duplicate names | Is an explicit disambiguator needed, such as an entry position, despite its sensitivity to archive rewriting? |
| Stable data references | Should references optionally include an archive version or content digest to detect changed data? |
| Legacy filenames | Is an explicit profile needed for names outside the UTF-8/Unicode-extra-field/CP437 rules? |
| Adoption by ZIP-based formats | Which media types should explicitly adopt or compose these selectors, and how should conflicts with their own fragment syntax be resolved? |

The suggested next discussion is **selection within an archived data file**, especially composition with the spreadsheet draft. A URI has a single fragment component; a second raw `#` is not a composition mechanism. Any extension will need an explicit delimiter and encoding rule, plus a way to determine the selected member's media type.

---

## 7. Status

**Draft 0.1 — 2026-10-29.** This is an internal working proposal for open source tooling, intended for discussion and implementation feedback. It is not an IETF standard or an IANA-registered fragment syntax. The selector names and semantics remain open to revision.
