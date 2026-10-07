# Architecture artwork

- `principle.png`: README and social preview.
- `principle.svg`: scalable version with embedded fonts.
- `principle.excalidraw`: editable diagram.
- `principle.spec.json`: source-grounded layout and content specification.

The diagram follows `plugins/prompt-translator/hooks/register.ts`. It shows the
submission hook, protected fragments, external translation API, restoration and
validation, failure stop, and delivery to Claude with reply-language context.
The wait card is a UI note; it does not enter the model context.

Diagram content uses this repository's MIT license. Embedded Excalifont and
Noto Sans SC fonts use SIL Open Font License 1.1; the corresponding notices are
included here. The output was rendered as a static paper-style sketch.
