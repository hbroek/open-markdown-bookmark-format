# The Open Markdown Bookmark Format

Version 1.0 — 2026-11-23

A bookmark file in this format is plain [CommonMark](https://commonmark.org/)
that reads well on its own, in any markdown viewer, editor, or static-site
generator — not just in a bookmark tool. There is no custom syntax: a folder
is a bold list item, a bookmark is a link, and everything else is ordinary
markdown.

```markdown
# Bookmarks

- **Dev**
  - **Web**
    - [Go](<https://go.dev/>) — The Go site
      Keywords: lang, go
- [Top level](<https://top.test/>)
```

This repository is the format's home: a vendor-neutral specification that
any application can read and write, plus an [extension mechanism](#4-extensions)
for data that's specific to one tool and not part of the portable core.

## Origin and implementations

The core format (§§1–3, 5–6 below) originated in
[mardo](https://github.com/hbroek/mardo), a markdown-native bookmark manager —
this spec documents mardo's format so other tools can read and write it
interoperably, not a new format competing with it.

Known implementations:

- [mardo](https://github.com/hbroek/mardo) — the format's origin.
- [LinkaGoGo](https://www.linkagogo.com) — reads and writes the core format,
  plus an optional extension carrying its own ratings, favorites, visit
  history, reminders, aliases, smart URLs, and RSS associations. See
  [§4.1](#41-worked-example-linkagogos-extension) and LinkaGoGo's own
  [format page](https://www.linkagogo.com/bookmarks-markdown.md) for details.

Using this format in your own tool? Open a pull request adding it to this
list.

## 1. Document shape

- The file is a single nested bullet list. Nesting is 2 spaces per level.
- An optional single `# Title` heading at the very top of the file is the
  *document's* title, not a folder — `# Bookmarks` in the example above. A
  writer should include it; a reader must not treat it as a folder named
  "Bookmarks".
- Everything else in the list is either a **folder** or a **bookmark**.

## 2. Folders

A folder is a list item whose text is bold:

```markdown
- **Folder Name**
```

Its children — subfolders and bookmarks — are the list items nested one level
deeper, in any order a writer chooses. A reader must accept either order
(subfolders before or after a folder's own bookmarks).

## 3. Bookmarks

A bookmark is a list item that starts with a markdown link:

```markdown
- [Title](<https://example.com/>) — Optional description
    Keywords: one, two, three
```

- **Title and URL**: a standard markdown inline link, `[Title](URL)`. The URL
  is wrapped in angle brackets (`<...>`) so it survives unescaped if it
  contains a space, pipe, or other character markdown would otherwise treat
  as the end of the link destination. A bare autolink (`<https://example.com/>`
  or even a plain `https://example.com/` with no brackets) is also accepted
  as a bookmark with no title — the URL stands in for the title.
- **Description**: optional text after the link, separated by an em dash
  (`—`), en dash (`–`), hyphen (`-`), or colon (`:`). A writer should use an
  em dash; a reader must accept any of the four.
- **Keywords**: an optional line directly below the bookmark, indented one
  level deeper than the bullet, reading `Keywords: ` followed by a
  comma-separated list. A reader that does not recognize a continuation line
  (this one, or any extension field from §4) should simply ignore it rather
  than erroring — the bookmark itself is still valid without it.

## 4. Extensions

Everything in §§1–3 is the *portable core* — what every conforming reader is
expected to understand, and the only thing a writer should rely on when
targeting an unknown reader. A tool may add more data to a bookmark as
additional continuation lines, in the same `Label: value` shape as
`Keywords:`, for whatever it needs that isn't part of the core: ratings,
visit history, reminders, sync metadata, whatever is specific to that tool.

**The mechanism is deliberately free-form.** A continuation line is just
`Label: value`, indented one level deeper than its bookmark's bullet. There
is no registry of reserved labels and no required namespace prefix — a
reader that doesn't recognize a label simply doesn't match it against
anything it knows and drops it, the same way it drops any other line it
doesn't understand. This keeps a single-purpose tool's writer simple (no
prefixing boilerplate) at the cost of a small chance two unrelated tools
pick the same label for different things; since unrecognized labels are
always ignored rather than misapplied, a collision only ever means a value
goes unread, never that it's misread. A tool introducing its own extension
should document its labels publicly (as LinkaGoGo does below) so other
implementers can recognize and interoperate with them if they choose to.

A file using extension fields is still valid, portable markdown, and still
opens correctly in any reader, including one that has no notion of the
extension — that reader just won't preserve the extended data on a
subsequent re-export, since it never modeled it in the first place. Writing
extension fields is always optional; a writer that only implements §§1–3 is
fully conforming.

### 4.1. Worked example: LinkaGoGo's extension

[LinkaGoGo](https://www.linkagogo.com) is a bookmark manager whose richer
[XBEL export](https://www.linkagogo.com/xbel-linkagogo.dtd) already carries
ratings, favorite status, visit counts, reminders, aliases, smart URLs, and
RSS feed associations — data the portable core intentionally leaves out.
Rather than invent a second format for round-tripping that data as plain
text, LinkaGoGo extends this one:

```markdown
- [Go](<https://go.dev/>) — The Go site
    Keywords: lang, go
    Added: 2020-01-15T10:00:00Z
    Visited: 2021-06-15T12:00:00Z
    Visits: 3
    Rating: 4
    Favorite: yes
    Alias: godev
    Smart URL: <https://pkg.go.dev/search?q=%s>
    RSS: <https://go.dev/blog/feed.atom>
    Reminder: daily
```

| Label | Value |
|---|---|
| `Added` | When the bookmark was created, as an RFC 3339 timestamp (UTC). |
| `Visited` | When it was last visited, as an RFC 3339 timestamp (UTC). |
| `Visits` | Total visit count, a positive integer. |
| `Rating` | 1–5. |
| `Favorite` | `yes` if favorited; the line is omitted otherwise. |
| `Alias` | A short keyword shortcut for the bookmark, escaped as in §5. |
| `Smart URL` | A URL template (e.g. containing `%s`), wrapped in `<...>` like the bookmark's own URL. |
| `RSS` | The feed URL associated with the bookmark, wrapped in `<...>` the same way. |
| `Reminder` | `daily`, a weekday name, an ordinal day of the month (`1st`…`31st`), or a plain number of days. |

This is LinkaGoGo's own extension, not part of the portable core — a reader
that doesn't know these labels, mardo included, opens the file correctly and
simply doesn't carry this metadata through on a subsequent re-export. See
LinkaGoGo's own [format page](https://www.linkagogo.com/bookmarks-markdown.md)
for how it's produced and consumed.

## 5. Escaping

Markdown's own special characters — `` \ ` * _ [ ] < > & ~ | `` — are
backslash-escaped wherever they appear in a title, folder name, description,
or keyword, so that arbitrary bookmark data never breaks the list structure.
A URL additionally percent-encodes `<`, `>`, space, and newline, since those
cannot appear literally inside the `<...>` link-destination brackets.
Percent-encoded sequences are left as-is on import — they're ordinary URI
escaping, not markdown escaping, and must not be decoded.

## 6. Reading a hand-written list (not just a generated one)

A writer following §§1–3 and 5 produces files a reader can parse with a
simple, literal reading of this spec. But a reader should be more forgiving,
so that a list someone wrote by hand — not generated by any tool — imports
cleanly too:

- Any heading, `#` through `######`, is also a folder, nested by heading
  level. (As in §1, a lone `# Title` at the top of an otherwise heading-free
  document is the exception — that one is the document's title.)
- A list item with no link is also a folder if it is bold (`**Name**` or
  `__Name__`) **or** if it has nested list items under it. A plain, non-bold,
  childless line of text is not a folder — it is simply not part of the
  bookmark structure, and a reader should skip it rather than erroring.
- Fenced code blocks (` ``` ` or `~~~`) are skipped entirely, so a list that
  happens to include an example snippet elsewhere in the same document isn't
  misread as bookmarks.

## Contributing

This repository is the spec, not a reference implementation — it has no
code of its own. Contributions welcome:

- Fixes or clarifications to the spec text: open an issue or pull request.
- A new implementation: open a pull request adding it to the
  [implementations list](#origin-and-implementations) above.
- A new extension, in the style of §4.1: open a pull request adding a
  worked-example section documenting your tool's labels, so other
  implementers can recognize them.

## License

Public domain — [CC0 1.0](LICENSE). Use it, implement it, extend it, no
attribution required.
