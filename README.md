# GPT-Line Master Repository

This repository is the **master coordination repository** for the GPT-Line system.

It is intended to contain:

- the top-level product documentation
- the standalone specification files for each service
- Git submodules that point to the actual implementation repositories of each service

The actual code for each service should live in its own dedicated GitHub repository, and this master repository should include those repositories as submodules.

---

## Purpose of this repository

This repository exists to give you one central place that defines the full GPT-Line system and ties all service repositories together.

It should be used for:

- keeping the authoritative service specifications
- tracking the overall architecture
- linking all service repositories together through Git submodules
- helping developers understand which repo belongs to which service
- serving as the entry point for the whole project

This repository is **not** intended to contain the full implementation of all services directly inside it.

---

## GPT-Line services

The GPT-Line system is split into five separate services, each intended to live in its own repository:

1. `gpt-line-telephony`
2. `gpt-line-realtime-bridge`
3. `gpt-line-core-api`
4. `gpt-line-payments`
5. `gpt-line-admin`

---

## Specification documents currently in this repository

This repository contains separate repository-ready specification documents for the five GPT-Line services:

1. `gpt-line-telephony.md`
2. `gpt-line-realtime-bridge.md`
3. `gpt-line-core-api.md`
4. `gpt-line-payments.md`
5. `gpt-line-admin.md`

Each document is written so it can be handed directly to a developer or coding agent responsible for a single empty GitHub repository.

Each document includes:
- the service mission
- locked architectural decisions
- exact API contracts it consumes and/or exposes
- data model where relevant
- business rules
- security constraints
- required repo deliverables
- required tests
- definition of done

The agreements between services are repeated inside each document so the developer for one repo should not need to read the others to build a compatible implementation.

---

## Planned submodule structure

Once the service repositories are created, this master repository should include them as Git submodules, for example:

```text
/services/gpt-line-telephony
/services/gpt-line-realtime-bridge
/services/gpt-line-core-api
/services/gpt-line-payments
/services/gpt-line-admin
```

Each submodule should point to its own standalone GitHub repository.

---

## Recommended repository role for each submodule

### `gpt-line-telephony`

Owns:

* Asterisk
* IVR
* prompts
* DTMF handling
* caller routing
* handoff to GPT and payments

### `gpt-line-realtime-bridge`

Owns:

* live audio/session bridge
* OpenAI Realtime connection
* interruption handling
* warning/cutoff signaling

### `gpt-line-core-api`

Owns:

* caller accounts by phone number
* remaining seconds
* package catalog
* call authorization
* call finalization
* ledger
* admin backend APIs

### `gpt-line-payments`

Owns:

* package purchase flow
* PCI-safe payment orchestration
* CardCom charging
* payment callbacks
* applying credits through Core API

### `gpt-line-admin`

Owns:

* internal dashboard
* Firebase Authentication with Google Sign-In
* operator access control by email allowlist
* account management UI
* call monitoring UI
* payment inspection UI

---

## Suggested workflow

1. Keep the specification files in this master repository.
2. Create one GitHub repository per service.
3. Give each service document to the developer or coding agent responsible for that service repository.
4. Add each implementation repository back into this master repository as a Git submodule.
5. Use this repository as the central place for system-level coordination.

---

## Current contents

At this stage, this repository contains the specifications and project structure planning.

As development progresses, it should also contain:

* submodule references to each service repo
* top-level architectural notes
* deployment coordination notes
* shared operational documentation where useful

---

## Goal

The goal is that each service can be developed independently in its own repository, while this master repository remains the single top-level entry point that connects the full GPT-Line system together.
