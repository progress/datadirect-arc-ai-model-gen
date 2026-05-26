# AGENTS — arc-genai-agents-v2 Project Agents Index

> **Version:** 1.0

This file documents the custom AI agents available in the arc-genai-agents project.
These agents can be used with any AI coding assistant that supports agent/prompt files
(e.g. VS Code, GitHub Copilot, Cursor, Windsurf, or any OpenAI-compatible tool).

## Registered Agents

- [ARCGenAI-Generator](./agents/ARCGenAI-Generator.agent.md): REST Configuration Generator agent for AutoREST .rest configuration files from Swagger documents.
- [ARCGenAI-EntityGen](./agents/ARCGenAI-EntityGen.agent.md): Internal single-entity sub-agent used by ARCGenAI-Generator to generate temporary per-entity `.entity.tmp` files.
- [ARCGenAI-StaticValidator](./agents/ARCGenAI-StaticValidator.agent.md): Deterministic static validator for generated or hand-edited `.rest` files.
- [ARCGenAI-Launcher](./agents/ARCGenAI-Launcher.agent.md): Launches a `.rest` file in ARC Composer (AutoREST JAR design mode). Does not consume validator output and writes no status artifacts.

---

## Usage

Point your AI coding assistant at the relevant `.agent.md` file and provide a Swagger/OpenAPI
document as input. The agent will generate a `.rest` configuration file compatible with
AutoREST/DataDirect.

To add more agents, place their `.agent.md` files in `.github/agents/` and add them to this list.
