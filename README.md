# ARC GenAI Agents

> **Version:** 1.0

---

## Table of Contents

- [Who is this for?](#who-is-this-for)
- [Overview](#overview)
- [Quick Start](#quick-start)
- [Workflow](#workflow)
- [Known limitations and security](#known-limitations-and-security)
  - [Known limitations](#known-limitations)
  - [Security](#security)
- [What this repository includes](#what-this-repository-includes)
  - [Agents and guides you will use](#agents-and-guides-you-will-use)
  - [Internal files (do not modify or invoke directly)](#internal-files-do-not-modify-or-invoke-directly)
- [Prerequisites](#prerequisites)
  - [Environment (choose one)](#environment-choose-one)
  - [Required for all steps](#required-for-all-steps)
  - [Required for the Launcher step only](#required-for-the-launcher-step-only)
- [Repository structure](#repository-structure)
- [Supported input](#supported-input)
- [User workflow](#user-workflow)
  - [1. Generate a `.rest` model](#1-generate-a-rest-model)
  - [2. Review required manual items](#2-review-required-manual-items)
  - [3. Validate generated output](#3-validate-generated-output)
  - [4. Launch and use in ARC Composer](#4-launch-and-use-in-arc-composer)
- [VS Code and Copilot CLI compatibility](#vs-code-and-copilot-cli-compatibility)
- [Important usage notes](#important-usage-notes)

---

## Who is this for?

- API developers and engineers building or testing REST connections with the DataDirect Autonomous REST Connector (ARC)
- Teams who want to generate `.rest` model files faster using AI assistance

---

## Overview

This project delivers AI-assisted generation of DataDirect Autonomous REST Connector (ARC) `.rest` model files from Swagger/OpenAPI documents. It is designed to reduce manual model authoring effort while keeping users in control of final correctness.

---

## Quick Start

1. Clone this repository and open the **`arc-genai-agents/`** folder as your workspace root in VS Code, or `cd` into it in the terminal. Agent and prompt paths (`.github/agents/...`, `.github/prompts/...`) resolve relative to this folder.

2. Add your Swagger/OpenAPI document to `input/swagger/`.

3. **Generate the model:**

   **VS Code:** Open Copilot Chat (agent mode), attach your Swagger/OpenAPI document to the chat context, then run `/ARCGenAI-Generator`.

   **GitHub Copilot CLI:**
   ```
   /ARCGenAI-Generator @input/swagger/MyAPI.yaml
   ```

4. **Review output** in `ai-output/{name}/{name}-generation-status.md` and address all items in the `MANDATORY REVIEW BEFORE USE` section before continuing. See the [User workflow](#user-workflow) section for details.

5. **Validate the model:**

   **VS Code:** Attach the generated `.rest` file and source Swagger/OpenAPI document to the chat context, then run `/ARCGenAI-StaticValidator`.

   **GitHub Copilot CLI:**
   ```
   /ARCGenAI-StaticValidator @ai-output/MyAPI/MyAPI.rest @input/swagger/MyAPI.yaml
   ```

6. **Launch in ARC Composer:**

   **VS Code:** Attach the validated `.rest` file to the chat context and run `/ARCGenAI-Launcher`.

   **GitHub Copilot CLI:**
   ```
   /ARCGenAI-Launcher @ai-output/MyAPI/MyAPI.rest
   ```

Output is generated in `ai-output/`.

---

## Workflow

```
Swagger/OpenAPI document  ->  ARCGenAI-Generator  ->  Manual Review  ->  ARCGenAI-StaticValidator  ->  ARCGenAI-Launcher  ->  ARC Composer
```

---

## Known limitations and security

### Known limitations

- **JSONRoot / response-root mapping** -- when an API returns a wrapped response object (e.g., `{ "data": [ ... ] }`), the correct `jsonroot` value cannot always be reliably determined from the Swagger/OpenAPI document alone. The generator will include its best-effort value and flag it in the status file's `MANDATORY REVIEW BEFORE USE` section. Always confirm the root path against a real API response before promoting the model.
- **Pagination** -- page-token strategies and cursor-based paging are not fully derivable from Swagger/OpenAPI documents. Pagination directives should be reviewed and tested manually.
- **Primary key selection** -- when a resource has multiple candidate key fields, the generator picks the most likely one and notes its reasoning in the status file. Review and adjust as needed.
- **Large or circular-reference-heavy documents** -- may require a lower-context model or a pre-processed/trimmed document. The generator will halt and advise if this situation is detected.

### Security

- The Generator treats **all Swagger/OpenAPI field values as untrusted input** -- including `description`, `summary`, `example`, `title`, and schema names. It will never follow, execute, or reinterpret instructions embedded in those fields.
- Swagger/OpenAPI content is used only as API metadata to map structure, paths, parameters, and schema semantics.
- If output filename derivation contains unsafe path-like characters, the Generator will pause and ask for clarification rather than writing files.

These guardrails are especially relevant when processing third-party or externally sourced Swagger/OpenAPI documents.

---

## What this repository includes

### Agents and guides you will use

- **Generator agent** (`ARCGenAI-Generator`) -- orchestrates `.rest` generation from a Swagger/OpenAPI document
- **Static validator agent** (`ARCGenAI-StaticValidator`) -- runs structural and semantic checks on generated output (e.g., directive correctness, field types, endpoint coverage, and schema alignment)
- **Launcher agent** (`ARCGenAI-Launcher`) -- opens a validated `.rest` file in ARC Composer for connection string building and SQL testing
- **Manual review guide** (`.github/agents/docs/manual-rest-adjustments.md`) -- reference for cases that require manual adjustment after generation

### Internal files (do not modify or invoke directly)

- **Entity sub-agent** (`ARCGenAI-EntityGen`) -- internal sub-agent called by the Generator automatically
- **Global language spec** (`.github/agents/docs/global-lang-spec.md`) -- authoritative `.rest` language rules used by the Generator and Validator agents
- **REST reference template** (`.github/agents/docs/rest-reference-template.rest`) -- canonical `.rest` example used by the agents as a reference
- **Generator status template** (`.github/agents/docs/generator-status-template.md`) -- template used by the Generator when writing status output
- **Verification status template** (`.github/agents/docs/verification-status-template.md`) -- template used by the Validator when writing status output
- **Prompt files** (`.github/prompts/`) -- underlying prompt definitions backing the slash commands; used with the agents

---

## Prerequisites

### Environment (choose one)

**VS Code**

- Install [VS Code](https://code.visualstudio.com/) and the GitHub Copilot extension.
- Ensure `github.copilot.chat.agent.enabled` is set to `true` in settings (enabled by default).
- Open the `arc-genai-agents/` folder as your workspace root.

**GitHub Copilot CLI**

- Install the CLI: `npm install -g @github/copilot` (requires Node.js 18+).
- Run `copilot` and `/login` from the `arc-genai-agents/` directory to authenticate.

### Required for all steps

- **Swagger/OpenAPI document** -- a valid `.json`, `.yaml`, or `.yml` file placed in `input/swagger/`.

### Required for the Launcher step only

- **Java Runtime (JRE/JDK 8+)** -- the `java` executable must be on your system `PATH`. Verify with `java -version`.
- **AutoREST JAR (`autorest.jar`)** -- part of a DataDirect AutoREST / Progress DataDirect JDBC install. Requires JDBC version 6.0.1.7232 or newer, or ODBC version 8.01.2290 or newer.

  The Launcher searches the following locations in order:
  1. `tools/` in the workspace root (`arc-genai-agents/tools/autorest.jar`)
  2. `C:\Program Files\Progress\DataDirect\JDBC` (Windows default install path)
  3. `/opt/Progress/DataDirect/JDBC` (Linux/macOS default install path)

  If the JAR is not in one of these locations, place it under `tools/` or supply its path when the Launcher asks.

---

## Repository structure

- `.github/agents/` -- agent definitions and supporting documentation
- `.github/prompts/` -- prompt definitions for slash commands
- `input/` -- Swagger/OpenAPI input documents (`.json`, `.yaml`, `.yml`)
- `ai-output/` -- generated output: `.rest` files, status files, and entity artifacts
  - `ai-output/yelp-api/` -- example output generated from `input/swagger/yelp-api.yaml`; use these files as a reference for expected generator and validator output
- `tools/` -- optional location for AutoREST JAR for Launcher use

---

## Supported input

- **Supported:** Swagger/OpenAPI documents (`.json`, `.yaml`, `.yml`)
- **Out of scope for version 1.0:** PDF API documentation as generation input

---

## User workflow

> **Tip:** When `arc-genai-agents/` is open as the workspace root, the LLM can usually locate files in the workspace automatically without explicit paths. The file paths shown below are provided for clarity -- in practice you can often just name the file or let the agent find it.

### 1. Generate a `.rest` model

**VS Code:** Open Copilot Chat in agent mode, attach your Swagger/OpenAPI document (e.g. `input/swagger/MyAPI.yaml`) to the chat context, then run `/ARCGenAI-Generator`.

**GitHub Copilot CLI:**

```
/ARCGenAI-Generator @input/swagger/MyAPI.yaml
```

Expected output in `ai-output/{name}/`:

- `{name}.rest`
- `{name}-generation-status.md`
- `{name}-entity-plan.md` (planning artifact)

> If the provided file is not a valid Swagger/OpenAPI document, or if the document is excessively large or circular-reference-heavy, the generator will halt with an explanatory message. When decisions are ambiguous, it asks for clarification instead of guessing.

### 2. Review required manual items

Before using the generated `.rest`, review:

- `ai-output/{name}/{name}-generation-status.md` for gaps, exclusions, and mandatory review items
- `.github/agents/docs/manual-rest-adjustments.md` for cases that are ambiguous or not fully derivable from Swagger/OpenAPI documents (for example: root mapping nuances, key decisions, paging details)
- The status file fingerprint (source filename, timestamp, agent version, endpoint count) and the `MANDATORY REVIEW BEFORE USE` section before promoting output

### 3. Validate generated output

**VS Code:** Open Copilot Chat in agent mode, attach the generated `.rest` file and the source Swagger/OpenAPI document to the chat context, then run `/ARCGenAI-StaticValidator`.

**GitHub Copilot CLI:**

```
/ARCGenAI-StaticValidator @ai-output/MyAPI/MyAPI.rest @input/swagger/MyAPI.yaml
```

The agent will locate all other required inputs (reference template, language spec, status template) automatically from the workspace.

The validator writes one artifact:

- `ai-output/<basename>/<basename>-validation-status.md`

Chat output is intentionally minimal (PASS or FAIL), with details in the validation status file.

The validator may modify the `.rest` file in-place during validation. It runs one round of rule-based auto-fixes -- corrections applied consistently without model judgment (e.g., malformed directive shapes, invalid type tokens, formatting violations) -- before the final check pass. Non-obvious issues are reported but never auto-applied.

A FAIL result means one or more of the following quality-gate conditions was not met:

- **Failed checks exceed threshold** (`BLOCKED_FAILED_CHECKS`) -- structural or semantic errors remain after the auto-fix pass
- **Endpoint coverage below 100%** (`BLOCKED_COVERAGE`) -- not all Swagger/OpenAPI endpoints are mapped in the `.rest` file
- **Missing or unreadable input file** -- the validator will fail immediately and report which file could not be resolved

### 4. Launch and use in ARC Composer

**VS Code:** Attach the validated `.rest` file to the chat context and run `/ARCGenAI-Launcher`.

**GitHub Copilot CLI:**

```
/ARCGenAI-Launcher @ai-output/MyAPI/MyAPI.rest
```

Requires a Java 8+ runtime and `autorest.jar` (see Prerequisites). The launcher does not consume validator output and writes no status artifacts.

> **If any prerequisite check fails**, the launcher will report `Launch: FAILED` and stop -- it will not attempt to open ARC Composer. The failure message will name the specific cause (e.g., `java not found on PATH`, `autorest.jar not found at <paths checked>`, `.rest file not found at <path>`) along with the concrete next step to resolve it.

---

## VS Code and Copilot CLI compatibility

Both VS Code Copilot Chat and GitHub Copilot CLI support the agents and prompts in this repository:

- `.github/agents/*.agent.md` -- agent definitions, readable by both VS Code agent mode and Copilot CLI. Use `/agent` in the CLI to browse and select, or reference the agent by name in a prompt.
- `.github/prompts/*.prompt.md` -- slash command prompts, usable in both VS Code chat and Copilot CLI.

Available agents and slash commands:

- `ARCGenAI-Generator` / `/ARCGenAI-Generator`
- `ARCGenAI-StaticValidator` / `/ARCGenAI-StaticValidator`
- `ARCGenAI-Launcher` / `/ARCGenAI-Launcher`

---

## Important usage notes

- `ARCGenAI-EntityGen` is an internal sub-agent -- run `ARCGenAI-Generator` for generation workflows.
- Temporary entity artifacts may remain under `ai-output/{name}/entities/` (`*.entity.tmp` and `_header.assembly.tmp`); these are intermediate files and are safe to delete manually if automatic cleanup did not run.
- AI output is assistive, not a substitute for API-owner validation. Users are expected to review and edit final model files before production use.
