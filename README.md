# GPT-Line Service Specifications Bundle

This archive contains separate repository-ready specifications for the five GPT-Line services:

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
