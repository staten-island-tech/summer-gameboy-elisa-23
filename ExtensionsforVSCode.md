Here are the recommended extensions for Game Boy assembly development in VSCode:

---

### 🎮 Must-Have — RGBDS Z80 (`donaldhays.rgbds-z80`)

This is the essential one. It provides a full grammar definition for syntax highlighting, instruction snippets that document every Game Boy CPU instruction (including cycle counts!), a problem matcher to highlight invalid lines when you compile, and IntelliSense for symbols you declare — it even picks up your documentation comments.

Install it by searching **"RGBDS"** in the VSCode Extensions panel, or run:
```
ext install donaldhays.rgbds-z80
```

---

### 🔧 WSL Extension (`ms-vscode-remote.remote-wsl`)

Since you're on WSL, this one is critical. It lets VSCode connect directly into your WSL environment so your terminal, file system, and build tasks all run natively in Linux. Search **"WSL"** in Extensions.

---

### 🐛 Debugging — Emulicious Debugger

The [Emulicious](https://emulicious.net/) emulator has a VSCode debug adapter that lets you set breakpoints and step through your code. It's the most capable free option for GB debugging with VSCode integration.

---

### ✨ Quality of Life

- **Makefile Tools** (`ms-vscode.makefile-tools`) — useful if your project uses a Makefile to build (common for RGBDS projects)
- **Hex Editor** (`ms-vscode.hexeditor`) — handy for inspecting your compiled `.gb` ROM files directly
- **EditorConfig** (`editorconfig.editorconfig`) — keeps indentation/formatting consistent, important since assembly is whitespace-sensitive

---

The RGBDS Z80 extension + WSL extension combo is really the core setup. Once you have those, the VSCode terminal inside WSL will let you run `make` or your build commands and see errors highlighted inline in your source files.