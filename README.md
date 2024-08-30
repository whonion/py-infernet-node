[![pre-commit](https://github.com/ritual-net/infernet-node-internal/actions/workflows/workflow.yaml/badge.svg)](https://github.com/ritual-net/infernet-node-internal/actions/workflows/workflow.yaml)

# Infernet Node

The Infernet Node is the off-chain counterpart to the [Infernet SDK](https://github.com/ritual-net/infernet-sdk) from [Ritual](https://ritual.net), responsible for servicing compute workloads and delivering responses to on-chain smart contracts.

Developers can flexibily configure an Infernet Node for both on- and off-chain compute consumption, with extensible and robust parameterization at a per-container level.

> [!IMPORTANT]
> Infernet Node architecture, quick-start guides, and in-depth documentation can be found on the [Ritual documentation website](https://docs.ritual.net/infernet/node/introduction)

> [!WARNING]
> This software is being provided as is. No guarantee, representation or warranty is being made, express or implied, as to the safety or correctness of the software.

## Configuration

At its core, the Infernet Node operates according to a set of runtime configurations. Some of these configurations have sane defaults and do not need modification. See [config.sample.json](./config.sample.json) for an example, or copy to begin modifying:

```bash
# Copy and modify sample config
cp config.sample.json config.json
vim config.json
```

### Required configuration parameters

- **startup_wait** (`float`, _optional_). The number of seconds to wait for containers to start up before starting the node.
- **chain** (`object`). Chain configurations.
  - **enabled** (`boolean`). Whether chain is enabled on this node.
  - **trail_head_blocks** (`integer`). _if enabled_: how many blocks to stay behind head when syncing. Set to `0` to ignore.
  - **rpc_url** (`string`). _if enabled_: the HTTP(s) JSON-RPC url.
  - **registry_address** (`string`). _if enabled_: the `Registry` contract address.
  - **wallet** (`object`). _if enabled_:
    - **max_gas_limit** (`integer`). Maximum gas limit per node transaction
    - **private_key** (`string`). Node wallet private key
    - **payment_address** (`string`, _optional_). Public address of the node's escrow wallet. This is an instance of Infernet's `Wallet` contract. If not provided, the node will skip subscriptions that provide payment.
    - **allowed_sim_errors** (`array[string]`, _optional_). Allowed error messages to ignore when simulating transactions. Checks for inclusion in error message. Case-insensitive. i.e. `["out of gas"]` matches `"Contract reverted: Out of gas"`. Defaults to `[]`.
  - **snapshot_sync** (`object`, _optional_). Snapshot sync configurations.
    - **sleep** (`float`, _optional_). Number of seconds to sleep between snapshot syncs. Defaults to `1.0`.
    - **batch_size** (`int`, _optional_). Number of subscriptions to sync in each batch. Defaults to `200`.
    - **starting_sub_id** (`int`, _optional_). Subscription id to start the syncing process from. Defaults to `0`.
    - **sync_period** (`float`, _optional_). The syncing period in seconds, to monitor the chain for new blocks and subscriptions. Defaults to `0.5`.
- **docker** (`object`, _optional_). Docker credentials to pull private containers with
  - **username** (`string`). The Dockerhub username.
  - **password** (`string`). The Dockerhub [Personal Access Token](https://docs.docker.com/security/for-developers/access-tokens/) (PAT).
- **containers** (`array[container_spec]`, _optional_). Array of supported container specifications.
  - **container_spec** (`object`). Specification of supported container.
    - **id** (`string`). **Must be unique**. ID of supported service.
    - **image** (`string`). Dockerhub image to run container from. Must be local, public, or accessible with provided DockerHub credentials.
    - **command** (`string`, _optional_). The command and flags to run the container with.
    - **env** (`object`, _optional_). The environment variables to pass into the container.
    - **port** (`integer`, _optional_). Local port to expose this container on. _Auto-assigned if not specified_.
    - **external** (`boolean`, _optional_). Whether this container can be the first container in a [JobRequest](https://docs.ritual.net/infernet/node/api#jobrequest). Defaults to `true`.
    - **description** (`string`, _optional_). Description of service provided by this container.
    - **gpu** (`boolean`, , _optional_). Whether this should be a GPU-enabled container. Host must also be GPU-enabled. Defaults to `false`.
    - **volumes** (`array[string]`, _optional_). The volume mounts for this container.
    - **allowed_addresses** (`array[string]`, _optional_). Container-level firewall. Only specified addresses allowed to request execution of this container, with request originating from on-chain contract.
    - **allowed_delegate_addresses** (`array[string]`, _optional_). Container-level firewall. Only specified addresses allowed to request execution of this container, with request originating from on-chain contract but via off-chain delegate subscription (with this address corresponding to the delegate subscription `owner`).
    - **allowed_ips** (`array[string]`, _optional_). Container-level firewall. Only specified IPs and/or [CIDR blocks](https://www.ipaddressguide.com/cidr) allowed to request execution of this container.
    - **accepted_payments** (`dict[string, string]`, _optional_). Payment configurations for this container. This is a dictionary of accepted token addresses (zero address for native token i.e. ETH) and their corresponding minimum payment amount. If not provided, no payments will be received. If provided, subscriptions that don't meet these requirements will be skipped.
    - **generates_proofs** (`boolean`, _optional_). Whether this container generates proofs. Defaults to `false`. If `false`, the node will skip subscriptions that require proofs from this container.

### Sane default configuration parameters

- **log** (`object`, _optional_). Logging configurations
  - **path** (`string`, _optional_). The local file path for logging. Defaults to `infernet_node.log`.
  - **max_file_size** (`integer`, _optional_). The maximum file size for the log file in bytes. Defaults to `2^30` (i.e. `1 GB`).
  - **backup_count** (`integer`, _optional_). The number of latest historical log files to retain, before permanent deletion. Defaults to `2`.
- **server** (`object`). Server configurations.
  - **port** (`integer`). Port to run server on. Defaults to `4000`.
  - **rate_limit** (`object`, _optional_). REST server rate-limiting configurations. If not provided, defaults are used.
    - **num_requests** (`integer`, _optional_). Maximum number of requests per `period`. Defaults to `60`.
    - **period** (`float`, _optional_). Time period in seconds for allowing `num_requests`. Defaults to `60.0`.
- **redis** (`object`). Redis configurations.
  - **host** (`string`). Host to connect to. Defaults to `"redis"`.
  - **port** (`integer`). Port to connect to. Defaults to `6379`.
- **forward_stats** (`boolean`). Whether to send diagnostic system stats to Ritual. Defaults to `true`.
- **startup_wait** (`float`). Number of seconds to wait for containers to start up before starting the node.
  Defaults to `60`.

## Deployment

### Locally via Docker

```bash
# Set tag
tag="1.3.0"

# Build image from source
docker build -t ritualnetwork/infernet-node:$tag .

# Configure node
cd deploy
cp ../config.sample.json config.json
# FILL IN config.json #

# Run node and dependencies
docker compose up -d
```

### Locally via Docker (GPU-enabled)

The GPU-enabled version of the image comes pre-installed with the [NVIDIA CUDA Toolkit](https://developer.nvidia.com/cuda-toolkit?ref=blog.kobus.me). Using this image on your GPU-enabled machine enables the node to interact with the attached accelerators for diagnostic and purposes, such as heartbeat checks and utilization reports.

```bash
# Set tag
tag="1.3.0"

# Build GPU-enabled image from source
docker build -f Dockerfile-gpu -t ritualnetwork/infernet-node:$tag-gpu .

# Configure node
cd deploy
cp ../config.sample.json config.json
# FILL IN config.json #

# Run node and dependencies
docker compose -f docker-compose-gpu.yaml  up -d
```

### Locally via source

```bash
# Create and source new python venv
python3.11 -m venv env
source ./env/bin/activate
pip install -r requirements.txt

# Install dependencies
make install

# Configure node
cp config.sample.json config.json
# FILL IN config.json #

# Run node
make run
```

### Remotely via AWS / GCP

Follow README instructions in the [infernet-deploy](https://github.com/ritual-net/infernet-deploy) repository.

## Publishing a Docker image

```bash
# Set tag
tag="1.3.0"

# Build for local platform
make build

# Multi-platform build and push to repo
make build-multiplatform
```

## License

[BSD 3-clause Clear](./LICENSE)
