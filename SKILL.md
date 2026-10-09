---
name: docgen
description: Convert documents with the DocGen API — HTML or a web page to PDF, Markdown to HTML, HTML to EPUB, MOBI or AZW3 (Kindle) — saving the result as a local file. Use when the user wants a PDF, an e-book or a Kindle file generated from HTML, Markdown or a URL.
---

# DocGen

DocGen (https://docgen.novtopro.com) is a synchronous conversion API: each call returns the finished file (the Artifact) in the response body. Nothing is stored server-side, so always save the response with `curl -o`.

## API key

Read the key from the `DOCGEN_API_KEY` environment variable and send it as `Authorization: Bearer $DOCGEN_API_KEY` (or `X-Api-Key`). Keys start with `dk_live_`.

When `DOCGEN_API_KEY` is unset or empty, stop and ask the user to create a key at https://docgen.novtopro.com/dashboard (sign-in is with Google, so only a human can do it) and export it as `DOCGEN_API_KEY`. Continue once they have.

## Calls

Every conversion is `POST https://docgen.novtopro.com/api/v1/<format>` with a JSON body. Build the body with a JSON encoder (`jq` below) so quotes and newlines in the source are escaped; `-d @-` reads it from stdin. `-w '%{http_code}\n'` prints the status: on `200` the `-o` file is the Artifact; on anything else it holds a JSON error — handle it (see Errors) and delete the file.

PDF from HTML (optional `css`, `title`):

```sh
jq -n --rawfile html invoice.html '{html: $html}' |
  curl -sS https://docgen.novtopro.com/api/v1/pdf \
    -H "Authorization: Bearer $DOCGEN_API_KEY" -H 'Content-Type: application/json' \
    -d @- -o invoice.pdf -w '%{http_code}\n'
```

PDF from a public web page — send `url` instead of `html`, never both (http(s) only; localhost, private addresses and redirects are rejected):

```sh
curl -sS https://docgen.novtopro.com/api/v1/pdf \
  -H "Authorization: Bearer $DOCGEN_API_KEY" -H 'Content-Type: application/json' \
  -d '{"url": "https://example.com/report"}' -o report.pdf -w '%{http_code}\n'
```

PDF options:

- `engine`: `tiger` (default; light, for simple invoices and receipts), `lion` (paged documents, CJK), `wolf` (print-grade CSS paging). The free plan allows only `tiger`; send `lion` or `wolf` only when the user asks for them.
- `pageSize` (`letter` default, `a4`, `legal`, `tabloid`), `orientation` (`portrait`, `landscape`), `margins` (points, one number or `{top, right, bottom, left}`) apply to Tiger only. On Lion and Wolf, set the page with CSS `@page { size: A4; margin: 20mm }`.
- No engine runs JavaScript: send the final HTML.

Markdown (GitHub-flavored) to a complete HTML document (optional `css`):

```sh
jq -n --rawfile markdown notes.md '{markdown: $markdown}' |
  curl -sS https://docgen.novtopro.com/api/v1/markdown \
    -H "Authorization: Bearer $DOCGEN_API_KEY" -H 'Content-Type: application/json' \
    -d @- -o notes.html -w '%{http_code}\n'
```

E-books from HTML: `title` is required; `author`, `css` and `cover` (image bytes, base64) are optional. The same body works for all three formats:

```sh
jq -n --rawfile html book.html --arg title 'My Book' --arg author 'Ann Lee' \
    --arg cover "$(base64 < cover.jpg | tr -d '\n')" \
    '{html: $html, title: $title, author: $author, cover: $cover}' > book.json

curl -sS https://docgen.novtopro.com/api/v1/epub \
  -H "Authorization: Bearer $DOCGEN_API_KEY" -H 'Content-Type: application/json' \
  -d @book.json -o book.epub -w '%{http_code}\n'

curl -sS https://docgen.novtopro.com/api/v1/mobi \
  -H "Authorization: Bearer $DOCGEN_API_KEY" -H 'Content-Type: application/json' \
  -d @book.json -o book.mobi -w '%{http_code}\n'

curl -sS https://docgen.novtopro.com/api/v1/azw3 \
  -H "Authorization: Bearer $DOCGEN_API_KEY" -H 'Content-Type: application/json' \
  -d @book.json -o book.azw3 -w '%{http_code}\n'
```

E-books and PDFs take HTML, so Markdown goes through `/api/v1/markdown` first — chain the calls, one Conversion each (there is no one-step endpoint). Markdown to EPUB:

```sh
jq -n --rawfile markdown book.md '{markdown: $markdown}' |
  curl -sS https://docgen.novtopro.com/api/v1/markdown \
    -H "Authorization: Bearer $DOCGEN_API_KEY" -H 'Content-Type: application/json' \
    -d @- -o book.html -w '%{http_code}\n'

jq -n --rawfile html book.html --arg title 'My Book' '{html: $html, title: $title}' |
  curl -sS https://docgen.novtopro.com/api/v1/epub \
    -H "Authorization: Bearer $DOCGEN_API_KEY" -H 'Content-Type: application/json' \
    -d @- -o book.epub -w '%{http_code}\n'
```

Plan, limits and this month's usage (`GET`, not metered):

```sh
curl -sS https://docgen.novtopro.com/api/v1/me -H "Authorization: Bearer $DOCGEN_API_KEY"
```

## Limits

- free: 50 conversions a month, 256 KB request body, 5 requests a minute, Tiger only.
- pro: unlimited conversions, 4 MB, 120 requests a minute, Tiger, Lion and Wolf.

Every successful conversion counts, Markdown → HTML included; failures don't.

## Errors

Errors are JSON, `{"error": "<code>"}`. Act on the code:

- `invalid_json`, `html_required`, `markdown_required`, `title_required`, `ambiguous_source` (both `html` and `url`), `url_invalid` (400): fix the request body and retry.
- `url_not_allowed` (400), `url_fetch_failed` (502): the page is local/private, or couldn't be fetched (timeout, non-2xx, redirect, non-HTML). Fetch the HTML yourself and send it as `html`, or tell the user.
- `missing_api_key`, `invalid_api_key` (401): the key is unset, unknown or revoked. Ask the user for a valid key from https://docgen.novtopro.com/dashboard.
- `account_canceled` (403): the subscription is canceled. Tell the user; retrying won't help.
- `engine_not_allowed` (403): the plan doesn't include the requested engine. Tell the user and let them choose: upgrade to pro, or rerun with `tiger` themselves. Keep the engine they asked for until they decide.
- `body_too_large` (413): the body (or the fetched page) exceeds the plan's limit. Shrink it — compress or downscale images, split the document — or tell the user.
- `rate_limit_exceeded` (429): wait the number of seconds in the `Retry-After` response header (add `-D headers.txt` to the curl to capture it), then retry.
- `quota_exceeded` (429): this month's conversions are used up; `Retry-After` is the seconds until the next UTC month. Tell the user rather than waiting.
- `conversion_failed` (502): no file was produced (e.g. an invalid base64 `cover`). Check the input; tell the user if it persists.

## Reference

- OpenAPI 3.1: https://docgen.novtopro.com/openapi.json
- Full API docs as plain text: https://docgen.novtopro.com/llms-full.txt
- Docs for humans: https://docgen.novtopro.com/docs
