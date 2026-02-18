# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Obsidian Export Image is an Obsidian plugin that exports notes as images (PNG, JPG, WebP, PDF). It supports watermarks, author info, metadata display, image splitting, batch folder export, and 20+ locales.

## Build & Dev Commands

```bash
npm run dev          # Development build with esbuild (watch mode disabled, inline sourcemaps)
npm run build        # Production build (tsc type-check + esbuild, minified, no sourcemaps)
npm run lint         # Lint with XO (wraps ESLint)
npm run lint-fix     # Auto-fix lint issues
npm run typesafe-i18n  # Regenerate i18n type definitions
```

Build output: `main.js` (CJS bundle), `main.js.LEGAL.txt`

Release artifacts (committed): `main.js`, `manifest.json`, `styles.css`

## Architecture

### Entry Point

`main.ts` re-exports `src/ExportImagePlugin.ts`, which extends Obsidian's `Plugin` class. The plugin registers:
- Two commands: export full note, export selection
- File menu items (right-click on files/folders)
- Editor context menu items
- Settings tab via `SettingRenderer`

### Export Pipeline

1. **Markdown rendering** - Obsidian's `MarkdownRenderer.render()` converts markdown to DOM
2. **Async renderer wait** - `waitForAsyncRenderers()` in `src/utils/index.ts` uses MutationObserver to wait for async code block processors (Dataview, SQLSeal) to stabilize
3. **React composition** - `Target` component (`src/components/common/Target.tsx`) wraps rendered content with watermark (`@pansy/react-watermark`), author info, metadata, and padding
4. **DOM-to-image capture** - `src/dom-to-image-more.js` (vendored fork) converts the composed DOM to a blob
5. **Output** - `src/utils/capture.ts` handles save (via `file-saver`), copy to clipboard, PDF generation (`jspdf`), and multi-file ZIP (`jszip`)

### Two Export Modes

- **File export** (`src/components/file/exportImage.tsx`) - Opens a modal with live WYSIWYG preview (`ModalContent.tsx`), settings form, zoom/pan (`react-zoom-pan-pinch`), and save/copy buttons. Also supports "quick export" (selection to clipboard without modal).
- **Folder export** (`src/components/folder/`) - Batch exports all markdown files in a folder, with progress tracking.

### Image Splitting

`src/utils/split.ts` implements four split modes:
- `none` - Single image
- `fixed` - Fixed pixel height with configurable overlap
- `hr` - Split at horizontal rules
- `auto` - Split at paragraph boundaries, targeting a height threshold

### Settings System

Two parallel config definitions exist:
- `src/formConfig.ts` - Declarative `SettingItem[]` for the Obsidian settings tab, rendered by `SettingRenderer.ts`
- `src/components/file/ModalContent.tsx` - Inline `FormSchema<ISettings>` for the export preview modal

Both use dot-path notation (e.g., `watermark.text.content`) to read/write nested `ISettings` fields.

Settings types are declared globally in `src/type.d.ts` (ambient declarations, no imports needed).

### i18n

Uses `typesafe-i18n` with type-safe translations. Base locale is English (`src/i18n/en/index.ts`). The `src/L.ts` module detects Obsidian's locale and exports a typed translation function `L`.

To add translations: create a new locale folder under `src/i18n/`, add translations, then run `npm run typesafe-i18n` to regenerate types.

Auto-generated i18n files (`src/i18n/*.ts` at root level) are excluded from linting.

### Vendored Library

`src/dom-to-image-more.js` is a vendored/modified copy of `dom-to-image-more`. It is excluded from TypeScript compilation and linting. Types come from `@types/dom-to-image`.

## Key Types

All in `src/type.d.ts` as ambient declarations:
- `ISettings` - Plugin settings (width, format, padding, watermark, author info, split config)
- `FileFormat` - `'png0' | 'png1' | 'jpg' | 'pdf' | 'webp'` (png0 = opaque, png1 = transparent)
- `SplitMode` - `'none' | 'fixed' | 'hr' | 'auto'`
- `ResolutionMode` - `'1x' | '2x' | '3x' | '4x'`
- `FormSchema<T>` / `FieldSchema<T>` - Form configuration types

## Linting

XO is the primary linter (configured in `.xo-config.js`). It wraps ESLint with strict defaults. Key overrides:
- `@typescript-eslint/ban-ts-comment: off` - Some `@ts-ignore` usage exists for Obsidian API gaps
- `no-await-in-loop: off` - Sequential async operations are intentional (image processing)
- Uses spaces (not tabs)
- Ignores: `src/dom-to-image-more.js`, `main.js`, `src/i18n/*.ts`

## Obsidian Plugin Conventions

- `manifest.json` defines plugin metadata and minimum Obsidian version (1.4.0)
- `versions.json` maps plugin versions to minimum Obsidian versions
- `version-bump.mjs` syncs version across `package.json`, `manifest.json`, and `versions.json`
- The plugin uses React 19 for UI (rendered into Obsidian modals via `createRoot`)
- External modules (`obsidian`, `electron`, `@codemirror/*`, `@lezer/*`, Node builtins) are excluded from the bundle
