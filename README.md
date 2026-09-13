# HTML5 Parser Mutation Trap Arsenal v2.0

## Overview
Comprehensive test harness and standalone payloads for HTML5 parser mutation traps,
namespace pivots, and sanitizer bypass techniques.

## Files

| File | Description |
|------|-------------|
| `harness.html` | Main interactive test harness with 10 payloads, isolated execution frames, DOM viewer, and naive sanitizer simulation |
| `payload_qctx.html` | Quadruple-Context Namespace Pivot — Table→MathML→Table→SVG→HTML |
| `payload_template_resurrection.html` | Template .content DocumentFragment deferred execution |
| `payload_isindex.html` | isindex formaction auto-wrap implicit form generation |
| `payload_stack_exhaustion.html` | Recursive foreignObject compositing layer exhaustion |
| `payload_ns_rebind.html` | Namespace prefix rebinding XHTML→MathML desync |
| `payload_ce_gadget.html` | Custom element upgrade gadget with data-x eval |
| `payload_style_cdata.html` | Style block CDATA/comment state confusion |
| `payload_menu_dialog.html` | menu+dialog+menuitem deep nesting scoping bypass |

## Kill Chains Covered

1. **Adoption Agency Scope Marker Overflow** — Deep formatting stacks trigger re-parenting
2. **Template Content Resurrection** — cloneNode(true) resurrects sanitized fragments
3. **isindex Formaction Auto-Wrap** — Obsolete element generates implicit executable form
4. **Recursive foreignObject Stack Exhaustion** — Layer limit DoS with potential state leakage
5. **Namespace Prefix Rebinding** — XML sanitizer vs HTML5 tree builder desync
6. **Custom Element Upgrade Gadgets** — Sanitizer-agnostic deferred execution
7. **Comment/CDATA State Confusion** — Serializer vs parser disagreement on style blocks
8. **menu+dialog+menuitem** — Obsolete elements with legacy parser rules

## Usage

Open `harness.html` in a browser. Each payload card has:
- **Test in Isolated Frame** — Execute in sandboxed iframe
- **Copy HTML** — Copy raw payload to clipboard
- **Show Parsed DOM** — View browser's parsed DOM tree
- **Test vs DOMPurify** — Run through naive sanitizer simulation

## WASM Integration

Click the "WASM: OFF" toggle to load Pyodide. Enables:
- Persistent state across page reloads
- Python-based payload generation and analysis
- Cross-reference with existing fuzzer datasets

## Targets

- DOMPurify / bleach
- Email clients (Gmail, Outlook, Fastmail)
- WYSIWYG editors (CKEditor, TinyMCE)
- Markdown renderers (GitHub, Reddit)
- Electron apps
- PDF renderers (Chrome headless, PDF.js)
- SSRF avatar/icon fetchers
- Browser extension content scripts

---
Generated for research purposes. Use responsibly.
