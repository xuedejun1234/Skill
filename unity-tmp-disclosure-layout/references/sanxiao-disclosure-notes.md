# SanXiao Disclosure Text: Project Notes

## Confirmed project setup

- Prefab: `Assets/Resources/Prefab/Pre.prefab`.
- Addict body: `Addict/Scroll View/Viewport/Content/Text`.
- Age body: `Age/Scroll View/Viewport/Content/Text`.
- Controller: `Assets/Scripts/Login/StartupDisclosureSequence.cs`.
- The prefab body labels were converted to `TextMeshProUGUI`; titles and button labels may remain legacy UI Text and should not be converted unless requested.
- The serialized body font reference in the prefab points to `Assets/Resources/TMP/DisclosureFont.asset`, generated from `SourceHanSerifCN-SemiBold-7.otf`. Other fonts also exist in the project; always inspect the live prefab reference rather than guessing from the Project window.
- The prior `StandardFont.asset` had a broken source-font reference/material shader. Regenerating a TMP font asset from a valid source font fixed the glyph source for these disclosure bodies. Do not reuse a broken TMP asset merely because its name looks generic.
- TMP Essential Resources/settings were imported before assigning the components.

These are SanXiao-specific observations, not requirements for all Unity projects.

## Layout behavior implemented for this case

The notice controller's `KeepNumbersTogether(GameObject panel)`:

- Finds the panel's `ScrollRect`, `Content`, and body `TextMeshProUGUI`.
- Normalizes CRLF/CR newlines, forces TMP layout, and reads `TMP_TextInfo` line starts.
- Detects wrapped numeric spans such as a number with punctuation and common Chinese currency/age/time units, then adds a safe line break before a span if TMP split it.
- Detects a rendered line that begins with Chinese or ASCII punctuation; moves that punctuation and any immediately preceding numeric span to the prior line when possible.
- Recomputes preferred text height, resizes the body/Content, and resets the scroll view to the top.
- Runs for the Addict notice when first shown and for the Age notice once when opened. Keep it one-time/idempotent so showing a panel does not accumulate line breaks.

The punctuation set used in the project includes Chinese comma/full stop/question/exclamation/semicolon/colon/enumeration comma and closing brackets/quotes, plus ASCII punctuation. Adapt the set to the actual notice text and TMP version. Opening brackets should not be moved as if they were closing punctuation.

## Issues encountered and handling

### Chinese punctuation appeared at the beginning of a wrapped line

TMP wrapping can place punctuation on the next visual line depending on font metrics and width. First inspect the actual wrapped output at the target canvas size. If TMP's leading-character rules support the exact characters and are not shared with unrelated UI, configure those. Otherwise use a controlled post-layout pass with `TMP_TextInfo`; insert explicit newlines from a stable source string, never by repeatedly mutating the already-adjusted text.

### The imported font asset showed missing glyphs or an error material

A font source file and a TMP font asset are different resources. Verify the TMP font asset's source font GUID and material shader. If either is invalid, create a new `TMP_FontAsset` from the valid imported `.otf/.ttf/.ttc`, select an appropriate atlas mode, and assign that asset to the body text. Verify every character in both notices, including Chinese punctuation, numerals, percent signs, and units. If using a dynamic atlas, ensure the source font remains included in the build and test a device build; for a static atlas, bake the required character set.

### Scrolling stopped working after the conversion

Keep `ScrollRect` on the same parent and preserve its viewport/content references. The body TMP rect must be tall enough for the full text; size `Content` to at least the larger of preferred body height and viewport height, then set `verticalNormalizedPosition` to `1` after layout. Do not add a second `ScrollRect` to the body text.

### Wrapped output or measurements were incorrect

Call `Canvas.ForceUpdateCanvases()` and `ForceMeshUpdate()` after activating the panel and before reading line info or preferred size. Ensure the text rect has a valid width first. Re-run mesh layout after each text change. Use `TMP_CharacterInfo.index` to map a TMP visible character back to the source string; guard against whitespace, rich-text tags, hidden characters, and out-of-range indexes.

### Number + unit split across lines

Keep the complete numeric expression and its attached unit together (for example, `16周岁`, `100元`, `20小时`). Do not alter the legal copy or renumber clauses. Preserve original paragraph separators and only insert a line break at a safe boundary before the expression if necessary.

## Validation checklist

- Both Addict and Age body nodes are TMP components with the intended font asset.
- Titles, confirm buttons, and unrelated `Text (Legacy)` nodes are unchanged.
- No visible line begins with a prohibited punctuation mark at the target portrait aspect ratio.
- Numeric quantities stay with their units; paragraph numbers and wording remain unchanged.
- Long text scrolls from top to bottom without clipping, and opening the panel repeatedly does not add more newlines.
- Unity compiles without TMP missing-reference or shader errors.
