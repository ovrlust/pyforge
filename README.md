# PyForge

Python source-to-native compiler: AST obfuscation, Cython compilation, and runtime anti-tampering.

---

## The problem

Python code is trivially reversible. `.pyc` files decompile cleanly with tools like `uncompyle6`, and existing protectors like PyArmor work at the bytecode level, which the reversing community has had clean bypasses for since 2022. PyForge goes deeper: obfuscate at the source level, compile to native machine code via Cython, and embed runtime protections directly into the binary.

---

## What it does

- **AST obfuscation** — identifier renaming, per-string HMAC-SHA256 + XOR encryption with per-build key rotation, integer/bytes literal rewriting, and control flow flattening (while-loop state machine replacement)
- **Cython compilation** — Python source transpiled to C and compiled to a native binary; no `.py` or `.pyc` files in the output
- **Runtime anti-debug** — ptrace detection (Linux), `sysctl` kern.proc checks (macOS), `IsDebuggerPresent` / `CheckRemoteDebuggerPresent` (Windows), timing-based detection, and environment variable inspection
- **SHA-256 self-integrity check** — two-pass build: binary compiled with a zeroed hash field, hash computed over the result, patched back in place; any modification to the binary causes exit at startup
- **Encrypted packer** — ChaCha20-Poly1305 encrypted stub; on Linux the payload runs from an anonymous `memfd` and never touches disk, on macOS the temp file is unlinked before `execv`
- **Dependency bundling** — third-party packages zipped and embedded as a C byte array, extracted to `sys.path` at runtime via zipimport
- **Standalone mode** — stdlib + native extensions appended as a compressed payload to the binary; no Python installation required on the target machine
- **Licensing** — optional HTTP license check injected at compile time, configurable endpoint + product ID + key env var
- **Cross-platform packaging** — `.exe` (Windows), `.app` / `.dmg` (macOS), AppImage / binary (Linux), auto-detected by platform
- **Compatibility analysis** — AST pre-scan detects `exec()`, `globals()`, dynamic attribute access, and async code; disables passes that would break runtime behavior

---

## Pipeline

```mermaid
flowchart LR
    A[Source .py] --> B[AST analysis\ncompat check]
    B --> C[Obfuscator\nrename · encrypt strings\nflatten control flow]
    C --> D[Cython backend\nPython → C]
    D --> E[Protection injection\nanti-debug · self-hash C]
    E --> F[C compiler\nclang / gcc]
    F --> G[Self-hash patch\ntwo-pass SHA-256]
    G --> H{Mode}
    H --> I[Packager\n.exe / .app / .dmg / bin]
    H --> J[Packer\nChaCha20 encrypted stub]
    H --> K[Standalone\nbundled stdlib payload]
```

---

## Tech stack

- **Python 3.9+** — pipeline orchestration, AST transforms, obfuscation passes
- **Cython** — Python-to-C transpilation
- **clang / gcc** — C compilation and linking
- **zlib** — payload compression (stdlib, no added deps)
- **ChaCha20-Poly1305** — implemented in pure C and pure Python, no OpenSSL dependency
- **codesign** — ad-hoc signing on macOS post-compilation
- **create-dmg / AppImageTool** — platform packaging

---

## CLI

```
pyforge compile app.py                   # auto-detect format (exe/dmg/bin)
pyforge compile app.py -f dmg            # force macOS DMG
pyforge compile app.py --pack            # encrypted packer stub
pyforge compile app.py --standalone      # bundle stdlib, no Python required
pyforge compile app.py --no-obfuscate    # skip obfuscation, compile only
pyforge protect app.py                   # obfuscate to app.protected.py, no compile
pyforge protect-binary program.exe       # wrap any binary in encrypted stub
```

---

## Before / after (obfuscation)

**Input:**
```python
def calculate_discount(price, customer_tier):
    if customer_tier == "premium":
        return price * 0.8
    return price
```

**After obfuscation pass:**
```python
def _ctx_a3f9b812(node_c471, _ref_0da4e291):
    _cfg_k = b'\x9f\x3a...'
    if _ref_0da4e291 == _cfg(b'\x7c\x21...', _cfg_k):
        return node_c471 * ((0x4 << 0x3) + -((0x1 * 0x4) + 0x18) + 0x64) / 0x64
    return node_c471
```

String literals encrypted with per-build keys. Variable names randomized but visually plausible. Integer constants rewritten as computed expressions. The compiled output is native machine code — none of this survives in recoverable form in the binary.

---

## Status

Active development. Source is private. Available for review during technical interviews — contact below.

Full pipeline shipping: obfuscation, Cython compilation, anti-debug on all three platforms, self-hash integrity, ChaCha20 packer with memfd execution on Linux, dependency bundling, standalone mode, licensing, cross-platform packaging, control flow flattening, and a register-based VM for protecting individual functions.

---

## About

Built PyForge to understand how commercial Python protectors actually work under the hood. Most existing tooling is opaque, and the reversing community has documented clean bypasses for all of it. The parts that turned out to be genuinely interesting: getting Cython's generated C to accept injected protection code without breaking the build, the two-pass self-hash (you can't hash a binary that contains its own hash, so you build twice), and the memfd packer on Linux that keeps the payload off the filesystem entirely. Next is improving the function-level VM — the goal being that even with a debugger attached, extracting the logic of a protected function means reverse engineering a custom ISA that changes every build.
