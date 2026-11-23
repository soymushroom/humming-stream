# Repository guidance

- Read `docs/initial-plan.md` before implementing the score stream, VST host, MCP, or API.
- Treat the plan as the current design source of truth. Keep the external score payload compact: use fraction strings for rational values, `kind` plus `notation` for notes, and omit `requestId` from score and tempo command payloads. Use `expectedRevision` for stale-write protection.
- Do not infer build, test, lint, or architecture conventions. This repository currently has no application source or project configuration. Once implementation files are added, derive commands from their manifests and verify architecture from the source.
