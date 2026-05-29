# Hardware Script v0.1.2 - IDE Integration and Launch Strategy

**Document Type**: Developer Experience and Launch Guide  
**Status**: Implementation Guide  
**Last Updated**: March 2026

---

## Purpose of This Document

This document explains how IDE integration works for new programming languages and provides a practical launch strategy for Hardware Script.

**Key insight**: Language integration requires two separate pieces - the compiler (executable) and the IDE experience (extension).

---

## Understanding Language Integration

### The Two-Box Model

**When you create a programming language, you distribute it in two separate pieces:**

**Box 1: The Core System (The Executable)**
```
The hws compiler
The hpm package manager
The actual compilation logic
```

**Users install via terminal**:
```bash
cargo install hwc-cli
# or
brew install hwc
```

**This does the actual work**: Turning `.hw` files into Gerber and 3D files.

**It knows nothing about**: Screen colors, icons, or IDE features.

**Box 2: The IDE Experience (The Extension)**
```
File icons
Syntax highlighting
Auto-complete
Error diagnostics
```

**Users install from IDE marketplace**:
```
VS Code Marketplace
IntelliJ Plugin Repository
```

**This provides the visual experience**: Colors, icons, IntelliSense.

**It knows nothing about**: Actual compilation or hardware logic.

### The Rust Example

**To write Rust code**:

1. **Install compiler** (terminal):
   ```bash
   rustup install stable
   ```

2. **Install IDE extension** (VS Code):
   ```
   Install "rust-analyzer" from marketplace
   ```

**Two separate installations, two separate purposes.**

---

## File Icons: How They Work

### Who Handles Them?

**Short answer**: You design them, IDEs display them via extensions.

### The Process

**1. You design the logo** (usually SVG):
```
.hw  → Blue microchip icon
.hwx → Green component icon
.hwx → Purple executable icon
.hwmat → Orange material icon
```

**2. You package them into a VS Code extension**:
```json
// In package.json
{
  "contributes": {
    "iconThemes": [
      {
        "id": "hardware-script-icons",
        "label": "Hardware Script Icons",
        "path": "./icons/hardware-script-icon-theme.json"
      }
    ]
  }
}
```

**3. Users install your extension**:
```
VS Code reads the configuration
Displays your SVG next to matching files
```

### Design Recommendations

**Keep icons simple and distinct**:
- Use consistent color scheme
- Make them recognizable at small sizes
- Follow VS Code icon guidelines
- Use SVG format for scalability

---

## Syntax Highlighting: How It Works

### Who Handles It?

**Short answer**: You write the rules, the IDE applies the colors.

### The Process

**Syntax highlighting doesn't come from the compiler executable.**

**It comes from a TextMate Grammar file** (JSON with regex patterns).

### Example Grammar Rules

```json
{
  "scopeName": "source.hw",
  "patterns": [
    {
      "name": "keyword.control.hw",
      "match": "\\b(define|Space|add|route|import)\\b"
    },
    {
      "name": "constant.numeric.hw",
      "match": "\\[\\d+,\\s*\\d+,\\s*\\d+\\]"
    },
    {
      "name": "comment.line.hw",
      "match": "#.*$"
    },
    {
      "name": "string.quoted.double.hw",
      "match": "\"[^\"]*\""
    },
    {
      "name": "entity.name.type.hw",
      "match": "\\b[A-Z][a-zA-Z0-9_]*\\b"
    }
  ]
}
```

### How It Works

**1. You write regex patterns**:
```
Keywords: \b(define|Space|add)\b → keyword.control.hw
Coordinates: \[\d+,\s*\d+,\s*\d+\] → constant.numeric.hw
Comments: #.*$ → comment.line.hw
```

**2. IDE tags matching text**:
```
"define" → keyword.control.hw
"[1, 10, 10]" → constant.numeric.hw
"# comment" → comment.line.hw
```

**3. Theme applies colors**:
```
keyword.control.hw → Purple (in Dark+ theme)
constant.numeric.hw → Light blue
comment.line.hw → Green
```

**Different themes use different colors for the same tags.**

---

## Creating a VS Code Extension

### Quick Start

**1. Install prerequisites**:
```bash
npm install -g yo generator-code
```

**2. Generate extension**:
```bash
yo code
```

**3. Choose "New Language Support"**:
```
? What type of extension? New Language Support
? Language id: hw
? Language name: Hardware Script
? File extensions: .hw
? Scope names: source.hw
```

**4. Edit the generated files**:
```
hardware-script/
├── package.json          # Extension metadata
├── syntaxes/
│   └── hw.tmLanguage.json  # Grammar rules
└── language-configuration.json  # Brackets, comments
```

**5. Test locally**:
```bash
# Press F5 in VS Code
# Opens new window with extension loaded
```

**6. Publish to marketplace**:
```bash
vsce package
vsce publish
```

### Minimal Extension Structure

**package.json**:
```json
{
  "name": "hardware-script",
  "displayName": "Hardware Script",
  "description": "Language support for Hardware Script (.hw)",
  "version": "0.1.0",
  "engines": {
    "vscode": "^1.80.0"
  },
  "categories": ["Programming Languages"],
  "contributes": {
    "languages": [{
      "id": "hw",
      "aliases": ["Hardware Script", "hw"],
      "extensions": [".hw", ".hwx", ".hwx"],
      "configuration": "./language-configuration.json"
    }],
    "grammars": [{
      "language": "hw",
      "scopeName": "source.hw",
      "path": "./syntaxes/hw.tmLanguage.json"
    }]
  }
}
```

---

## The Launch Strategy

### GitHub Organization Setup

**Create the organization structure**:
```
github.com/hardware-script/
├── hwc                    # Main compiler repo
├── hwc-vscode            # VS Code extension
├── hwc-registry          # Package registry
├── hwc-stdlib            # Standard library
└── hwc-examples          # Example projects
```

**Benefits**:
- Professional appearance
- Clear organization
- Room for growth
- Community contributions

### The Ecosystem MVP

**What you need for launch**:

**1. Working compiler** ✅
```
hws build board.hw
```

**2. Basic documentation** ✅
```
README.md
GETTING-STARTED.md
ARCHITECTURE.md
```

**3. Example project** ✅
```
examples/simple-led/
├── board.hw
├── components/
└── README.md
```

**4. VS Code extension** (optional but recommended)
```
Syntax highlighting
File icons
```

**5. Package registry structure** (can be empty)
```
registry/
├── README.md
└── packages.yaml
```

### The Launch Pitch

**Title**:
```
Hardware Script: Text-based hardware design that compiles 
to PCB/FPGA/ASIC in <10ms
```

**Key points**:
- ✅ Fully text-based (no GUI)
- ✅ Git-friendly (meaningful diffs)
- ✅ LLM-native (AI can design hardware)
- ✅ Fast compilation (<10ms)
- ✅ Open-source (MIT)
- ✅ Cross-platform

**The hook**:
```
"I'm building the npm for hardware. Here's a working compiler 
that turns text files into PCB manufacturing files."
```

### Launch Platforms

**Primary**:
- Hacker News (Show HN)
- Reddit (r/programming, r/hardware, r/rust)
- Twitter/X

**Secondary**:
- Dev.to
- Lobsters
- LinkedIn

**Community**:
- Discord server
- GitHub Discussions

---

## The Demo Video

### 60-Second Structure

**0:00-0:10 - The Hook**:
```
"Watch me design a PCB using only a text editor"
```

**0:10-0:20 - Show VS Code**:
```
Open .hw file with syntax highlighting
Show the simple, readable syntax
```

**0:20-0:30 - Compile**:
```bash
hws build led.hw
# Compiles in <10ms
```

**0:30-0:45 - Show Outputs**:
```
Open Gerber file in viewer
Open 3D model
Show it's real, manufacturable
```

**0:45-0:60 - The Vision**:
```
"This is v0.1. The vision is npm for hardware.
Star the repo to follow development."
```

### Recording Tips

- Use clean terminal theme
- Show file tree in VS Code
- Use large, readable font
- No mistakes (edit if needed)
- Add captions for accessibility

---

## What Makes It an Ecosystem MVP

### Even Without Complete Implementation

**The ecosystem architecture is visible**:

**1. Multiple repositories**:
```
Compiler, registry, stdlib, examples
```

**2. Clear documentation**:
```
Vision, architecture, getting started
```

**3. Extension points**:
```
Package manager structure
Component library format
Plugin architecture
```

**4. Community readiness**:
```
Contributing guide
Code of conduct
Issue templates
```

### The Message

**You're not selling a finished product.**

**You're selling**:
- A working proof of concept
- A clear vision
- A solid architecture
- An invitation to build together

**The community will help build the rest.**

---

## Launch Checklist

### Must Have (Tuesday)

- ✅ Working compiler
- ✅ Example project that compiles
- ✅ README with clear explanation
- ✅ GETTING-STARTED guide
- ✅ GitHub organization
- ✅ Demo video or GIF

### Nice to Have (Tuesday)

- ⭐ VS Code extension (even basic)
- ⭐ Syntax highlighting
- ⭐ File icons
- ⭐ Multiple examples

### Can Wait (Week 2)

- ⏳ Package manager implementation
- ⏳ Standard library components
- ⏳ Advanced features
- ⏳ Performance optimization

---

## Key Takeaways

1. **Two-box model** - Compiler and IDE extension are separate

2. **File icons via extensions** - You design, IDE displays

3. **Syntax highlighting via grammar** - Regex patterns + theme colors

4. **VS Code extension is easy** - Use generator, publish to marketplace

5. **Launch with ecosystem structure** - Even if not fully implemented

6. **Vision matters more than completeness** - Show the architecture

7. **Demo video is critical** - 60 seconds, clear, compelling

8. **Community will help** - Leave room for contributions

9. **Professional appearance** - GitHub org, documentation, examples

10. **The pitch is the vision** - "npm for hardware"

---

## Summary

**IDE integration requires two pieces**:
- Compiler (executable) - Does the work
- Extension (IDE plugin) - Provides the experience

**Launch strategy**:
- GitHub organization with clear structure
- Working compiler with examples
- VS Code extension (even basic)
- Demo video showing the workflow
- Clear vision and invitation to contribute

**The message**: "Here's a working proof of concept and a clear vision. Help me build the npm for hardware."

**Don't stress about completeness** - The foundation is solid, the vision is clear, the community will help build the rest.

---

**Document Status**: Developer Experience and Launch Guide  
**Last Updated**: March 2026  
**Part of**: Hardware Script v0.1.2 Documentation Suite
