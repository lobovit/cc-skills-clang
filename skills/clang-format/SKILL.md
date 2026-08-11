---
name: clang-format
description: "C/Clang code formatting with clang-format — configuration, .clang-format files, style options, editor integration, and CI enforcement. Use when setting up clang-format for a project, creating or modifying .clang-format configuration, analyzing existing code style, or troubleshooting formatting behavior. Covers LLVM, Google, Chromium, Mozilla, WebKit styles and custom configurations."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents. Requires clang-format.
metadata:
  author: lobor
  version: "1.0.0"
  openclaw:
    emoji: "✨"
    homepage: https://github.com/lobor/cc-skills-clang
    requires:
      bins:
        - clang-format
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(clang-format:*) Bash(git:*) Agent
---

**Persona:** You are a C formatting engineer. You configure clang-format to match existing project style, not to impose a new one — consistency with the codebase matters more than personal preference.

# clang-format Configuration

## Quick Start

```bash
# Format a single file
clang-format -i src.c

# Format all C files in directory
find . -name '*.c' -o -name '*.h' | xargs clang-format -i

# Dry run — show what would change
clang-format --dry-run src.c

# Output formatted version without modifying
clang-format src.c

# Dump default config
clang-format --dump-config
```

## Style Presets

```bash
# Use a built-in style
clang-format --style=llvm -i src.c
clang-format --style=google -i src.c
clang-format --style=chromium -i src.c
clang-format --style=mozilla -i src.c
clang-format --style=webkit -i src.c
```

| Style | Indent | Braces | Line Length | Best For |
| --- | --- | --- | --- | --- |
| LLVM | 2 spaces | Attach | 80 | LLVM/Clang projects |
| Google | 2 spaces | Attach | 80 | Google-style C |
| Chromium | 2 spaces | Attach | 80 | Chromium-style |
| Mozilla | 2 spaces | Attach | 80 | Mozilla projects |
| WebKit | 4 spaces | Attach | 80 | WebKit projects |

## .clang-format File

Place in project root. Example:

```yaml
BasedOnStyle: LLVM
IndentWidth: 4
ColumnLimit: 100
UseTab: Never
TabWidth: 4

# Braces
BreakBeforeBraces: Attach
AllowShortFunctionsOnASingleLine: Empty
AllowShortIfStatementsOnASingleLine: false
AllowShortLoopsOnASingleLine: false

# Alignment
AlignAfterOpenBracket: Align
AlignConsecutiveMacros: true
AlignEscapedNewlines: Left
AlignOperands: Align
AlignTrailingComments: true

# Includes
SortIncludes: CaseSensitive
IncludeBlocks: Preserve

# Pointer alignment
PointerAlignment: Right    # char *ptr
DerivePointerAlignment: false

# Spacing
SpaceAfterCStyleCast: false
SpaceBeforeParens: ControlStatements
SpacesInParentheses: false

# Indentation
IndentCaseLabels: true
IndentPPDirectives: None
NamespaceIndentation: None

# Other
ReflowComments: true
AllowAllParametersOfDeclarationOnNextLine: true
BinPackArguments: true
BinPackParameters: true
```

## Analyze Existing Code Style

Before applying clang-format, check if the project already has a consistent style:

```bash
# Check for existing .clang-format
ls -la .clang-format

# Check common indentation patterns
grep -rn "^\t" src/*.c | head -5   # tabs
grep -rn "^    " src/*.c | head -5  # 4 spaces
grep -rn "^  " src/*.c | head -5    # 2 spaces

# Check brace style
grep -A1 "if\|for\|while" src/*.c | head -20
```

## Git Integration

```bash
# Pre-commit hook — format staged files
#!/bin/bash
# .git/hooks/pre-commit
files=$(git diff --cached --name-only --diff-filter=ACM | grep -E '\.(c|h)$')
if [ -n "$files" ]; then
    echo "$files" | xargs clang-format -i
    git add $files
fi
```

## CI Enforcement

```yaml
# GitHub Actions
- name: Check formatting
  run: |
    find . -name '*.c' -o -name '*.h' | \
      xargs clang-format --dry-run --Werror
```

## Selective Formatting

```cpp
// clang-format off
// Custom-formatted code preserved
const int matrix[4][4] = {
    {1, 0, 0, 0},
    {0, 1, 0, 0},
    {0, 0, 1, 0},
    {0, 0, 0, 1},
};
// clang-format on
```

## Key Options Reference

| Option | Values | Default | Description |
| --- | --- | --- | --- |
| `BasedOnStyle` | LLVM/Google/... | LLVM | Start from preset |
| `IndentWidth` | 1-8 | 2 | Spaces per indent |
| `ColumnLimit` | 0-999 | 80 | Max line length (0 = no limit) |
| `UseTab` | Never/ForIndentation/Always | Never | Tab usage |
| `BreakBeforeBraces` | Attach/Allman/Linux/... | Attach | Brace placement |
| `PointerAlignment` | Left/Right/Middle | Right | `char *ptr` vs `char* ptr` |
| `SortIncludes` | Never/CaseSensitive/... | CaseSensitive | Include ordering |
| `AllowShortFunctionsOnASingleLine` | None/InlineOnly/Empty/All | All | One-line functions |
| `AllowShortIfStatementsOnASingleLine` | Never/WithoutElse/OnlyFirstIf/Always | Never | One-line ifs |
| `SpaceBeforeParens` | Never/ControlStatements/... | ControlStatements | Space before `(` |
| `IndentCaseLabels` | true/false | false | Indent `case` labels |

## Editor Integration

### VS Code
```json
{
    "C_Cpp.clang_format_style": "file",
    "editor.formatOnSave": true,
    "[c]": { "editor.defaultFormatter": "xaver.clang-format" }
}
```

### Vim/Neovim
```vim
" autocmd BufWritePre *.c,*.h :ClangFormat
let g:clang_format#auto_format = 1
```

### Emacs
```elisp
(require 'clang-format)
(add-hook 'c-mode-hook (lambda () (clang-format-mode)))
```

## Common Patterns

### Match Existing Project Style

```bash
# 1. Check existing style
grep -rn "^\t" src/*.c | wc -l   # count tabs
grep -rn "^  " src/*.c | wc -l   # count 2-space

# 2. Create .clang-format matching observed style
# 3. Format only new code (not existing)
git diff HEAD~1 --name-only | grep '\.c$' | xargs clang-format -i
```

### Protect Specific Code

```c
// clang-format off
// Performance-critical layout
static const struct dispatch_table table[] = {
    [MSG_TYPE_PING]  = handle_ping,
    [MSG_TYPE_PONG]  = handle_pong,
    [MSG_TYPE_DATA]  = handle_data,
    [MSG_TYPE_ERROR] = handle_error,
};
// clang-format on
```

## Cross-References

- → See `lobor/cc-skills-clang@clang-code-style` skill for style decisions that inform format config
- → See `lobor/cc-skills-clang@clang-naming` skill for naming conventions
