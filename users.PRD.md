# users.PRD.md

## Title

Unified Multi-User Access Registry (`users.json`) for OpenClaw

## Status

Draft PRD (research-based)

## Context

You want OpenClaw to support multiple human users cleanly across Telegram + WhatsApp (and group contexts), with per-user roles and group memberships, without manually duplicating allowlists in many channel-specific config blocks.

Current OpenClaw already supports:

- Multi-agent routing via `bindings` (channel/account/peer, plus Discord role-aware routing)
- DM policies + allowlists (`dmPolicy`, `allowFrom`, `groupPolicy`, etc.)
- Pairing allow stores in `~/.openclaw/credentials/*-allowFrom.json`
- Command auth (`commands.allowFrom`) and owner resolution

But there is no first-class cross-channel user registry (`users.json`) to define identities once and project permissions consistently.

---

## Reality Check (what exists today)

### What I verified in source/docs

- Routing and specificity: `src/routing/resolve-route.ts`
- Binding type (`accountId`, `peer`, `guildId`, `teamId`, `roles`): `src/config/types.agents.ts`
- Command sender authorization path: `src/auto-reply/command-auth.ts`
- Multi-agent concept + one-number DM split docs: `docs/concepts/multi-agent.md`
- DM/group policy and allowlist docs: `docs/gateway/configuration-reference.md`, `docs/gateway/security/index.md`

### What is missing

- No `users.json` file or `users` top-level config schema in repo
- No cross-channel user identity abstraction (same person across Telegram ID + WhatsApp E.164)
- No global role model for user capabilities that can be consumed by all channels consistently

---

## Problem Statement

Managing many users currently requires scattered configuration:

- Per-channel `allowFrom`
- Channel-specific group allow rules
- Optional command allowlists
- Optional peer-level bindings

This creates drift, duplication, and risk of inconsistent permissions.

---

## Goals

1. Add a single source of truth for users (`users.json`).
2. Support identities across Telegram + WhatsApp (extensible to others).
3. Support per-user roles and group memberships.
4. Project user membership into existing authorization/routing paths without breaking current configs.
5. Keep backward compatibility (existing configs continue working unchanged).

## Non-Goals (v1)

- Replacing all existing channel-level allowlist fields immediately
- Enterprise RBAC UI
- Complex ABAC/policy engine

---

## Proposed Design

## 1) New registry file

Location (recommended):

- `~/.openclaw/users.json` (state-level, shared across agents by default)

Optional override:

- `users.path` in `openclaw.json`

### Schema (v1 draft)

```json
{
  "version": 1,
  "users": [
    {
      "id": "juan",
      "displayName": "Juan Tamez",
      "enabled": true,
      "roles": ["owner", "coordinator"],
      "identities": {
        "telegram": ["6586915095"],
        "whatsapp": ["+15551234567"]
      },
      "groups": ["ops-core", "sages-admin"],
      "meta": {
        "notes": "primary operator"
      }
    }
  ],
  "groups": [
    {
      "id": "ops-core",
      "roles": ["operator"],
      "channels": {
        "whatsapp": {
          "dm": true,
          "groups": ["1203630...@g.us"]
        },
        "telegram": {
          "dm": true,
          "groups": ["-1001234567890"]
        }
      }
    }
  ],
  "roles": {
    "owner": {
      "capabilities": ["commands:*", "tools:elevated", "admin:config"]
    },
    "operator": {
      "capabilities": ["commands:core", "tools:standard"]
    },
    "viewer": {
      "capabilities": ["commands:status"]
    }
  }
}
```

## 2) Normalized identity key

Introduce canonical actor key:

- `provider:userId` (examples: `telegram:6586915095`, `whatsapp:+1555...`)

Resolution pipeline:

1. Parse inbound sender identity from channel adapter (already available in context)
2. Normalize format with existing channel formatter logic
3. Resolve to `users.json` entry
4. Attach `resolvedUser` to message/command context

## 3) Authorization integration points

### A. Command auth (`resolveCommandAuthorization`)

- File: `src/auto-reply/command-auth.ts`
- Add optional pre-check:
  - If `users` registry enabled and sender resolved:
    - compute `isAuthorizedSender` from role capabilities + channel/group scope
  - Else fallback to current behavior (`commands.allowFrom` + channel allowFrom)

### B. DM/group access checks (provider preflight)

- Telegram/WhatsApp/Signal/Discord preflight modules already evaluate policy + allowlists.
- Add optional derived allow decision from `users.json` membership.
- Keep existing `dmPolicy`/`groupPolicy` semantics as guardrails.

### C. Routing/bindings

- Keep `bindings` as route primitive.
- Add optional helper expansion:
  - Bind by role/group/user alias via CLI helper (compiled to concrete bindings)
  - Routing engine remains deterministic and unchanged.

## 4) Capability model (minimum viable)

Capabilities are evaluated in three buckets:

- `commands:*` / `commands:<name>`
- `tools:standard` / `tools:elevated`
- `admin:config` / `admin:routing`

Role assignment precedence:

1. Direct user roles
2. Group-inherited roles
3. Optional defaults

Deny-by-default for unknown users unless explicit open mode is configured.

---

## Backward Compatibility Strategy

1. `users.json` is optional in v1.
2. If absent, behavior is exactly current behavior.
3. If present, rollout mode controlled by config:
   - `users.mode: "observe" | "enforce"`
   - `observe`: log decisions only
   - `enforce`: applies to auth decisions
4. Existing allowlists remain valid and can coexist.

---

## Migration Plan

### Phase 0 — Discovery + Instrumentation

- Add user-resolution service and structured logs:
  - resolved user id
  - roles
  - decision source (`legacy-allowFrom` vs `users-registry`)

### Phase 1 — Observe Mode

- Parse `users.json`, evaluate policy, do not enforce.
- Add `/whoami` enrichment showing resolved user + roles.

### Phase 2 — Enforce for Commands

- Enforce command auth via users registry when enabled.
- Keep message intake policy fallback to legacy.

### Phase 3 — Enforce for DM/Group intake

- Enforce sender/group access from registry + policy.
- Retain emergency bypass toggles.

### Phase 4 — Tooling + UX

- CLI commands:
  - `openclaw users list`
  - `openclaw users add/remove`
  - `openclaw users validate`
  - `openclaw users whois <provider:id>`

---

## Config Additions (draft)

Primary idea (as requested): reference users registry from `openclaw.json`.

```json5
{
  users: {
    file: "~/.openclaw/users.json", // optional; if missing => legacy behavior
    mode: "observe", // observe | enforce
    fallbackToLegacyAllowlists: true,
    precedence: "users-first", // users-first | legacy-first
  },
}
```

### Load behavior contract

1. If `users.file` is **not set**:
   - OpenClaw behaves exactly as today (no behavior changes).

2. If `users.file` **is set** but file **does not exist / invalid JSON**:
   - `observe` mode: warn + continue legacy behavior.
   - `enforce` mode: configurable fail strategy:
     - recommended default: `failOpenToLegacy: true` (safe continuity)
     - optional strict mode: `failClosed: true` (deny until fixed)

3. If `users.file` exists and validates:
   - OpenClaw must load and obey the users policy according to `mode`.
   - `observe`: evaluate users policy but do not enforce, log diffs.
   - `enforce`: users policy becomes authoritative per precedence rules.

### Additional config knobs (planning)

```json5
{
  users: {
    file: "~/.openclaw/users.json",
    mode: "enforce",
    fallbackToLegacyAllowlists: true,
    precedence: "users-first",

    // startup/runtime behavior
    reloadOnChange: true,
    reloadDebounceMs: 500,

    // failure policy
    failOpenToLegacy: true,
    failClosed: false,

    // diagnostics
    logDecisions: true,
    logDecisionDiffs: true,
  },
}
```

### Operational expectation

- `users.file` is the switch from legacy-only access control to users-registry-aware control.
- No file = normal legacy behavior.
- Valid file + enforce = system obeys users policy for settings/authorization decisions.

---

## Security Requirements

1. Unknown user in enforce mode must be denied unless explicit open override.
2. `tools:elevated` requires explicit capability.
3. Role escalation must be config-controlled and auditable.
4. All deny decisions must emit machine-readable reason codes.

---

## Implementation Plan (code areas)

### New modules (proposed)

- `src/users/registry.ts` (load/validate/cache users.json)
- `src/users/identity-resolver.ts` (provider identity normalization + lookup)
- `src/users/authorizer.ts` (capability evaluation)
- `src/users/types.ts` (schema types)

### Existing modules to integrate

- `src/auto-reply/command-auth.ts`
- provider preflight/auth modules (telegram/whatsapp/discord/signal)
- config schema + docs:
  - `src/config/*` schema files
  - `docs/gateway/configuration-reference.md`
  - `docs/gateway/security/index.md`

---

## Testing Strategy

### Unit

- Identity normalization per provider
- Role + group inheritance
- Capability checks (allow/deny)
- Observe vs enforce behavior

### Integration

- Telegram DM authorized via users.json
- WhatsApp DM authorized via users.json
- Group membership gates
- commands.allowFrom coexistence and precedence

### Regression

- No users config present => no behavior change
- Existing allowFrom and pairing flows remain valid

---

## Risks

1. Identity normalization mismatches across providers
2. Confusing precedence between legacy allowlists and users roles
3. Over-broad role capability definitions

Mitigations:

- Observe mode first
- explicit precedence table in docs
- strict test matrix for both Telegram and WhatsApp

---

## Open Questions

1. Should users registry be global (`~/.openclaw/users.json`) or per-agent by default?
2. Should role capabilities cover only commands/tools first, or also routing decisions in v1?
3. How should pairing approvals interact with users registry (auto-create user vs temporary allow)?
4. Do we need channel-specific per-group role overrides in v1?

---

## Acceptance Criteria

1. OpenClaw can load and validate `users.json`.
2. In `observe` mode, logs include resolved user and decision source.
3. In `enforce` mode, command auth can be fully driven by roles from `users.json`.
4. Telegram + WhatsApp users with mapped identities are authorized consistently.
5. Legacy configs without `users` section continue to work unchanged.

---

## Suggested First Milestone (small, high value)

Deliver **Phase 1 + command-only enforcement flag**:

- users schema + loader
- identity resolution for Telegram + WhatsApp
- command auth integration in observe mode
- docs + tests

This gives immediate value with low break risk and creates the foundation for full multi-user policy.

---

## Salmita Alignment (Template-Driven Design)

This PRD is now explicitly aligned to Salmita’s existing user registry style.

Reference file found on host:

- `/home/xel/git/sages-openclaw/workspace-tulin/salma/salmita/usuarios.json`

What Salmita already models well:

- `usuarios[]` with human name, phone, optional `telegram_id`, groups, roles
- `permisos{}` role definitions with explicit `puede` and `no_puede`
- `grupos[]` with WhatsApp group ids (`...@g.us`)
- Strong WhatsApp-first reality (Telegram optional per user)

### Direct mapping from `usuarios.json` → proposed `users.json`

| Salmita field               | Proposed OpenClaw field               | Notes                              |
| --------------------------- | ------------------------------------- | ---------------------------------- |
| `usuarios[].nombre`         | `users[].displayName`                 | human label                        |
| `usuarios[].telefono`       | `users[].identities.whatsapp[]`       | canonical E.164                    |
| `usuarios[].telegram_id`    | `users[].identities.telegram[]`       | optional                           |
| `usuarios[].roles[]`        | `users[].roles[]`                     | role ids                           |
| `usuarios[].grupos[]`       | `users[].groupRefs[]`                 | names should resolve to stable ids |
| `grupos[].id`               | `groups[].channels.whatsapp.groups[]` | WhatsApp group id                  |
| `permisos.<rol>.puede[]`    | `roles.<rol>.capabilities.allow[]`    | allow grants                       |
| `permisos.<rol>.no_puede[]` | `roles.<rol>.capabilities.deny[]`     | explicit denies                    |

### Important adaptation

Salmita groups are name-based links from users (`grupos: ["Mantenimiento Salma"]`).
For OpenClaw, prefer stable group ids and keep display names separate to avoid rename breakage.

---

## WhatsApp-First Product Scope

Given your requirement, v1 and v2 should prioritize WhatsApp behavior first, then Telegram parity.

### v1 scope (WhatsApp-centric)

1. Resolve sender by E.164 first
2. Resolve group chat membership by `...@g.us`
3. Apply role capabilities from registry
4. Enforce command/tool permissions in DMs and groups
5. Keep Telegram support as optional mapped identity

### v2 scope

1. Deep Telegram parity for groups/topics
2. Multi-channel identity linking UI/CLI
3. Advanced policy inheritance (org/team/subteam)

---

## Proposed JSON Contract (Salmita-Compatible Variant)

```json
{
  "version": 1,
  "meta": {
    "description": "Usuarios y permisos multi-canal (WhatsApp-first)",
    "source": "salmita-compatible"
  },
  "groups": [
    {
      "id": "mtto-salma",
      "displayName": "Mantenimiento Salma",
      "channels": {
        "whatsapp": {
          "groups": ["120363422678028970@g.us"]
        }
      }
    }
  ],
  "roles": {
    "owner": {
      "description": "Acceso total",
      "capabilities": {
        "allow": ["*"],
        "deny": []
      }
    },
    "mantenimiento": {
      "description": "Operaciones de mantenimiento",
      "capabilities": {
        "allow": ["ver-unidades", "ver-fallas", "registrar-reparaciones"],
        "deny": ["ver-financieros", "exportar-datos-financieros"]
      }
    }
  },
  "users": [
    {
      "id": "juan",
      "displayName": "Juan",
      "enabled": true,
      "identities": {
        "whatsapp": ["+5218121937470"],
        "telegram": ["6586915095"]
      },
      "roles": ["owner"],
      "groupRefs": ["mtto-salma"]
    }
  ]
}
```

---

## Authorization Semantics (Detailed)

## Capability precedence

When evaluating a user action:

1. Aggregate all role `allow` capabilities
2. Aggregate all role `deny` capabilities
3. If `deny` contains capability (or wildcard family), deny wins
4. Else if `allow` contains `*` or exact capability, allow
5. Else deny (default)

## Channel + context gates

Authorization should pass **both**:

- Identity gate: user must resolve from sender identity
- Context gate: user must be allowed in that chat context
  - DM: policy for direct chats
  - Group: user must be member of at least one role/group policy that includes that group id

## Command families (proposal)

- `commands:status`, `commands:session`, `commands:model`, `commands:admin`
- `tools:standard`, `tools:elevated`
- `routing:bind`, `routing:override`
- `config:read`, `config:write`, `config:restart`

## Domain capabilities (Salmita-style, optional namespace)

Allow custom business permissions to coexist:

- `domain:ver-unidades`
- `domain:registrar-fallas`
- `domain:ver-financieros`
- etc.

This keeps compatibility with Salmita’s existing permission vocabulary.

---

## Conflict Resolution with Existing OpenClaw Config

In enforce mode, define deterministic precedence:

1. **Hard safety policy** (global security constraints) — always first
2. **users.json explicit deny**
3. **users.json explicit allow**
4. **legacy channel allowlist/pairing result** (if fallback enabled)
5. default deny

Config switch:

```json5
users: {
  enabled: true,
  mode: "enforce",
  fallbackToLegacyAllowlists: true,
  precedence: "users-first" // users-first | legacy-first
}
```

Recommended default: `users-first`.

---

## Data Quality Rules for `users.json`

1. `users[].id` unique (case-insensitive)
2. No duplicate identity across different users for same provider
3. WhatsApp numbers normalized to E.164
4. Telegram IDs normalized as numeric strings
5. All `groupRefs` must exist in `groups[]`
6. All assigned roles must exist in `roles{}`
7. Unknown fields rejected in enforce mode (optional warn in observe mode)

---

## Salmita Migration Blueprint (No code yet)

## Step 1: Ingest

- Read Salmita `usuarios.json`
- Build normalized in-memory model

## Step 2: Normalize

- Convert group names to stable ids
- Normalize numbers and telegram ids
- Merge duplicate people (if same number/telegram)

## Step 3: Validate

- Check role references
- Check group references
- Check duplicate identities

## Step 4: Emit

- Generate `users.json` compatible with OpenClaw PRD schema
- Keep trace block:
  - `meta.source = "salmita/usuarios.json"`
  - `meta.generatedAt`
  - `meta.mappingVersion`

## Step 5: Observe rollout

- Run users registry in observe mode
- Compare decisions with legacy allowlists
- Produce drift report

---

## Drift Report Spec (for rollout confidence)

For each inbound event, log:

- `senderRaw`
- `resolvedUserId`
- `context` (dm/group, provider, groupId)
- `legacyDecision` (allow/deny)
- `usersDecision` (allow/deny)
- `decisionDiff` (match|mismatch)
- `reasonCodes[]`

Rollout gate to enforce mode:

- mismatch rate < 1% over N events
- zero critical false-allow events

---

## Observability & Auditing Requirements

1. `/whoami` should include:
   - resolved user id
   - effective roles
   - effective capabilities summary
   - decision source (`users`, `legacy`, `mixed`)

2. Audit log event on every deny:
   - capability requested
   - role evaluation path
   - context policy mismatch (if any)

3. Config health command should report:
   - users file path
   - parse/validation status
   - last loaded timestamp
   - total users/groups/roles

---

## Extended Open Questions (WhatsApp-first)

1. Should one WhatsApp number be allowed to map to multiple users in shift scenarios?
2. Should group membership be explicit per user, or inherited only from role/group refs?
3. Should pairing approval auto-create a disabled user stub in observe mode?
4. Should owner role bypass context gates (groups) or only capability gates?
5. Should we support temporary roles with expiration (`expiresAt`) for contractors?

---

## Final PRD Direction

This PRD now targets a **Salmita-compatible, WhatsApp-first users registry** for OpenClaw, preserving existing OpenClaw routing/policy architecture while introducing:

- unified identities
- role-based capabilities with explicit deny support
- group-aware context control
- staged migration with drift measurement

No implementation is proposed yet in code; this document is intentionally planning-only and rollout-oriented.

---

## Organizational Role Taxonomy (Expanded)

This taxonomy is derived from your current Salmita role set and reframed for OpenClaw policy planning.

### Tier 0 — Platform Ownership

## `owner`

- Scope: global and unrestricted
- Typical holder: Juan / platform owner
- Capability profile:
  - `*` (all command/tool/admin/domain capabilities)
- Governance notes:
  - should be very small cardinality (1-2 users)
  - must be auditable and explicitly assigned

## `director`

- Scope: strategic + operational global access (nearly owner)
- Capability profile:
  - broad allow, potentially excluding irreversible platform actions depending on policy
- Governance notes:
  - can be modeled as owner-equivalent in small orgs
  - optional safeguard: no direct `config:write` in production without dual approval

### Tier 1 — Technical Operations

## `developer`

- Scope: engineering and technical diagnostics
- Capability profile:
  - allow: code/config/ops diagnostic capabilities
  - deny: finance-sensitive business data by default
- Salmita alignment:
  - maps to `codigo`, `git`, `skills`, `config-sistema`, and maintenance operational visibility

### Tier 2 — Business Operations (Domain)

## `mantenimiento`

- Scope: maintenance workflows
- allow examples:
  - fault/repair/diesel records, maintenance status/timing, maintenance cost visibility
- deny examples:
  - global financial reports, payroll/contracts, broad exports

## `trafico`

- Scope: route/shipment/traffic operations
- allow examples:
  - shipment visibility, mermas analysis, traffic reporting
- deny examples:
  - maintenance internals, sensitive finance, configuration/code

## `recursos-humanos`

- Scope: HR-supporting operational visibility (minimal)
- allow examples:
  - limited unit lookup and list-style summaries
- deny examples:
  - technical operations, maintenance workflows, finance and exports

### Tier 3 — Read-Only / Monitoring

## `monitoreo`

- Scope: minimal read-only operational awareness
- allow examples:
  - unit list, fault list, diesel snapshots
- deny examples:
  - mutations, financial visibility, admin/config operations

---

## Capability Family Taxonomy (Recommended)

To avoid ad-hoc permission growth, define capability namespaces:

1. `platform:*`
   - `platform:admin`, `platform:restart`, `platform:routing`, `platform:config:read|write`
2. `commands:*`
   - `commands:status`, `commands:session`, `commands:models`, `commands:acp`, `commands:admin`
3. `tools:*`
   - `tools:standard`, `tools:elevated`
4. `channel:*`
   - `channel:whatsapp:dm`, `channel:whatsapp:group:<id>`, `channel:telegram:dm`
5. `domain:*` (Salmita-compatible)
   - `domain:ver-unidades`, `domain:registrar-fallas`, etc.

### Why this split

- keeps OpenClaw platform permissions separate from business-domain permissions
- allows WhatsApp-first enforcement while preserving future Telegram parity
- enables clean audit traces by namespace

---

## Role Composition Matrix (Planning)

| Role             | platform:\* | commands:\* | tools:\*            | channel:\* | domain:\*              |
| ---------------- | ----------- | ----------- | ------------------- | ---------- | ---------------------- |
| owner            | full        | full        | elevated            | full       | full                   |
| director         | high        | full        | standard/elevated\* | full       | full                   |
| developer        | medium      | high        | standard            | scoped     | technical + ops subset |
| mantenimiento    | none        | medium      | none                | scoped     | maintenance subset     |
| trafico          | none        | medium      | none                | scoped     | traffic subset         |
| recursos-humanos | none        | low         | none                | scoped     | hr subset              |
| monitoreo        | none        | low         | none                | scoped     | read-only subset       |

`*` director elevated use is optional policy decision.

---

## Complete Planning Sample Generated from Current Salmita Dataset

Generated artifact (planning-only, no runtime wiring):

- `/home/xel/git/openclaw/users.sample.from-salmita.planning.json`

Generation source:

- `/home/xel/git/sages-openclaw/workspace-tulin/salma/salmita/usuarios.json`

Dataset snapshot in generated sample:

- users: 10
- roles: 7
- groups: 2

### Notes on generated sample

1. It preserves Salmita roles as role ids.
2. It maps `puede`/`no_puede` into:
   - `roles.<id>.capabilities.allow[]`
   - `roles.<id>.capabilities.deny[]`
3. Domain capabilities are prefixed as `domain:<permiso>`.
4. Group name refs were normalized into stable `groupRefs` ids.
5. Duplicate display names are disambiguated with unique `users[].id` generation.

---

## Planning Validation Checklist for the Generated Sample

Before any implementation phase, validate this artifact conceptually:

1. **Identity uniqueness**
   - no WhatsApp number assigned to multiple active users (unless intentionally shared)
2. **Role closure**
   - all user roles exist in `roles{}`
3. **Group closure**
   - all `groupRefs` exist in `groups[]`
4. **Deny precedence sanity**
   - explicit denies align with intended organizational constraints
5. **Owner/director review**
   - confirm privileged roles reflect real governance

---

## Deployment Planning Addendum (VPS + Docker)

Given production will run on a VPS with Docker, users policy planning must include deployment wiring:

1. OpenClaw runtime config includes `users.file` path.
2. Container must mount that file path from host.
3. Recommended rollout:
   - start in `observe`
   - compare drift
   - move to `enforce`
4. Keep rollback path:
   - switch to `observe` or remove `users.file` reference

Companion deployment planning doc created in Salmita repo:

- `/home/xel/git/sages-openclaw/workspace-tulin/salma/salmita/docs/openclaw/openclaw-vps-deploy-users-registry-plan.md`

## Planning-Only Next Refinement (no code)

If you want another planning pass, next artifact can be:

1. `users.taxonomy.v2.md`
   - final role glossary + escalation/delegation policy
2. `users.policy-matrix.v2.csv`
   - role × capability matrix in spreadsheet-friendly format
3. `users.migration.playbook.md`
   - rollout scripts/steps (still planning only, no implementation)
