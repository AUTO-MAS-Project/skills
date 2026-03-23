---
name: mas-skills
description: Master entrypoint for MAS backend engineering standards. Use when a task needs consistent conventions across schema naming, module boundaries, function design, API contracts, or data modeling, and route the work to one or more MAS sub-skills.
---

# MAS Skills

## Objective
Provide one unified routing entrypoint for backend standards and enforce consistent implementation decisions.


## Sub-Skills
Use these skills as needed:

1. `mas-schema-naming`: canonical naming for shared schema semantics in future domain work.
2. `mas-module-boundary`: module ownership and dependency direction across backend layers.
3. `mas-function-design`: function-level design for responsibility, signatures, side effects, and errors.
4. `mas-api-contract`: endpoint contract standards for HTTP/WS request-response behavior.
5. `mas-data-model`: modeling standards for schema/config/task layers and compatibility evolution.

## Global Constraints
Apply these constraints before selecting or combining sub-skills.

1. Make minimal necessary changes first; avoid broad refactors unless explicitly requested.
2. Align with current code style and existing project conventions in the touched module.
3. Avoid over-engineering, over-abstraction, and defensive programming that does not match existing code patterns.
4. Study similar existing implementations deeply before coding and follow established local patterns.

## Routing Rules
Choose sub-skills by task intent.

1. Task mentions field names, shared terms, schema key consistency:
Use `mas-schema-naming`.
2. Task mentions layer ownership, imports, coupling, or where code should live:
Use `mas-module-boundary`.
3. Task mentions function splitting, signature quality, return/error behavior:
Use `mas-function-design`.
4. Task mentions endpoint payloads, response model, error contract, websocket payloads:
Use `mas-api-contract`.
5. Task mentions model structure, typing/defaults/constraints, migration of model fields:
Use `mas-data-model`.

## Combined Execution Order
When multiple concerns appear, apply this order:

1. `mas-module-boundary`
2. `mas-data-model`
3. `mas-schema-naming`
4. `mas-function-design`
5. `mas-api-contract`

Reason: place code correctly first, stabilize model structure second, then naming, then function behavior, then transport contract.

## Output Requirements
When using this hub:

1. State which sub-skills are selected.
2. Explain why each selected sub-skill is needed.
3. Apply only the minimum set required by the task.
4. Keep compatibility-first decisions for legacy modules unless explicitly asked to refactor broadly.

## Review Checklist
1. Selected sub-skills match the user request scope.
2. No layer-boundary violations are introduced.
3. Shared schema semantics remain canonical.
4. Function behavior and API contract stay consistent after changes.
