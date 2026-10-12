# action_text.cr

A 1:1 Crystal port of **actiontext 8.1.4** (the Rails rich-text editing
component, namespace `ActionText`). Implements the `has_rich_text`
attribute macro with autosave, the `action_text_rich_texts` store, and
the `ActionText::Content` pipeline: HTML fragment
parsing/serialization, plain-text conversion, safe-list sanitization
(safe list taken verbatim from rails-html-sanitizer 1.7.1 plus the
action-text attachment tags) and the `trix-content` layout render —
byte-identical to the gem. Suite: **48/48 oracle** cases recorded from
actiontext 8.1.4 and replayed without edits.

> ⚠️ EXPERIMENTAL FRAMEWORK, DO NOT USE IN PRODUCTION YET.

## Installation

```yaml
dependencies:
  action_text_cr:
    github: OrelSokolov/action_text.cr
```

```sh
shards install
```

Depends on `active_record_cr` and the `lexbor` shard.

## Usage

```crystal
require "action_text_cr"

class Message < ActiveRecord::Base
  has_rich_text :body
end

message = Message.create!(body: "<h1>Hello</h1><div>World<a href='x'>l</a></div>")
message.body.to_s           # "<div class=\"trix-content\">\n  ...\n</div>\n"
message.body.to_plain_text  # block/br/quote/list rules, chomp("") semantics
message.body.to_body_content.links  # ["x"]
message.body?               # true (non-blank plain text)
```

## Feature comparison vs actiontext 8.1.4

### Rich text attribute & persistence

| Feature | Original (actiontext) | Port (action_text.cr) | Status |
|---|---|---|---|
| `has_rich_text :name` — getter / setter / presence | ✅ | Macro generating `body` / `body=` / `body?` | ✅ |
| Autosave of the RichText row with its owner | ✅ | `after_create` / `after_update` callbacks | ✅ |
| `create!` / `create` accepting the rich-text key in the attribute hash | ✅ | Key pulled out before AR create runs | ✅ |
| `action_text_rich_texts` store (name/body/record_type/record_id) | ✅ | `RichText` row class over `Base.connection` | ⚠️ partial (hand-rolled, not a full AR model) |
| `with_rich_text_*` scope / eager loading | ✅ | ❌ | ❌ |

### Content pipeline & rendering

| Feature | Original | Port | Status |
|---|---|---|---|
| `Content#to_s` — `trix-content` layout render | ✅ | Static layout: `<div class="trix-content">\n  …\n</div>\n` | ✅ |
| `Content#to_plain_text` | ✅ | Bottom-up rules 1:1 (block/br/div/figcaption/blockquote quotes/ul-ol-li with indentation, `chomp("")`) | ✅ |
| `Content#to_html`, `#links` | ✅ | Same (lexbor serialization 1:1 with Nokogiri HTML5) | ✅ |
| Serialization (`Content.load` / `.dump`) | ✅ | Same | ✅ |
| Sanitization (Rails::HTML4 safe-list + attachment tags, unwrap-keep-content) | ✅ | Safe list verbatim from rails-html-sanitizer 1.7.1 | ✅ |
| `Content#to_markdown` | ✅ | ❌ | ❌ (P2) |
| `RichText#to_editor_html` / Trix conversion | ✅ | ❌ | ❌ (P1) |

### Attachments & views

| Feature | Original | Port | Status |
|---|---|---|---|
| `<action-text-attachment sgid=…>` resolution (sgid → allowed classes) | ✅ | ❌ | ❌ (P1) |
| ActiveStorage blob attachments in content, attachment galleries | ✅ | ❌ | ❌ (P1/P2) |
| Custom render partial `app/views/action_text/contents/_content` | ✅ | ❌ | ❌ (P1) |
| `content.html` direct render | ✅ | ❌ | ❌ |
| `encrypted: true` rich text | ✅ | ❌ | ❌ (P2) |
| Trix editor | client-side JS | Not ported — browser JS works as-is | ✅ (n/a) |

## Deviations & Crystal-specific notes

- `RichText` is a hand-rolled row class on `ActiveRecord::Base.connection`,
  not a full ActiveRecord model — the AR codegen would override `#to_s` /
  `#body` with attribute accessors (later defs win in Crystal).
- `RichText#body` is the raw HTML string exactly as stored in the DB;
  get the `Content` object via `to_body_content`.
- HTML fragment transformations run on lexbor instead of Nokogiri;
  serialization output is 1:1 with the Nokogiri HTML5 oracle.
- The sanitizer reproduces the libxml2 `\n` artifact only for the
  unwrap of a trailing script/style child (the oracle cases cover this).

Full list with rationale: `PLAN.md` (Russian).

## Testing

```sh
make spec    # crystal spec --link-flags="-lm"; lexbor needs libm at link time
```

Requires a local PostgreSQL (override with `AT_TEST_URL`). Oracle:
48 cases recorded with the real actiontext 8.1.4
(`oracle/cases.rb` → `oracle/expected.yml`, SQLite in-memory on the
Ruby side); the Crystal spec `spec/oracle_spec.cr` replays them 1:1
without edits — **48/48**.
