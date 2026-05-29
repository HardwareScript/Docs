# Native Diagnostic System (v0.1.6)

## Overview
As of v0.1.6, the Hardware-Script (`hwc`) compiler has transitioned from using `miette`'s default graphical rendering to a bespoke, **Native Diagnostic System**. This system provides high-fidelity, professional-grade error reporting while maintaining full control over terminal aesthetics and IDE integration.

## Rationale: Why move away from Miette rendering?

While `miette` is an excellent diagnostic library for metadata collection, its default rendering engine introduced several constraints that were incompatible with the Hardware-Script design philosophy:

1. **Redundant Box Headers**: `miette` enforces a box-drawing style (e.g., `╭─[48:1]`) that consumes horizontal space and adds visual clutter. In a professional compiler, we prefer the "Rust-style" minimalist header.
2. **IDE Jump-Link Reliability**: We required absolute control over the `--> file:line:col` format. By handling this manually, we ensure that every IDE (VS Code, JetBrains, etc.) can reliably parse the file path for Ctrl-click navigation.
3. **Cross-Platform Offset Drift**: Windows (CRLF) vs Linux (LF) line endings often caused "off-by-one" errors in `miette`'s default location calculator. Our native `SourceMap` implementation explicitly handles both formats at the byte level.
4. **Color Palette Control**: `miette`'s themes were difficult to customize without affecting the entire structure. Our native system uses `owo-colors` with a curated palette (Dimmed Gutters, Bold White Paths, Bold Red Errors) that feels premium and "Hardware-centric."
5. **Architectural Purity**: By moving rendering logic to a dedicated `hwc-diagnostics` crate, we've decoupled the *reporting* of errors from their *presentation*, allowing us to easily swap output formats (e.g., JSON, Plain Text, Rich ANSI) in the future.

## Implementation Details

### 1. The `SourceMap` & `LocationCalculator`
A dedicated byte-offset to line/column converter that:
- Pre-scans line endings (`\n` and `\r\n`).
- Provides 1-indexed coordinates for human readability.
- Supports efficient lookups via a pre-calculated line index.

### 2. The `DiagnosticPrinter`
A state-of-the-art Unicode printer implemented in `printer.rs`:
- **Header**: Compact `error[CODE]: Message` format.
- **Location**: Clickable `--> path:line:col` headers.
- **Snippets**: Multi-line context (1 line above/below) with dimmed gutters.
- **Pointers**: Sophisticated Unicode underlining using characters like `─┬` and `╰──` for clarity.

### 3. The `DiagnosticCollector` Integration
The `DiagnosticCollector` was refactored to:
- Accept a `file_name` at construction time for context-aware reporting.
- Use the `DiagnosticPrinter` for all `print_all()` and `print_all_with_dedup()` calls.
- Support "Silent Errors" (empty messages) to allow the collector to handle output while the CLI command handles the exit state.

## Results
The new system provides a "Foundry-grade" terminal experience:
- **Clean**: No unnecessary box frames.
- **Informative**: Context lines show surrounding code for better troubleshooting.
- **Compatible**: Clickable links work perfectly in all modern terminal emulators.
- **Monochrome-Balanced**: Uses color as an accent rather than a background, reducing eye strain.

## Future Work
- **Deduplication**: Improved logic for merging multiple errors pointing to the same location.
- **Syntax Highlighting**: Integration with the lexer to provide light syntax coloring within the error snippets.
- **Rich Suggestion API**: Enhanced `help` messages with suggested code diffs.
