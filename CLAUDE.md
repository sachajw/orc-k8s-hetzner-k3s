# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**hetzner-k3s** is a CLI tool written in Crystal that creates production-ready Kubernetes clusters on Hetzner Cloud in 2-3 minutes. It provisions k3s clusters with automatic installation of Cloud Controller Manager, CSI driver, System Upgrade Controller, and Cluster Autoscaler.

## Build Commands

```bash
# Install dependencies
shards install

# Build (development)
crystal build src/hetzner-k3s.cr

# Build (release, for macOS - requires OpenSSL 3)
brew install openssl@3
export LDFLAGS="-L/usr/local/opt/openssl@3/lib"
export CPPFLAGS="-I/usr/local/opt/openssl@3/include"
crystal build src/hetzner-k3s.cr --release

# Build (release, static binary for Linux - use Alpine)
crystal build src/hetzner-k3s.cr --release --static
```

## Testing

No unit test suite exists. E2E tests are in `e2e-tests/`:

```bash
# Run single e2e test
cd e2e-tests
cp env.sample env  # Edit with your Hetzner API token
./run-single-test.sh config-sshport-image.yaml IMAGE=ubuntu-22.04 SSHPORT=222

# Run all e2e tests (sequential, respects Hetzner quotas)
./run-all-tests.sh

# View test results
./list-test-results.sh
```

## Architecture

```
src/
├── hetzner-k3s.cr              # CLI entry point (Admiral framework)
├── configuration/
│   ├── loader.cr               # YAML config loading and validation
│   ├── main.cr                 # Main configuration model
│   ├── models/                 # Node pools, networking, addons, datastore
│   └── validators/             # Config validation rules
├── cluster/
│   ├── create.cr               # Cluster creation orchestration
│   ├── delete.cr               # Cluster deletion
│   ├── upgrade.cr              # k3s version upgrades
│   ├── run.cr                  # Execute commands on nodes
│   ├── instance_builder.cr     # Server instance creation
│   ├── firewall_manager.cr     # Hetzner firewall rules
│   ├── network_manager.cr      # Private network setup
│   └── load_balancer_manager.cr
├── hetzner/
│   └── client.cr               # Hetzner Cloud API client (crest HTTP)
├── kubernetes/
│   ├── installer.cr            # k3s installation orchestrator
│   ├── kubeconfig_manager.cr   # Kubeconfig generation
│   ├── control_plane/          # Master node setup
│   ├── worker/                 # Worker node setup
│   └── software/               # CNI, CSI, autoscaler components
└── templates/                  # Cloud-init and shell script templates (Crinja)
```

## Key Patterns

**Concurrency**: Uses Crystal fibers and channels for parallel instance creation and k3s installation. Master/worker installation queues process nodes concurrently.

**Configuration**: YAML-based with comprehensive validation. Models use Crystal's YAML serialization macros.

**API Client**: `hetzner/client.cr` wraps Hetzner Cloud REST API with retry logic (retriable gem) and caching for locations/instance types.

**Templates**: Crinja templates in `templates/` generate cloud-init scripts and shell installation scripts.

## CLI Commands

```bash
hetzner-k3s create --config cluster.yaml    # Create cluster
hetzner-k3s delete --config cluster.yaml    # Delete cluster
hetzner-k3s upgrade --config cluster.yaml --new-k3s-version v1.32.0+k3s1
hetzner-k3s releases                        # List available k3s versions
hetzner-k3s run --config cluster.yaml --command "uptime"  # Run on all nodes
```

## Key Dependencies

- **admiral**: CLI framework for command routing
- **crest**: HTTP client for Hetzner API
- **crinja**: Template engine for installation scripts
- **retriable**: Retry mechanism for API resilience
- **tasker**: Task/worker pool for concurrency
