# Requirements: v1.1 Port Management

## Port Management

- [x] **PORT-01**: Skill asks the user whether to expose the service on a host port before assigning any port numbers (default: no — internal Docker network only)
- [x] **PORT-02**: If user opts out of host port binding, compose output omits the `ports:` entry — service is reachable only within the Docker network
- [x] **PORT-03**: If user opts in, skill scans all `.devtools/*/.*compose.yml` and `.devtools/*/.env` files to collect every host port already bound by installed services (main, UI, and monitoring ports)
- [x] **PORT-04**: If the user's chosen port matches any already-bound host port, skill names the conflict (e.g., "port 6379 is already used by redis") and prompts for a different port
- [x] **PORT-05**: Re-prompt loop continues until a non-conflicting port is confirmed, or the user opts out of host binding

## Future Requirements

- Runtime port availability check (`lsof`/`netstat`) to also catch non-Docker processes using a port

## Out of Scope

- Runtime port availability check via system tools — violates zero-dependency constraint; compose/env file scanning is sufficient and safe
- Updating ports for already-installed services — separate "update service" flow, not in scope for v1.1

## Traceability

| REQ-ID | Phase | Status |
|--------|-------|--------|
| PORT-01 | Phase 8 | Complete |
| PORT-02 | Phase 8 | Complete |
| PORT-03 | Phase 9 | Complete |
| PORT-04 | Phase 9 | Complete |
| PORT-05 | Phase 9 | Complete |
