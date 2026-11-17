# Draft upstream issue report: IDML object styles lose anchored-object settings

**Title:** IDML import/export drops anchored-object settings stored on object styles

## Description

An object style with `EnableAnchoredObjectOptions="true"` and an `AnchoredObjectSetting` loses that category and its settings when a document is opened and exported through DesignCraft. The loss occurs for direct IDML import and for INDD documents after conversion to IDML, because both routes use the same DesignCraft IDML importer.

This is distinct from [#555](https://github.com/storytold/designcraft/issues/555), fixed by [#562](https://github.com/storytold/designcraft/pull/562). That fix uses an applied object's style chain to resolve unset attributes when calculating the anchored object's position. The style's own settings still are not retained in the DesignCraft object-style model or written back during IDML export.

## Reproduction and evidence

An initial audit using permitted existing paired files found four object styles with active anchored-object options applied to five page items. The source IDML contains 49 `AnchoredObjectSetting` records; DesignCraft's export contains 31. The active style flags and their style-level settings disappear on both import routes, while the nondefault settings on actual anchored items survive. No source files are attached or named.

A synthetic IDML fixture can reproduce this without an INDD binary: define an object style with `EnableAnchoredObjectOptions="true"` and explicit `AnchoredObjectSetting` attributes, apply it to an anchored item, import and export the document, then check the style data. The fixture should also change one anchored-style value in the document model and verify the changed value is exported and survives re-import.

## Expected behavior

The object style retains its anchored-object category and settings in the editable document model. Exporting the document preserves those settings, and editing a setting changes the exported style value.

## Actual behavior

The style importer reads fill, stroke, paragraph, text-frame, effects, and corner settings but does not retain `AnchoredObjectSetting`. The object-style exporter writes no anchored-object category or settings. This is a structural/editability loss; the audit does not claim a current-page visual difference.
