# Technical Analysis: Quadruple-Context Namespace Pivot

## Parser State Machine Walkthrough

### Initial State
```
Insertion Mode: "in body"
Open Elements Stack: [html, body]
Active Formatting Elements: []
```

### Step 1: <table>
```
Insertion Mode: "in table"
Open Elements Stack: [html, body, table]
Foster Parenting: ACTIVE
```
The table element switches insertion mode to "in table". Any misplaced content
will be foster parented to the table's parent (body) before the table.

### Step 2: <tr><td>
```
Insertion Mode: "in cell"
Open Elements Stack: [html, body, table, tbody, tr, td]
```

### Step 3: <math>
```
Insertion Mode: "in body" (MathML integration point)
Open Elements Stack: [html, body, table, tbody, tr, td, math]
Foster Parenting: STILL ACTIVE (outer table context preserved)
```
MathML is an HTML integration point per HTML5 §13.2.6.1. The parser temporarily
switches to HTML rules, but the outer table context remains on the stack.

### Step 4: <mtext>
```
Insertion Mode: "in body"
Open Elements Stack: [..., math, mtext]
```
mtext is a MathML text integration point. HTML rules continue.

### Step 5: <table> (inside MathML)
```
Insertion Mode: "in table"
Open Elements Stack: [..., math, mtext, table]
Foster Parenting: ACTIVE (second layer)
```
Second table creates SECOND foster parenting layer. Content misplaced here
gets foster parented to the mtext, but the outer table's foster parenting
is still active for elements that pop back out.

### Step 6: <mglyph>
```
Insertion Mode: "in table"
Open Elements Stack: [..., math, mtext, table, mglyph]
```
mglyph is a void element in MathML but the parser may treat it differently
depending on whether it's in HTML or MathML context.

### Step 7: <style>
```
Insertion Mode: "in head" (style in table → foster parent to before table)
BUT: Inside MathML integration point, so parser hesitates.
```
This is where sanitizer divergence begins. The sanitizer sees:
- "We're in MathML/SVG context → style is safe (CSS only)"
But the browser sees:
- "We're in HTML integration point → style follows HTML rules → can break out"

### Step 8: <svg><foreignObject>
```
Insertion Mode: "in body" (SVG foreignObject is HTML integration point)
Open Elements Stack: [..., svg, foreignObject]
```
Third integration point! SVG foreignObject switches back to HTML rules.

### Step 9: <div xmlns="...">
```
Insertion Mode: "in body"
Open Elements Stack: [..., foreignObject, div]
```
Now we're in pure HTML land inside the foreignObject.

### Step 10: <p><b><p><button>
```
Open Elements Stack: [..., div, p, b, p, button]
Active Formatting Elements: [p, b, p, button]
```
These are **adoption agency scope markers**. The second <p> inside <b> triggers
the adoption agency algorithm (HTML5 §13.2.6.4.7).

### Step 11: <script>
```
The script is inside the button, which is inside the second p.
When the adoption agency runs (due to stack depth or explicit triggers),
the script may be re-parented OUT of the button → OUT of the foreignObject →
OUT of the SVG → and into the body, outside all sanitized containment.
```

## Why Sanitizers Fail

### DOMPurify
DOMPurify uses a two-pass approach:
1. Regex pre-scan for obvious XSS patterns
2. DOM-based sanitization with allowed tag/attribute lists

The regex pre-scan sees: `<table><math><mtext><table><mglyph><style><svg><foreignObject>`
It classifies this as "safe SVG/MathML with style" and allows it through.

The DOM sanitizer builds a tree, walks it, and removes disallowed elements.
But the tree it builds is NOT the tree the browser builds on re-parse.
Specifically:
- DOMPurify's parser doesn't implement MathML text integration points correctly
- It doesn't track foster parenting across integration point boundaries
- The adoption agency algorithm isn't triggered during sanitization

When the sanitized string is later inserted via `innerHTML`, the browser's parser:
1. Sees the integration points
2. Reconstructs the formatting element stack
3. Runs adoption agency on the deep <p><b><p><button> stack
4. Re-parents the <script> to body

Result: Sanitizer says "clean", browser says "execute".

### Server-Side Sanitizers (Python bleach, Ruby sanitize)
These use libxml2 or html5lib. The problems:
- libxml2 doesn't implement HTML5 integration points at all
- html5lib implements them but not the adoption agency algorithm fully
- Neither tracks foster parenting state across namespace boundaries

### Email Clients
Gmail/Outlook/Fastmail sanitize server-side before display.
The sanitized HTML is then displayed in a browser (or WebView).
The server-side sanitizer strips what it thinks is dangerous.
But the browser re-parses the sanitized output, triggering the mutation.

## Spec References

- HTML5 §13.2.6.1: Integration points
- HTML5 §13.2.6.4.7: Adoption agency algorithm
- HTML5 §13.2.6.4.9: Foster parenting
- HTML5 §13.2.6.4: The "in table" insertion mode
- WHATWG HTML Living Standard: MathML/SVG integration

## Mitigation

For defenders:
1. **Serialize and re-parse** — After sanitization, serialize to string and
   re-parse with the SAME parser that will display the content. If the tree
   changes between sanitization and display, the sanitizer failed.

2. **Use CSP** — `script-src 'none'` prevents execution even if parser bypass
   injects script elements. But note: this doesn't stop other vectors like
   `javascript:` URIs or custom element gadgets.

3. **iframe sandbox** — Display user content in sandboxed iframes with
   `sandbox="allow-same-origin"` (but NOT `allow-scripts`).

4. **Parser-aligned sanitizers** — Use a sanitizer that uses the browser's
   actual parser (like browser-native `setHTML` with sanitizer API, though
   support is limited).

## Detection

To detect if this bypass works against a target:
1. Inject the QCTX payload
2. Check if `<script>` element exists in document.body (not in foreignObject)
3. If yes, the adoption agency re-parented it → bypass successful
4. Check if script executed (CSP may block execution even if injection works)

```javascript
// Detection script
const scripts = document.querySelectorAll('body > script');
const bypass = scripts.length > 0 && scripts[0].textContent.includes('alert');
```
