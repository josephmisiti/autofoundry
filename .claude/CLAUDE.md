# Autofoundry

CLI tool to run ML experiment scripts across GPUs on multiple cloud providers.
`autofoundry run <script>` → pick GPUs → provision → execute → stream output → report metrics → teardown.

## Architecture

```
cli.py → planner.py → provisioner.py → executor.py → reporter.py
```

- **CLI**: `src/autofoundry/cli.py` — `run` and `build` commands via Typer
- **Models**: `src/autofoundry/models.py` — GpuOffer, InstanceConfig, InstanceInfo, Session
- **Config**: `src/autofoundry/config.py` — TOML config at `~/.config/autofoundry/config.toml`
- **Providers**: `src/autofoundry/providers/{runpod,vastai,primeintellect,lambdalabs}.py`
- **Theme**: `src/autofoundry/theme.py` — NGE-inspired terminal aesthetic ("sorties", "units", "supply lines")
- **State**: `src/autofoundry/state.py` — SQLite-backed session persistence (WAL mode)
- **Image Builder**: `src/autofoundry/image_builder.py` — Docker image build/push for pre-baking dependencies

## CLI Commands

- `autofoundry run <script>` — Main flow: configure → plan → provision → execute → report → teardown
  - `--num/-n`: Number of experiments
  - `--gpu/-g`: GPU type filter
  - `--resume/-r`: Resume a previous session
  - `--image/-i`: Custom Docker image
- `autofoundry build <setup_script>` — Pre-build Docker image with dependencies
  - `--tag/-t`: Docker image tag (required)
  - `--base/-b`: Base Docker image

## Provider API Details

### RunPod
- REST API at `https://rest.runpod.io/v1`, GraphQL at `https://api.runpod.io/graphql`
- REST uses `Authorization: Bearer` + `x-api-key` headers
- GraphQL MUST use `Authorization: Bearer` header (not `api_key` header) for user-scoped queries like `myself`
- Pod creation returns 201 (not 200) — check `status_code >= 300` for errors
- **SSH port mapping**: Container port 22 maps to a random host port. Read `pod["portMappings"]["22"]` for actual port. Do NOT hardcode port 22.
- **SSH key auth**: Keys must be registered in RunPod account settings via `updateUserSettings` GraphQL mutation. Env vars (`SSH_PUBLIC_KEY`, `PUBLIC_KEY`) alone are NOT sufficient.
- `ports` field in pod creation must be an array: `["22/tcp"]` not `"22/tcp"`
- GraphQL `gpuTypes` query does NOT support `maxGpuCount` field
- Provider image: `runpod/pytorch:1.0.2-cu1281-torch280-ubuntu2404`

### Vast.ai
- Response uses `"bundles"` key (not `"offers"`)
- Uses `api_key` as query param
- GPU name filtering must be client-side (substring match via `_find_gpu_variants`), not exact match
- Provider image: `runpod/pytorch:1.0.2-cu1281-torch280-ubuntu2404`

### PRIME Intellect
- API returns camelCase fields: `gpuType`, `gpuMemory`, `cloudId`, `prices.onDemand`
- `create_instance` requires nested `{"pod": {...}, "provider": {...}}` payload
- `stockStatus` values: "Low" counts as available; only exclude "", "out_of_stock", "unavailable"
- Null safety: use `str(item.get("field") or "")` for metadata dict values (Pydantic rejects None in `dict[str, str]`)
- Provider image: `cuda_12_4_pytorch_2_4`

### Lambda Labs
- REST API at `https://cloud.lambdalabs.com/api/v1` with Basic auth
- SSH key management: register or reuse existing keys
- Region-based availability (each region is a separate offer)
- Parses VRAM from instance description with regex
- Ubuntu-based images (pre-configured, not Docker)

## SSH Execution
- `executor.py` uses subprocess SSH/SCP (not asyncssh, despite the dependency)
- `BatchMode=yes` + `PasswordAuthentication=no` prevent password prompt hangs
- SSH retry loop (10 attempts, 5s delay) before upload — key auth can lag behind port availability
- Metadata passthrough: `GpuOffer.metadata` → `InstanceConfig.metadata` → provider's `create_instance`

## Session Persistence
- SQLite at `~/.config/autofoundry/sessions/{session_id}.db`
- Tables: session, instances, experiments, results, events
- Session states: CONFIGURING → PLANNING → PROVISIONING → RUNNING → REPORTING → COMPLETED/FAILED/PAUSED
- Resume support: `--resume` flag restarts stopped instances and runs pending experiments

## Test Scripts
- `scripts/run_autoresearch.sh` — Runs autoresearch (assumes pre-built image with deps)
- `scripts/run_autoresearch_full.sh` — Full setup + run from scratch
- `scripts/setup_autoresearch.sh` — Setup only (for image building)
- Script outputs `---` delimiter followed by `key: value` metrics, parsed by `executor.parse_metrics()`

## Status
- RunPod: fully working (provision, execute, stream, report, teardown) ✓
- Vast.ai: GPU listing works, provisioning untested
- PRIME Intellect: GPU listing works (inventory fluctuates), provisioning untested
- Lambda Labs: GPU listing works, provisioning untested
