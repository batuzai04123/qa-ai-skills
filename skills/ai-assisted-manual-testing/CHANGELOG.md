# Changelog — ai-assisted-manual-testing

All notable changes to this skill are recorded here.
Versions follow [Semantic Versioning](https://semver.org/): see
[CONTRIBUTING.md](../../CONTRIBUTING.md#updating-an-existing-skill) for what counts as a patch,
minor or breaking change.

## Unreleased

## 0.1.0 — 2026-10-07
### Added
- Initial version: step contract, capture rules, verdict rules, parallel sub-agent orchestration,
  and report output for Word, Google Docs, Excel, Google Sheets, PDF, HTML, Markdown,
  OpenDocument and wiki pages.

### Known issues
- Reference code (capture helpers, Word/Excel renderers, upload snippets) has not yet been
  validated end to end.
- Word renderer: image captions don't render in italics.
- Google Sheets: images in a converted `.xlsx` may not survive; verify after upload.
