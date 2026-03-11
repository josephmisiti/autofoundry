# Autofoundry

Run any ML experiment script across GPUs on multiple cloud providers with a single command.

Autofoundry is a CLI companion to [Karpathy's autoresearch](https://github.com/karpathy/autoresearch). Point it at a shell script, pick your GPU configuration, and it handles the rest: provisioning instances, distributing experiment runs, streaming results live, and producing a final metrics report.

## Supported Providers

- **RunPod** — Secure and Community cloud
- **Vast.ai** — Global GPU marketplace
- **PRIME Intellect** — Decentralized GPU network
- **Lambda Labs** — On-demand cloud GPUs

## Quickstart

```bash
# Install
uv sync

# First run — configure API keys and SSH key path
uv run autofoundry run your_experiment.sh

# Subsequent runs use saved config
uv run autofoundry run your_experiment.sh --num 4 --gpu H100
```

On first run, Autofoundry walks you through configuring provider API keys and your SSH key path. Config is saved to `~/.config/autofoundry/config.toml`.

## How It Works

```
autofoundry run train.sh --num 4 --gpu H100
```

1. **Query providers** — Fetches real-time GPU pricing and availability across all configured providers
2. **Select GPUs** — Interactive table lets you pick offers and quantities
3. **Provision** — Spins up instances in parallel, waits for SSH ready
4. **Execute** — Uploads your script, distributes experiments round-robin across instances, streams output live
5. **Report** — Parses metrics from script output and displays best/mean/worst summary
6. **Teardown** — Terminates all instances on completion (or Ctrl-C)

## Experiment Scripts

Your script runs in `/workspace` on a GPU instance with CUDA available. To report metrics back to Autofoundry, print a `---` delimiter followed by `key: value` lines:

```bash
#!/usr/bin/env bash
set -e

# ... your training code ...

echo "---"
echo "val_loss: 0.42"
echo "accuracy: 91.3"
echo "training_seconds: 300"
```

Autofoundry aggregates these across all experiment runs in the final report.

## Pre-building Images

For faster provisioning, pre-bake your dependencies into a Docker image:

```bash
uv run autofoundry build scripts/setup_autoresearch.sh --tag youruser/autoresearch:latest
uv run autofoundry run scripts/run_autoresearch.sh --image youruser/autoresearch:latest
```

## Resuming Sessions

If a session is interrupted, resume it to restart stopped instances and run remaining experiments:

```bash
uv run autofoundry run script.sh --resume <session-id>
```

## CLI Reference

```
autofoundry run <script> [OPTIONS]

Arguments:
  script              Path to experiment shell script

Options:
  --num, -n           Number of experiment runs (default: 1)
  --gpu, -g           GPU type to search for (default: H100)
  --resume, -r        Resume a previous session
  --image, -i         Custom Docker image

autofoundry build <setup_script> [OPTIONS]

Arguments:
  setup_script        Path to setup shell script

Options:
  --tag, -t           Docker image tag (required)
  --base, -b          Base Docker image
```

## Architecture

```
cli.py           Entry point — run and build commands
planner.py       GPU offer querying and selection
provisioner.py   Instance lifecycle management
executor.py      SSH-based script upload and execution
reporter.py      Metrics aggregation and display
providers/       Provider API implementations
models.py        Data models (GpuOffer, InstanceConfig, Session)
config.py        TOML configuration management
state.py         SQLite session persistence
theme.py         Terminal styling
image_builder.py Docker image pre-building
```

## Requirements

- Python 3.11+
- SSH key pair (ed25519 or RSA)
- At least one provider API key (RunPod, Vast.ai, PRIME Intellect, or Lambda Labs)

## License

[MIT](LICENSE)
