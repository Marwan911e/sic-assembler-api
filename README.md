# SIC Assembler API

A focused, educational API that assembles **SIC (Simplified Instructional Computer)** assembly code into machine-ready object code.

This project exposes a single HTTP endpoint that performs core assembler responsibilities:
- Parses labeled SIC source code
- Builds a symbol table and location counter
- Generates object code for multiple instruction formats
- Produces H/T/E records suitable for loader workflows

---

## Why this project matters

Compiler and systems topics can be difficult to demonstrate in interviews without a concrete artifact. This API provides exactly that: a practical implementation of classic assembler concepts in a modern web service format.

It’s ideal for:
- Systems programming coursework
- Compiler/assembler fundamentals demos
- API integration exercises (frontend + backend)
- Interview portfolios that need tangible low-level logic

---

## Features

- ✅ SIC instruction opcode table initialization (59 instructions)
- ✅ Support for instruction formats:
  - Format 1
  - Format 2 (register-based)
  - Format 3 (PC/base relative, immediate, indirect, indexed)
  - Format 4 (extended with `+` prefix)
- ✅ Pseudo-instruction handling:
  - `START`, `BASE`, `WORD`, `BYTE`, `RESW`, `RESB`, `END`
- ✅ Symbol table generation from labels
- ✅ Location counter computation in hexadecimal
- ✅ Object code output per source line
- ✅ H/T/E record generation
- ✅ Two input modes:
  - JSON text input (`code` field)
  - File upload (`multipart/form-data`)

---

## Tech Stack

- **Node.js** runtime
- **Express.js** API framework
- **multer** for source file uploads
- **body-parser** + **cors** middleware

---

## Project Structure

```text
sic-assembler-api/
├── server.js          # API server + assembler logic
├── package.json       # scripts and dependencies
├── package-lock.json
└── uploads/           # temporary uploaded files
```

---

## Quick Start

### 1) Install dependencies

```bash
npm install
```

### 2) Start the API server

```bash
npm start
```

Server runs at:

```text
http://localhost:5000
```

> Note: The port is currently hardcoded to `5000` in `server.js`.

---

## API Reference

### `POST /assemble`

Assembles SIC source code and returns parsed metadata + object code + records.

#### Option A: JSON body

**Request**

```http
POST /assemble
Content-Type: application/json
```

```json
{
  "code": "COPY START 1000\nFIRST STL RETADR\nCLOOP JSUB RDREC\n LDA LENGTH\n RSUB\nRETADR RESW 1\nLENGTH WORD 3\nEND FIRST"
}
```

#### Option B: File upload

**Request**

```http
POST /assemble
Content-Type: multipart/form-data
```

Form field:
- `file`: text file containing SIC assembly source

---

## Response Format

```json
{
  "label": ["COPY", "FIRST", "CLOOP", "-", "-", "RETADR", "LENGTH", "-"],
  "instruction": ["START", "STL", "JSUB", "LDA", "RSUB", "RESW", "WORD", "END"],
  "reference": ["1000", "RETADR", "RDREC", "LENGTH", "-", "1", "3", "FIRST"],
  "locationCounter": ["1000", "1000", "1003", "1006", "1009", "100C", "100F", "1012"],
  "symbolTable": {
    "COPY": "1000",
    "FIRST": "1000",
    "CLOOP": "1003",
    "RETADR": "100C",
    "LENGTH": "100F"
  },
  "programLength": "12",
  "objectCode": ["No Object code", "172009", "4B2FFA", "032006", "4F0000", "No Object code", "3", "No Object code"],
  "records": [
    "H^COPY  ^001000^001012",
    "T^001000^0C^1720094B2FFA0320064F00003",
    "E^001000"
  ]
}
```

---

## cURL Examples

### Assemble inline code

```bash
curl -X POST http://localhost:5000/assemble \
  -H "Content-Type: application/json" \
  -d '{
    "code": "PROG START 1000\nLDA #3\nSTA ALPHA\nRSUB\nALPHA RESW 1\nEND PROG"
  }'
```

### Assemble from file

```bash
curl -X POST http://localhost:5000/assemble \
  -F "file=@program.asm"
```

---

## Supported Registers

| Register | Code |
|----------|------|
| A        | 0    |
| X        | 1    |
| L        | 2    |
| B        | 3    |
| S        | 4    |
| T        | 5    |
| F        | 6    |
| PC       | 8    |
| SW       | 9    |

---

## Validation / Tests

Current npm scripts:

```bash
npm start
npm test
```

`npm test` is currently a placeholder command in this repository and does not run a real test suite yet.

---

## Limitations & Next Improvements

- Input validation and richer API error responses can be expanded.
- Uploaded files are stored in `uploads/` and are not auto-cleaned.
- Port is fixed at `5000` (could be improved via environment configuration).
- Project can benefit from dedicated unit/integration tests.
- Logic is centralized in `server.js`; modularization would improve maintainability.

---

## Contributing

Contributions are welcome.

Suggested contribution areas:
- Better diagnostics for malformed assembly input
- More robust support for edge cases in addressing modes
- Test coverage for assembler pass logic
- Refactoring into parser / pass1 / pass2 / records modules

---

## License

This project is currently distributed under the repository’s existing license metadata (`ISC` in `package.json`).
