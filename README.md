# DocGen agent skill

An [Agent Skill](https://skills.sh) that teaches coding agents to call the [DocGen](https://docgen.novtopro.com) API with `curl`: HTML or a web page to PDF, Markdown to HTML, HTML to EPUB, MOBI and AZW3 (Kindle), saving each result as a local file.

## Install

```sh
npx skills add 3cats/docgen-skill
```

Or copy [`SKILL.md`](SKILL.md) into your agent's skills directory (e.g. `.claude/skills/docgen/SKILL.md`). The same file is served at https://docgen.novtopro.com/SKILL.md.

## Setup

Create an API key at https://docgen.novtopro.com/dashboard and export it where your agent runs:

```sh
export DOCGEN_API_KEY=dk_live_...
```

The examples use `curl` and `jq`.

## More

- API docs: https://docgen.novtopro.com/docs
- OpenAPI 3.1: https://docgen.novtopro.com/openapi.json
- Full docs as plain text: https://docgen.novtopro.com/llms-full.txt
