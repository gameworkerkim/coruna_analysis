# Coruna Exploit Kit - Deobfuscated (CVE-2024-23222)
# HEAVILY BASED ON AI FOR DEOBFUSCATE

## Overview

This directory contains the deobfuscated version of the **Coruna iOS exploit kit**, which targets **CVE-2024-23222** — a type confusion vulnerability in WebKit's JavaScript engine (JavaScriptCore). The exploit chain achieves remote code execution on iOS devices running iOS 13.0 through iOS 17.x.

The original obfuscated code was found in the `WebKitChainReverse/` directory. Multiple obfuscation layers were applied:
- **XOR-encoded strings**: `[array].map(x => String.fromCharCode(x ^ key)).join("")`
- **XOR-masked constants**: `(number1 ^ number2)` hiding numeric values
- **Minification**: Single-line files up to 292KB
- **Short/random identifiers**: Single-letter variables, random-looking names
- **Base64-wrapped payloads**: Stage2 uses `fgPoij()` (base64 decode + eval)
- **SHA1-keyed module system**: Modules identified by SHA1 hashes

## Exploit Chain Execution Order

```
group.html (entry point)
  │
  ├── 1. Platform Detection (platform_module.js)
  │     └── Detect iOS version, check lockdown mode, check simulator
  │
  ├── 2. Stage 1: WASM Memory Primitives (stage1_wasm_primitives.js)
  │     └── CVE-2024-23222 type confusion → arbitrary read/write
  │
  ├── 3. Runtime Detection
  │     └── Identify JSC memory layout (PSNMWj vs RoAZdq offsets)
  │
  ├── 4. Stage 2: PAC Bypass (stage2_pac_bypass.js)
  │     └── Intl.Segmenter iterator corruption → PAC signing capability
  │
  ├── 5. Stage 3: Sandbox Escape (stage3_sandbox_escape.js)
  │     └── Mach-O parsing, symbol resolution, ARM64 gadgets
  │
  ├── 6. Stage 4: Payload Stub (stage4_payload_stub.js)
  │     └── Delivers encrypted payload via qbrdr() handler
  │
  ├── 7. Stage 5: Main Payload (stage5_main_payload.js)
  │     └── PLASMAGRID stager (~292KB encrypted)
  │
  └── 8. Stage 6: Binary Blob (stage6_binary_blob.bin)
        └── PGP-encrypted binary data
```

## File Descriptions

### Core Framework (from group.html)

| File | Description |
|------|-------------|
| `group_loader.html` | Clean HTML wrapper referencing split-out scripts |
| `utility_module.js` | Type conversions, Int64 class, TypeHelper, pointer tag helpers |
| `platform_module.js` | iOS version detection, version-specific offsets, lockdown mode check |
| `sha256.js` | SHA-256 implementation for module filename hashing |
| `module_loader.js` | Module system: `hPL3On` (sync), `ZKvD0e` (async), `fgPoij` (base64) |
| `exploit_trigger.js` | Main orchestration function (`fqMaGkNR` → `triggerExploit`) |
| `fingerprint.js` | IP detection via icanhazip/ipify, telemetry to 8df7.cc C2 |

### Exploit Stages

| File | Size | Description |
|------|------|-------------|
| `stage1_wasm_primitives.js` | ~40KB | **[DEOBFUSCATED]** WebAssembly-based memory read/write primitives. Two classes (`WasmPrimitive64` for iOS >= 16.4, `WasmPrimitive16` for older) exploit WASM memory to build addrof/fakeobj via JIT type confusion (CVE-2024-23222). Includes heap spray, JIT compilation forcing, and Mach-O header scanning. |
| `stage2_pac_bypass.js` | ~70KB | **[DEOBFUSCATED]** PAC (Pointer Authentication Code) bypass via `Intl.Segmenter` iterator corruption. Contains Mach-O load command parser, dyld shared cache image list resolver, export trie parser, ARM64 gadget finder, and Intl.Segmenter vtable corruption for PAC signing capability. |
| `stage3_sandbox_escape.js` | ~147KB | **[DEOBFUSCATED]** Three nested modules: (1) Mach-O parser + ImageList (shared with Stage 2), (2) JIT cage bypass + PAC-aware function caller, (3) Mach-O payload builder (`MachOPayloadBuilder`) and sandbox escape entry (`executeSandboxEscape`). Builds a Mach-O binary in memory and uses PAC-signed function pointers to escape the WebKit sandbox. |
| `stage4_payload_stub.js` | ~1.7KB | Simple stub calling `window["qbrdr"]()` with ~1.2KB encrypted payload. |
| `stage5_main_payload.js` | ~292KB | Main post-exploitation payload (PLASMAGRID stager), AES-encrypted. |
| `stage6_binary_blob.bin` | ~227KB | PGP-format encrypted binary data (not JavaScript). |
| `stage6_README.md` | — | Documentation for the binary blob. |

## Key Technical Details

### CVE-2024-23222 (Stage 1)
A type confusion vulnerability in JavaScriptCore where:
1. A JIT-compiled function performs an incorrect type check
2. An array's type can be confused between object array and double array
3. This allows reading/writing object pointers as doubles and vice versa
4. Combined with WASM instance corruption, achieves full arbitrary read/write

### PAC Bypass (Stage 2)
ARM64e Pointer Authentication is bypassed by:
1. Creating an `Intl.Segmenter` and getting its iterator
2. Corrupting the iterator's internal vtable pointers
3. Redirecting virtual method calls through ARM64 gadgets
4. Achieving the ability to PAC-sign arbitrary pointers

### Version Support
The exploit supports a wide range of iOS versions through version-specific offset tables:
- **LTgSl5**: iOS 16.x offsets (versions 160000-171100)
- **PSNMWj**: iOS 17.x offsets (versions 100000-170000+)
- **RoAZdq**: iOS 17.x refined offsets

### C2 Infrastructure
- **Telemetry**: `https://8df7.cc/api/ip-sync/sync`
- **Analytics**: Google Analytics `G-LKHD0572ES`
- **Campaign ID**: `CHMKNI9DW334E60711`
- **Module salt**: `cecd08aa6ff548c2`

## Deobfuscation Notes

### XOR String Pattern
```javascript
// Obfuscated:
[35, 56, 33, 33].map(x => String.fromCharCode(x ^ 77)).join("")
// Deobfuscated: "null"
```

### XOR Constant Pattern
```javascript
// Obfuscated:
(896953977 ^ 896953862)
// Deobfuscated: 127 (MAX_SAFE_HI32)
```

### Module Hash Pattern
Modules are identified by 40-character SHA1 hashes. The async loader computes filenames as:
```
SHA256("cecd08aa6ff548c2" + moduleId).substring(0, 40) + ".js"
```

### Identifier Mapping (Key Examples)
| Obfuscated | Deobfuscated | Description |
|-----------|-------------|-------------|
| `m` | `Int64` | 64-bit integer class |
| `nn` | `TypeHelper` | DataView-based type conversion |
| `hPL3On` | `loadModuleSync` | Synchronous module loader |
| `ZKvD0e` | `loadModuleAsync` | Async module loader (XHR) |
| `fgPoij` | `evalBase64Module` | Base64 decode + eval |
| `fqMaGkNR` | `triggerExploit` | Main exploit entry point |
| `J` | `WasmPrimitive64` | 64-byte WASM primitive |
| `$` | `WasmPrimitive16` | 16-byte WASM primitive |
| `it` | `PACBypassBase` | PAC bypass base class |
| `nn` (stage2) | `PACBypass` | PAC bypass implementation |

## Disclaimer

This analysis is for **educational and defensive security research purposes only**. The deobfuscated code documents a real-world exploit chain to help security researchers understand the techniques used and develop better defenses.
