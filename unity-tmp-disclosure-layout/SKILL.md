---
name: unity-tmp-disclosure-layout
description: Convert Unity Chinese disclosure and age-notice body text to TextMeshPro while preserving UGUI scrolling and preventing Chinese punctuation from starting wrapped lines.
---

# Unity TMP Disclosure Layout

Use for Unity legal, age-rating, anti-addiction, or long Chinese notice text that must use TextMeshPro without punctuation at the start of a visual line. Keep the existing wording, prefab hierarchy, scroll behavior, and user-specified dimensions unless asked to change them.

## Workflow

1. Inspect the exact scene/prefab and identify the body text separately from titles, buttons, and legacy label components. Record its parent `ScrollRect`, `Viewport`, and `Content` references before editing.
2. Check the imported source font, TMP font asset, material/shader, and TMP Essential Resources. Assign a valid TMP font asset to each target `TextMeshProUGUI`; do not confuse a source `.ttf/.otf/.ttc`, TMP font `.asset`, and generated material. Verify that all Chinese glyphs in the text exist or can be populated by the font asset.
3. Preserve the scroll-view structure. After replacing the body label, set wrapping and overflow intentionally, recompute preferred height after TMP layout, resize `Content` as needed, and set the scroll position to the top.
4. Prevent punctuation at line starts by checking the rendered TMP line breaks after `ForceMeshUpdate()`. Prefer the TMP line-breaking rules when they can meet the requirement without changing unrelated text. If custom line breaks are required, insert them only at safe boundaries, preserve numbered paragraphs, and keep numbers together with their units.
5. Make the layout pass idempotent: start from the original text, avoid repeatedly inserting newlines on every show, stop when no violating wrap remains, and guard against zero-width or not-yet-laid-out text rects.
6. Apply changes to the prefab asset so they survive scene reloads. Confirm the final component type, assigned font, and serialized text in the prefab; compile through Unity and inspect the rendered notice at the target aspect ratio and scroll position.

For the SanXiao-specific component paths, imported font history, and line-wrap edge cases, read [references/sanxiao-disclosure-notes.md](references/sanxiao-disclosure-notes.md).
