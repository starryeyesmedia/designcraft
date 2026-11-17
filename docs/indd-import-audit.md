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

### Related anchored-object fix

Upstream issue [#555](https://github.com/storytold/designcraft/issues/555) is closed by merged PR [#562](https://github.com/storytold/designcraft/pull/562). It fixes a layout defect: when an anchored page item leaves an `AnchoredObjectSetting` attribute unset, the importer resolves that attribute from the item's applied object style chain and then the IDML defaults. The fix also corrects the direction of left-aligned X offsets. Current upstream main contains this fix.

The finding in this audit is separate, not a regression in #562: the style chain is available while importing an anchored item, but the object-style attributes themselves are not stored in `designcraft_doc::ObjectStyle` or written back by the IDML exporter. The sample shows loss of active style data after round-trip, including when items still have their own anchored settings. It does not show that #562's positioning fix is broken. A narrowly scoped issue report is drafted in [indd-anchored-style-loss-issue-draft.md](indd-anchored-style-loss-issue-draft.md).

At review time, open upstream PR [#2774](https://github.com/storytold/designcraft/pull/2774) changes `crates/idml/src/import/styles.rs` and `crates/idml/src/tests/object_styles.rs` to resolve fill, stroke, paragraph, text-frame, and corner categories through the based-on chain. It does not add anchored-object category preservation. It overlaps the same object-style import function and test file, so recheck its status and resolve the small textual overlap before integrating.

The separate #2920 correction is prepared on `codex/2920-anchored-object-style-settings` in the development fork, commit `b7a6f508`. Its synthetic regression failed against the original importer/exporter, then passed after the model/import/export correction; it verifies a changed setting survives export and re-import. The existing #562 item-level positioning tests also pass. The branch has not been submitted upstream.

## Follow-up report: restaurant-menu text and Overprint Preview (2026-11-17)

### Menu-specific report: awaiting diagnostic input

The report describes the first `T` in “TAZEWELL, VA • EST. 2002” displaced to the left, possible small-text positioning problems on later pages, and 23 preflight errors. The user confirmed the menu is open in their running DesignCraft instance for preview. This task session does not expose a desktop capture or control tool, and the task workspace has no menu source, exported IDML, preflight rows, or local control endpoint; the current execution host also has no visible DesignCraft process or GUI display. I therefore could not inspect the imported text, character runs, paragraph settings, frame geometry, assets, or the 23 individual issues, nor compare a matching INDD conversion against its source IDML. No cause is assigned to binary conversion, IDML parsing, composition, font substitution, or rendering.

`preflight.run` reports overset/unplaced stories, missing font styles and fonts, missing glyphs, missing or modified links, low-resolution raster images, RGB swatches in print documents, empty text frames, unsupported typography or graphic-layer options, and white/[Paper] overprint. The reported total cannot be classified without the issue rows and the private document. The file, source IDML if available, linked assets, and exact preflight output are needed for a reproducible comparison. Keep those materials in a private diagnostic location; none are added to this repository.

### Overprint Preview: controlled reproduction passes

The existing synthetic regression `designcraft_render::tests::overprint_preview_mixes_inks` draws a yellow rectangle under a magenta rectangle whose fill overprints. With regular Preview (`RenderOptions::overprint_preview = false`), magenta knocks out yellow. With Overprint Preview enabled, the inks combine to red. `black_overprints_only_with_the_preference` separately verifies 100% [Black] overprint follows the document preference. The View command toggles `ui.overprint_preview`, and the canvas passes that state to the renderer.

Both tests were included in the workspace test run of `cargo xtask ci`; the final run passed all nine quality-gate steps with the documented optional `craft-fonts` input. This rules out a general failure for the controlled overlapping-ink case. It does not reproduce or explain the menu report; that requires checking the actual overprint flags, inks, and preview mode in the private document. No separate Overprint Preview issue or code change is justified yet.

No converter or attribute mapping was changed for this follow-up. Keep the menu text-position and preflight investigation separate from the controlled overprint case and from #2920.

## Prioritized defects

1. **#2760 — page-item Nonprinting lost during INDD conversion (confirmed report, converter-side).** The issue reports that a page item marked Nonprinting in INDD opens with `nonprinting = false`, while direct IDML keeps it. The locked converter source has a `Nonprinting` mapping in `ITEM_DEFAULT_ATTRS` for `PageItemDefault`, but no corresponding page-item entry in `ITEM_ATTRS` (`indd-utils/src/idml/attrs.rs`). The DesignCraft importer, model, exporter, and print rendering already carry and honor the boolean. No permitted IDML evidence for a true page-item value was present in the 216 packages scanned, so the exact binary field must not be guessed.
2. **#2182 — master-guide coordinate mismatch (confirmed, converter-side).** One trusted pair proves a 515.9 pt location mismatch described above. The likely cause is the absent ruler-origin preference, but the current corpus evidence does not identify a binary signal for selecting `PageOrigin` over the converter's `SpreadOrigin` default. Do not hard-code a choice without new permitted evidence.
3. **#2920 — anchored object style settings (confirmed structural loss, now corrected on the development branch).** The fix adds an editable model field and preserves the category flag and raw settings through IDML import/export/re-import. No visual defect was established. The correction is committed on `codex/2920-anchored-object-style-settings`; the upstream PR remains intentionally unopened.
4. **Object-style inheritance is tracked in open upstream PR #2774.** Its current scope is fill, stroke, paragraph, text-frame, and corner settings and disabled-category boundaries. It does not cover anchored-object settings, though it touches the same import function and test file. This audit did not isolate an independent inheritance defect in those categories.
5. **Font substitution and the remaining visual effects need separate measurement.** The only same-save PDF comparison lacked Arial and Times New Roman. The selected trusted pairs also do not establish image-link fidelity, gradient rendering parity, or multiframe overflow and pagination against InDesign output.

## Validation run

- `cargo check --locked -p designcraft-cli` — passed.
- `cargo test --locked -p designcraft-idml` — passed (216 tests).
- `cargo build --locked -p designcraft-cli` — passed.
- `cargo fmt --all -- --check` — passed.
- `cargo clippy --locked -p designcraft-idml --all-targets -- -D warnings` — passed; Cargo printed existing workspace dependency and patch warnings.
- `tools/compare.py` — 12 selected pairs, 10 trusted; 99.55% value coverage on trusted pairs.
- `tools/render_compare.py` — one five-page same-save triple; converter routes pixel-identical on 5/5 pages, with fonts missing for the DesignCraft-to-InDesign comparison.
- Direct CLI IDML export and rendering comparisons were run for the paired INDD and IDML inputs described above.

This audit branch remains documentation-only. The separate #2920 implementation passed the full `cargo xtask ci` gate: fmt, PDF dependency check, workspace Clippy, tests, i18n report, doctests, assets, layers, and wasm. The initial run without `CRAFT_FONTS_DIR` hit one existing Arabic UI test failure because Noto Sans Arabic was not embedded; the final run used a temporary `craft-fonts` checkout (Noto Sans Arabic is OFL-1.1), and no font files were committed. The converter's Nonprinting and ruler-origin gaps still need permitted binary evidence before any mapping change.
