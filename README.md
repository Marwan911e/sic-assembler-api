<div align="center">

```
 ███████╗██╗ ██████╗
 ██╔════╝██║██╔════╝
 ███████╗██║██║
 ╚════██║██║██║
 ███████║██║╚██████╗
 ╚══════╝╚═╝ ╚═════╝
  ASSEMBLER  API
```

# SIC Assembler API

**A production-ready REST API that transforms SIC assembly source code into loader-ready object records.**

[![Node.js](https://img.shields.io/badge/Node.js-Runtime-339933?style=flat-square&logo=nodedotjs)](https://nodejs.org)
[![Express](https://img.shields.io/badge/Express-Framework-000000?style=flat-square&logo=express)](https://expressjs.com)
[![License: ISC](https://img.shields.io/badge/License-ISC-blue?style=flat-square)](https://opensource.org/licenses/ISC)
[![Port: 5000](https://img.shields.io/badge/Port-5000-orange?style=flat-square)]()

> *Compile, decode, and introspect SIC assembly in a single HTTP call.*

</div>

---

## Table of Contents

- [What is SIC?](#what-is-sic)
- [Why This Project?](#why-this-project)
- [Features at a Glance](#features-at-a-glance)
- [Architecture](#architecture)
- [Quick Start](#quick-start)
- [API Reference](#api-reference)
  - [Request Modes](#request-modes)
  - [Response Schema](#response-schema)
  - [Error Handling](#error-handling)
- [Object Code & Records](#object-code--records)
- [Instruction Formats](#instruction-formats)
- [Pseudo-Instructions](#pseudo-instructions)
- [Supported Registers](#supported-registers)
- [cURL Examples](#curl-examples)
- [End-to-End Walkthrough](#end-to-end-walkthrough)
- [Limitations & Roadmap](#limitations--roadmap)
- [Contributing](#contributing)

---

## What is SIC?

The **Simplified Instructional Computer (SIC)** is a hypothetical machine architecture defined in *System Software* by Leland Beck. It exists purely for education — a clean, minimal ISA that strips away the complexity of real-world architectures so students can focus on the concepts that matter:

- How assemblers translate mnemonics to opcodes
- How symbol tables get built across two passes
- How addressing modes affect instruction encoding
- How loaders consume H/T/E record formats

SIC and its extended variant **SIC/XE** appear in virtually every systems programming course. This API makes those concepts tangible.

---

## Why This Project?

Low-level systems concepts are hard to demonstrate without a concrete artifact. This API gives you exactly that.

| Use Case | How This Helps |
|---|---|
| **Coursework** | Verify your manual pass-1 / pass-2 work against a reference implementation |
| **Interview Portfolio** | Show real assembler logic behind a clean REST interface |
| **Frontend Integration** | Build an assembler IDE, syntax checker, or visualizer on top of this API |
| **Teaching** | Project object code line-by-line onto a screen and walk students through it |

---

## Features at a Glance

```
✅  59-instruction opcode table
✅  Two-pass assembly (symbol resolution + object code generation)
✅  All four SIC/XE instruction formats (1, 2, 3, 4)
✅  Complete addressing mode support (PC-relative, base-relative, immediate, indirect, indexed)
✅  Pseudo-instruction handling (START, BASE, WORD, BYTE, RESW, RESB, END)
✅  Symbol table extraction
✅  Hexadecimal location counter per line
✅  H / T / E record output for loaders
✅  Two input modes: inline JSON  OR  multipart file upload
```

---

## Architecture

```
POST /assemble
      │
      ├─ JSON body  ──────────┐
      └─ multipart/form-data ─┤
                              │
                        [ multer ]
                        [ body-parser ]
                              │
                        ┌─────▼──────┐
                        │  PASS ONE  │  Build symbol table
                        │            │  Compute location counters
                        └─────┬──────┘
                              │  Symbol Table
                        ┌─────▼──────┐
                        │  PASS TWO  │  Resolve symbols
                        │            │  Generate object codes
                        └─────┬──────┘
                              │
                        ┌─────▼──────┐
                        │  RECORDS   │  Emit H / T / E records
                        └─────┬──────┘
                              │
                         JSON Response
```

All logic lives in `server.js` and is exposed through a single Express endpoint.

### Project Structure

```
sic-assembler-api/
├── server.js           ← API server + full assembler logic
├── package.json        ← scripts and dependencies
├── package-lock.json
└── uploads/            ← temporary uploaded source files
```

---

## Quick Start

### Prerequisites

- [Node.js](https://nodejs.org) v14 or later
- npm (bundled with Node.js)

### 1. Clone & Install

```bash
git clone <your-repo-url>
cd sic-assembler-api
npm install
```

### 2. Start the Server

```bash
npm start
```

The API will be listening at:

```
http://localhost:5000
```

> **Note:** The port is hardcoded to `5000` in `server.js`. See [Roadmap](#limitations--roadmap) for planned environment-variable support.

### 3. Smoke Test

```bash
curl -s -X POST http://localhost:5000/assemble \
  -H "Content-Type: application/json" \
  -d '{"code": "PROG START 1000\nLDA #3\nSTA ALPHA\nRSUB\nALPHA RESW 1\nEND PROG"}' \
  | python3 -m json.tool
```

If you get a JSON response with `objectCode` and `records` keys — you're up and running.

---

## API Reference

### `POST /assemble`

The single endpoint. Accepts SIC assembly source and returns full assembler output.

| Property | Value |
|---|---|
| **Method** | `POST` |
| **Path** | `/assemble` |
| **Auth** | None |

---

### Request Modes

#### Mode A — Inline JSON

Send the source program as a JSON string with `\n` line separators.

```http
POST /assemble HTTP/1.1
Host: localhost:5000
Content-Type: application/json

{
  "code": "COPY START 1000\nFIRST STL RETADR\nCLOOP JSUB RDREC\n LDA LENGTH\n RSUB\nRETADR RESW 1\nLENGTH WORD 3\nEND FIRST"
}
```

#### Mode B — File Upload

Upload a `.asm` or plain-text source file as multipart form data.

```http
POST /assemble HTTP/1.1
Host: localhost:5000
Content-Type: multipart/form-data

file=@program.asm
```

The `file` field must contain a plain-text file with one SIC instruction per line. Uploaded files are saved temporarily to the `uploads/` directory.

---

### Response Schema

A successful assembly returns HTTP `200` with the following JSON body:

```jsonc
{
  // Source tokens, one entry per line
  "label":          ["COPY",   "FIRST",  "CLOOP",  "-",      ...],
  "instruction":    ["START",  "STL",    "JSUB",   "LDA",    ...],
  "reference":      ["1000",   "RETADR", "RDREC",  "LENGTH", ...],

  // Location counter values in hex, per line (pass one output)
  "locationCounter": ["1000",  "1000",   "1003",   "1006",   ...],

  // Symbol table: label → address (hex)
  "symbolTable": {
    "COPY":   "1000",
    "FIRST":  "1000",
    "CLOOP":  "1003",
    "RETADR": "100C",
    "LENGTH": "100F"
  },

  // Total program size in hex bytes
  "programLength": "12",

  // Object code per line ("No Object code" for pseudo-instructions)
  "objectCode": ["No Object code", "172009", "4B2FFA", "032006", ...],

  // Loader records
  "records": [
    "H^COPY  ^001000^001012",
    "T^001000^0C^1720094B2FFA0320064F00003",
    "E^001000"
  ]
}
```

#### Field Reference

| Field | Type | Description |
|---|---|---|
| `label` | `string[]` | Label token per source line; `"-"` if none |
| `instruction` | `string[]` | Mnemonic or pseudo-op per line |
| `reference` | `string[]` | Operand token per line |
| `locationCounter` | `string[]` | Hex address assigned to each line |
| `symbolTable` | `object` | Map of all resolved labels → hex addresses |
| `programLength` | `string` | Total byte length of program in hex |
| `objectCode` | `string[]` | Hex object code per line, or `"No Object code"` |
| `records` | `string[]` | H, T, and E records ready for a loader |

---

### Error Handling

The API currently returns assembler output for valid programs. Malformed input may result in unexpected output rather than a structured error response. Improved error diagnostics are tracked in the [Roadmap](#limitations--roadmap).

---

## Object Code & Records

The assembler emits three record types in the standard SIC loader format:

### H — Header Record

```
H^<name>^<start address>^<program length>
```

Example:
```
H^COPY  ^001000^001012
```

- `name` — program name, 6 characters, padded with spaces
- `start address` — 6-digit hex
- `program length` — 6-digit hex byte count

### T — Text Record

```
T^<start address>^<length>^<object codes...>
```

Example:
```
T^001000^0C^1720094B2FFA0320064F0000
```

- `start address` — 6-digit hex address of the first byte in this record
- `length` — 2-digit hex byte count of this record (max 30 bytes / 60 hex chars)
- `object codes` — concatenated hex object bytes with no separators

### E — End Record

```
E^<transfer address>
```

Example:
```
E^001000
```

- `transfer address` — hex address of the program entry point (from `END` operand)

---

## Instruction Formats

SIC/XE supports four instruction formats. The assembler automatically selects the correct format based on the mnemonic and any prefix characters.

### Format 1 — 1 byte

```
┌────────┐
│ opcode │  8 bits
└────────┘
```

No operand. Used by a small set of instructions (e.g., `FLOAT`, `FIX`, `NORM`).

---

### Format 2 — 2 bytes

```
┌────────┬────┬────┐
│ opcode │ r1 │ r2 │
└────────┴────┴────┘
  8 bits  4b   4b
```

Register-to-register operations. Operand is one or two register names (e.g., `ADDR A, X`).

---

### Format 3 — 3 bytes *(default)*

```
┌────────┬──┬──┬─┬─┬─┬─────────────┐
│ opcode │ni│xi│b│p│e│  disp/addr  │
└────────┴──┴──┴─┴─┴─┴─────────────┘
  6 bits  1b 1b 1b 1b 1b   12 bits
```

The workhorse format. Supports:

| Flag | Meaning |
|---|---|
| `n=0, i=1` | Immediate addressing (`#val`) |
| `n=1, i=0` | Indirect addressing (`@sym`) |
| `n=1, i=1` | Simple / indexed addressing |
| `x=1` | Indexed (`sym,X`) |
| `b=1` | Base-relative displacement |
| `p=1` | PC-relative displacement |

---

### Format 4 — 4 bytes *(extended, prefix `+`)*

```
┌────────┬──┬──┬─┬─┬─┬──────────────────────┐
│ opcode │ni│xi│b│p│e│       address        │
└────────┴──┴──┴─┴─┴─┴──────────────────────┘
  6 bits  1b 1b 1b 1b 1b      20 bits
```

Used when the target address exceeds 12-bit PC/base-relative range. Triggered by prefixing the mnemonic with `+` (e.g., `+JSUB BIGFUNC`). The `e` flag is set to `1`.

---

## Pseudo-Instructions

Pseudo-instructions (assembler directives) control assembly but do not generate machine code directly.

| Directive | Syntax | Effect |
|---|---|---|
| `START` | `name START addr` | Sets program name and starting address |
| `END` | `END sym` | Marks end of source; defines transfer address |
| `BASE` | `BASE sym` | Tells the assembler to use base-relative addressing with the given symbol |
| `WORD` | `label WORD n` | Allocates one word (3 bytes) initialized to `n` |
| `BYTE` | `label BYTE X'...'` or `C'...'` | Allocates hex or character bytes |
| `RESW` | `label RESW n` | Reserves `n` words (3n bytes), uninitialized |
| `RESB` | `label RESB n` | Reserves `n` bytes, uninitialized |

---

## Supported Registers

| Register | Code | Description |
|---|---|---|
| `A` | `0` | Accumulator |
| `X` | `1` | Index register |
| `L` | `2` | Linkage register (stores return address from `JSUB`) |
| `B` | `3` | Base register (used for base-relative addressing) |
| `S` | `4` | General-purpose |
| `T` | `5` | General-purpose |
| `F` | `6` | Floating-point accumulator |
| `PC` | `8` | Program counter |
| `SW` | `9` | Status word |

---

## cURL Examples

### Assemble a minimal program (inline)

```bash
curl -X POST http://localhost:5000/assemble \
  -H "Content-Type: application/json" \
  -d '{
    "code": "PROG START 1000\nLDA #3\nSTA ALPHA\nRSUB\nALPHA RESW 1\nEND PROG"
  }'
```

### Assemble from a source file

```bash
curl -X POST http://localhost:5000/assemble \
  -F "file=@program.asm"
```

### Pretty-print the response (requires Python)

```bash
curl -s -X POST http://localhost:5000/assemble \
  -H "Content-Type: application/json" \
  -d '{"code": "PROG START 1000\nLDA #3\nSTA ALPHA\nRSUB\nALPHA RESW 1\nEND PROG"}' \
  | python3 -m json.tool
```

---

## End-to-End Walkthrough

Let's trace exactly what happens when you assemble the classic `COPY` program.

### Source Input

```
COPY   START  1000
FIRST  STL    RETADR
CLOOP  JSUB   RDREC
       LDA    LENGTH
       RSUB
RETADR RESW   1
LENGTH WORD   3
       END    FIRST
```

### Pass One — Build Symbol Table

The assembler steps through each line, assigns hex addresses, and records labels:

| Line | Label | Instr | Operand | LC (before) | LC (after) |
|---|---|---|---|---|---|
| 1 | COPY | START | 1000 | — | 1000 |
| 2 | FIRST | STL | RETADR | 1000 | 1003 |
| 3 | CLOOP | JSUB | RDREC | 1003 | 1006 |
| 4 | — | LDA | LENGTH | 1006 | 1009 |
| 5 | — | RSUB | — | 1009 | 100C |
| 6 | RETADR | RESW | 1 | 100C | 100F |
| 7 | LENGTH | WORD | 3 | 100F | 1012 |
| 8 | — | END | FIRST | 1012 | — |

**Symbol Table after Pass One:**

```json
{
  "COPY":   "1000",
  "FIRST":  "1000",
  "CLOOP":  "1003",
  "RETADR": "100C",
  "LENGTH": "100F"
}
```

### Pass Two — Generate Object Code

Each instruction's operand is now resolved to an address and the object bytes are computed.

| Instr | Operand | Resolved Address | Object Code | Notes |
|---|---|---|---|---|
| STL | RETADR | 100C | `172009` | PC-relative: 100C − 1003 = 9 |
| JSUB | RDREC | (external) | `4B2FFA` | |
| LDA | LENGTH | 100F | `032006` | PC-relative: 100F − 100C = 3 → wait, disp relative to next PC |
| RSUB | — | — | `4F0000` | Fixed opcode, no operand |
| RESW | 1 | — | *(none)* | Storage only |
| WORD | 3 | — | `000003` | |

### Final API Response

```json
{
  "label":           ["COPY", "FIRST", "CLOOP", "-",      "-",      "RETADR", "LENGTH", "-"],
  "instruction":     ["START","STL",   "JSUB",  "LDA",    "RSUB",   "RESW",   "WORD",   "END"],
  "reference":       ["1000", "RETADR","RDREC", "LENGTH", "-",      "1",      "3",      "FIRST"],
  "locationCounter": ["1000", "1000",  "1003",  "1006",   "1009",   "100C",   "100F",   "1012"],
  "symbolTable": {
    "COPY":   "1000",
    "FIRST":  "1000",
    "CLOOP":  "1003",
    "RETADR": "100C",
    "LENGTH": "100F"
  },
  "programLength": "12",
  "objectCode": [
    "No Object code",
    "172009",
    "4B2FFA",
    "032006",
    "4F0000",
    "No Object code",
    "3",
    "No Object code"
  ],
  "records": [
    "H^COPY  ^001000^001012",
    "T^001000^0C^1720094B2FFA0320064F00003",
    "E^001000"
  ]
}
```

## Contributing

Contributions are welcome. Fork the repository, make your changes, and open a pull request.

### High-Value Contribution Areas

| Area | Description |
|---|---|
| **Error diagnostics** | Return structured errors with line numbers when assembly fails |
| **Edge case coverage** | Forward references, literals, base-relative vs PC-relative fallback |
| **Test coverage** | Jest or Mocha tests for individual assembler passes |
| **Modularization** | Split `server.js` into focused modules with clean interfaces |
| **Extended addressing** | Improve `BYTE` directive to handle more literal formats |

### Development Setup

```bash
# Install dependencies
npm install

# Start with auto-restart on file changes (requires nodemon)
npx nodemon server.js

# Run tests (placeholder — contribute real tests!)
npm test
```

---

## License

Licensed under the **ISC License**.  
See `package.json` for metadata. Adding a dedicated `LICENSE` file is recommended before public distribution.

---

<div align="center">

*Built on the SIC/XE architecture described in*  
*Leland Beck — System Software: An Introduction to Systems Programming*

</div>
