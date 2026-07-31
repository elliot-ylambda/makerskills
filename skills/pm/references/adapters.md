# PM adapters

How `pm` reads and writes each board. A board's `tool` value selects an adapter.

## Hosted Agent transport rule

Shell networking and user-held provider credentials are unavailable. Use only
the reviewed typed actions below. If the current session does not advertise the
required action, return `execution_unavailable`; never fall back to a provider
CLI, SDK, raw GraphQL, or direct HTTP request.

## Notion

Use `magister_integration_read` and `magister_integration_change` with
`service=notion`, following the `magister-notion` skill. The Gateway owns the
connection and exact approval.

Common shapes:

```http
# List cards by column (semantic read)
POST [service=notion]/databases/<db-id>/query
  JSON_BODY {"filter":{"property":"Status","select":{"equals":"In Progress"}}}

# Read one card
GET [service=notion]/pages/<page-id>

# Move a card (exact change approval)
PATCH [service=notion]/pages/<page-id>
  JSON_BODY {"properties":{"Status":{"select":{"name":"Ready"}}}}

# Create a card (exact change approval)
POST [service=notion]/pages
  JSON_BODY {"parent":{"database_id":"<db-id>"},"properties":{"Name":{"title":[{"text":{"content":"<title>"}}]},"Status":{"select":{"name":"Backlog"}}}}
```

Typical properties are `Name`, `Status`, `Priority`, `Owner`, and `Size`;
`boards.md` may override the status names.

## GitHub

Use `magister_integration_read` and `magister_integration_change` with
`service=github`, following the `magister-github` skill. Do not invoke the
networked `gh` client from sandboxed shell.

- Issues board: list issues with `GET repos/<owner>/<repo>/issues`; create with
  `POST repos/<owner>/<repo>/issues`; update labels/state with the documented
  issue route.
- Projects v2 requires GraphQL operations not represented by the current
  generic GitHub REST allowlist. Return `execution_unavailable` for Projects v2
  rather than guessing a route.

## Plane

`execution_unavailable`: there is no reviewed Hosted Agent Plane action. Ask
the user for an export or operate on a local board copy. Never request a Plane
API key.

## Linear

Use a Linear MCP/action only when it is explicitly advertised by the current
session. Otherwise return `execution_unavailable` and ask for an export. Never
request a Linear key or construct direct GraphQL traffic.

## Obsidian

Local Markdown boards are supported without network access. Columns are
`## Column Name` headings and cards are checkbox bullets:

```markdown
## Backlog
- [ ] Card A

## Ready
- [ ] Card B

## Done/Archived
- [x] Card C
```

- Read the exact board path from `boards.md`.
- Move a card by preserving its full text/metadata and relocating it.
- Create a card beneath the requested heading.
- Mark done with `[x]` and move it to the configured done section.

Preserve YAML/card metadata and unrelated board content.
