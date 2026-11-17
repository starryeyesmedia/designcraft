# INDD import fidelity audit

> **Reviewed:** 2026-11-17 · **Scope:** an initial audit using a selected subset of permitted third-party reference files. This is not a full-corpus fidelity claim.

## Import path

`crates/engine/src/cmd/interchange.rs` detects `.indd` input, calls `craft_indd::convert`, forwards converter warnings, then sends the resulting IDML through `import_report_with_previews`. Direct `.idml` input goes through the same importer. Document fonts are resolved after import. The IDML item importer copies `PageItem::nonprinting()` into the document model (`crates/idml/src/import/item.rs`); IDML export writes the model value (`crates/idml/src/export/item.rs`). Page and PDF rendering omit nonprinting items.

## Measured baseline

No sample file or generated output is stored in this repository. The source files remain in the converter's ignored corpus directory.

The DesignCraft lockfile pins `indd-utils` at `7f3060e5bce78d21ea70e0d5eacb96edce15dfbe`. The official upstream `indd-utils` main checkout was also inspected at `136f973a9d5f1b3ef34e46e27f4a08af196efde7`; the only differences from the pinned revision are repository metadata and documentation, not converter source.

| Check | Result |
|---|---|
| Converter subset | 12 existing paired files were compared; 10 passed the repository's trust checks and 2 were stale. The 10 trusted pairs had 0 conversion failures, 0 converter warnings, exact text in 178/178 stories, and 113,994/114,513 matched IDML values (99.55%). The converter produced 753 additional values (0.66% of reference values). This subset is not the full corpus. |
| INDD versus direct IDML in DesignCraft | Three permitted pairs were opened through both routes. Two direct CLI render comparisons had identical page pixels; a five-page same-save comparison also found INDD-converted and direct-IDML renders identical on all 5 pages. This shows the paths agree on these examples; it does not show either matches InDesign. |
| Independent PDF rendering | For the five-page same-save triple, page dimensions matched at 792 × 612 pt. DesignCraft had 3 pages with line-break differences and 1 moved line versus the InDesign PDF. Font completeness was 0/1: Arial and Times New Roman were unavailable. These layout differences cannot be attributed to formatting import while the source fonts are missing. |
| Font fallback | In the layout pair, fontconfig selected OpenAI Sans for all seven referenced families. In the PDF comparison pair, it selected substitutes for Arial and Times New Roman. The CLI did not emit a useful missing-font warning. |
| Linked assets | One of the five directly compared pairs has 31 external Link records and no packaged image files. Those assets could not be assessed from the IDML package, so they are excluded from formatting-loss conclusions. A separate package had four links but its PDF failed same-save checks, so it was excluded from the rendering comparison. |
| Nonprinting | None of 216 permitted IDML packages downloaded for this audit contained `Nonprinting="true"`. The tested files therefore do not reproduce #2760. |
| Anchored settings | A paired, 82-story document has four object styles with `EnableAnchoredObjectOptions="true"`, used by five page items. Its source IDML has 49 `AnchoredObjectSetting` records; both DesignCraft import routes export 31, and none of the four active style flags remain. The 10 nondefault settings on actual anchored items survive. The sample proves style and preference data loss; it does not prove a visual difference on the current page. |

For the separate guide comparison, source IDML and its paired INDD differ at master-spread guide `u945`: source IDML has `Location="481.88976377955123"`, while the converter output and DesignCraft's INDD route have `997.7952755905512`. Direct IDML import retains `481.88976378`. The reference IDML says `RulerOrigin="PageOrigin"`; the INDD preference chunk 0x808 is absent, so the converter defaults to `SpreadOrigin`. `docs/format/preferences.md` records other same-version pairs with this missing-chunk ambiguity and no binary field found to distinguish the origin. This is a confirmed conversion mismatch, but a clean-room-compliant correction needs more evidence than the current default assumption.

### Attribute-level differences in the trusted converter subset

The 10 trusted pairs contain 500 missing and 19 mismatched raw IDML values. These counts include nonvisual metadata. In the sampled values, `AppliedFont`, font style, point size, leading, kerning, tracking, paragraph alignment and indents, tab lists, and page-item fill and stroke values match. The observed differences are:

| Area | Difference | Current assessment |
|---|---|---|
| Font resources | 241 `Font` values and 6 `FontFamily` values are absent from generated IDML. The sampled text-range and style formatting references listed above match. | Missing font registry metadata is separate from unavailable installed fonts; family-resolution fidelity needs a paired installed-font case. |
| Layout and guides | One guide `Location` differs; `ViewPreference.RulerOrigin` differs in the same document. Two pairs also differ in `DocumentPreference.ColumnDirection` and `MarginPreference.ColumnDirection`. | Guide coordinates are a confirmed document-geometry defect. Column-direction cases need a rendering comparison before assigning visual impact. |
| Paragraph formatting | Seven `ParagraphStyle.HyphenQuality` values are absent in one pair. | Potential line-break effect; no matching-font visual comparison isolates it yet. |
| Transparency | Twenty-five `OpacityGradientStop` values are absent in one pair. The converter omits item stops equal to effective object-style stops. | Could be inherited duplicate data; visual impact is unconfirmed. |
| Editing and text behavior | Four `TextVariable/MatchParagraphStylePreference.AppliedParagraphStyle` values differ; one pair differs in language quote settings and spelling vendor. | Future editing or automatic text behavior; no current-page visual difference established. |
| Other metadata | Remaining gaps include object export options, hyperlink accessibility labels, document metadata, and one cell/table style value. | Kept out of the visual-loss count unless a case shows a visible or editable consequence. |

This comparison tests binary conversion against paired IDML. It does not imply that DesignCraft retains every matching IDML value: the anchored object-style settings above are a separate loss in the DesignCraft model and occur on both import routes.

DesignCraft reassigns IDs during export. Raw value comparison against the original IDs therefore produces misleading gaps; the app comparisons above use text, object structure, guides, and rendering.

## Prioritized defects

1. **#2760 — page-item Nonprinting lost during INDD conversion (confirmed report, converter-side).** The issue reports that a page item marked Nonprinting in INDD opens with `nonprinting = false`, while direct IDML keeps it. The locked converter source has a `Nonprinting` mapping in `ITEM_DEFAULT_ATTRS` for `PageItemDefault`, but no corresponding page-item entry in `ITEM_ATTRS` (`indd-utils/src/idml/attrs.rs`). The DesignCraft importer, model, exporter, and print rendering already carry and honor the boolean. No permitted IDML evidence for a true page-item value was present in the 216 packages scanned, so the exact binary field must not be guessed.
2. **#2182 — master-guide coordinate mismatch (confirmed, converter-side).** One trusted pair proves a 515.9 pt location mismatch described above. The likely cause is the absent ruler-origin preference, but the current corpus evidence does not identify a binary signal for selecting `PageOrigin` over the converter's `SpreadOrigin` default. Do not hard-code a choice without new permitted evidence.
3. **Anchored object style settings are discarded by the DesignCraft IDML model (confirmed structural loss).** `crates/idml/src/import/styles.rs` builds object styles from fill, stroke, paragraph, text-frame, effects, and corner values; it does not retain `AnchoredObjectSetting`. The model consequently exports neither the four active style flags nor their settings. This affects future editing and style application; a visual defect needs a paired case with an active anchored item.
4. **Object-style inheritance is tracked in open upstream PR #2774.** That PR covers fill, stroke, paragraph, text-frame, and corner settings and disabled-category boundaries. Check its status before making overlapping changes. This audit did not isolate an independent inheritance defect.
5. **Font substitution and the remaining visual effects need separate measurement.** The only same-save PDF comparison lacked Arial and Times New Roman. The selected trusted pairs also do not establish image-link fidelity, gradient rendering parity, or multiframe overflow and pagination against InDesign output.

## Validation run

- `cargo check --locked -p designcraft-cli` — passed.
- `cargo test --locked -p designcraft-idml` — passed (216 tests).
- `cargo build --locked -p designcraft-cli` — passed.
- `tools/compare.py` — 12 selected pairs, 10 trusted; 99.55% value coverage on trusted pairs.
- `tools/render_compare.py` — one five-page same-save triple; converter routes pixel-identical on 5/5 pages, with fonts missing for the DesignCraft-to-InDesign comparison.
- Direct CLI IDML export and rendering comparisons were run for the paired INDD and IDML inputs described above.

No code or tests have been changed in this audit. `cargo xtask ci` has not been run. The next safe implementation candidate is the DesignCraft anchored-style preservation gap; the converter's Nonprinting and ruler-origin gaps need permitted format evidence before binary mappings change.
