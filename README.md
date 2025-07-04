# chainflip-mainnet-apis

## Pre-requisites
- Docker (https://docs.docker.com/get-docker/)
- JQ (https://stedolan.github.io/jq/download/)

## Setup
### Clone the repo

```bash
git clone https://github.com/chainflip-io/chainflip-mainnet-apis.git
cd chainflip-mainnet-apis
```

### Generating Keys

> ⛔️ Please make sure you backup your keys. If you lose your keys, you will lose access to your funds. ⛔️

```bash
mkdir -p ./chainflip/keys/lp
mkdir -p ./chainflip/keys/broker
docker run --platform=linux/amd64 --entrypoint=/usr/local/bin/chainflip-cli chainfliplabs/chainflip-cli:berghain-1.7.5 generate-keys --json > chainflip/lp-keys.json
docker run --platform=linux/amd64 --entrypoint=/usr/local/bin/chainflip-cli chainfliplabs/chainflip-cli:berghain-1.7.5 generate-keys --json > chainflip/broker-keys.json
cat chainflip/broker-keys.json | jq -r '.signing_key.secret_key' > chainflip/keys/broker/signing_key_file
cat chainflip/lp-keys.json | jq -r '.signing_key.secret_key' > chainflip/keys/lp/signing_key_file
```

### Fund Accounts

> Note: The minimum funding amount for registering as a Broker or LP role is technically 1 FLIP. However, we recommend funding your accounts with at least 5 FLIP to account for transaction fees.

1. Get the public key of the Broker or LP account:

```bash
# Broker
cat chainflip/broker-keys.json | jq -r '.signing_account_id'

# LP
cat chainflip/lp-keys.json | jq -r '.signing_account_id'
```

2. Then head to the [Auctions Web App](https://auctions.chainflip.io/nodes)
3. Connect your wallet
4. Click "Add Node"
5. Follow the instructions to fund the account

### Running the APIs


#### Accepting external connections

> 💡 Note: By default, the LP/Broker or RPC node accept connection from localhost only. This is intentional for added security. However, if you wish to make requests to your LP/Broker or RPC node from another host or over VPN for example, feel free to update the ports in the `docker-compose.yml` file to accept connections from any host.

> This can be achieved by removing the `127.0.0.1:` before the port number. For example:
```yaml
  node-with-lp:
    image: ghcr.io/chainflip-io/chainflip-backend/chainflip-node:latest
    pull_policy: always
    stop_grace_period: 5s
    stop_signal: SIGINT
    platform: linux/amd64
    restart: unless-stopped
    user: root
    ports:
      - "9944:9944" # <--- This is updated to accept connections from any host
    volumes:
      - ./chainflip/keys/lp:/etc/chainflip-keys/lp
    entrypoint: /bin/sh
    command: >
      -c '
      /usr/local/bin/chainflip-node key insert \
        --chain=/etc/chainflip/berghain.chainspec.json \
        --base-path=/etc/chainflip/chaindata \
        --suri=0x$$(cat /etc/chainflip-keys/lp/signing_key_file) \
        --key-type=lqpr \
        --scheme=sr25519 &&
      /usr/local/bin/chainflip-node \
        --base-path=/etc/chainflip/chaindata \
        --chain=/etc/chainflip/berghain.chainspec.json \
        --max-runtime-instances=32 \
        --rpc-cors=all \
        --rpc-methods=safe \
        --rpc-external \
        --sync=light-rpc \
        --blocks-pruning=128 \
        --state-pruning=128 \
        --log info,grandpa=error,runtime::grandpa=off,aura=error,babe=error,txpool::api=error
      '
    profiles:
      - lp
```

#### Starting the APIs

> 💡 Note: As of version 1.9, the LP and Broker APIs binaries are deprecated. The LP and Broker RPCs are now integrated into the node itself. You only need to run a node and inject your LP key to enable LP RPCs or Broker key to enable broker RPCs.

The docker-compose file is configured with three distinct profiles, each serving a specific purpose:
- `rpc-node`: Basic RPC node without LP or Broker functionality
- `broker`: RPC node with Broker RPCs enabled
- `lp`: RPC node with LP RPCs enabled

You can run any of these services by specifying the appropriate profile when starting the containers.

#### RPC Node

Start a basic RPC node and wait for it to sync:
```bash
# Start the RPC node in detached mode
docker compose --profile rpc-node up -d
# View the logs to monitor sync progress
docker compose --profile rpc-node logs -f
```

#### Broker

Start a node with Broker RPCs enabled (requires broker keys in `./chainflip/keys/broker`) and wait for it to sync:
```bash
docker compose --profile broker up -d
docker compose --profile broker logs -f
```

#### LP

Start a node with LP RPCs enabled (requires LP keys in `./chainflip/keys/lp`) and wait for it to sync:
```bash
docker compose --profile lp up -d
docker compose --profile lp logs -f
```

> 💡 Note: Your node is considered synced when you see logs similar to:
```log
chainflip-mainnet-apis-node-1  | 2023-12-14 10:22:24 ✨ Imported #438968 (0x3fba…8e06)
chainflip-mainnet-apis-node-1  | 2023-12-14 10:22:28 ⏩ Block history, #26112 (8 peers), best: #438968 (0x3fba…8e06), finalized #438966 (0x99bd…0628), ⬇ 3.4MiB/s ⬆ 182.8kiB/s
```

You can check the node's health by using this rpc call:
```
curl -H "Content-Type: application/json" \
    -d '{"id":1, "jsonrpc":"2.0", "method": "system_health"}' \
    http://localhost:9944
```

### Interacting with the APIs

> Note: The following commands take a little while to respond because it submits and waits for finality. If you get `Method not found` error make sure you are running the right profile and the appropriate key exists at the expected location either `./chainflip/keys/lp` for lp or `./chainflip/keys/broker` for broker.


#### Broker

Register a broker account:

```bash
curl -H "Content-Type: application/json" \
    -d '{"id":1, "jsonrpc":"2.0", "method": "broker_registerAccount"}' \
    http://localhost:9944
```

Request a swap deposit address:

```bash
curl -H "Content-Type: application/json" \
    -d '{"id":1, "jsonrpc":"2.0", "method": "broker_requestSwapDepositAddress", "params": ["ETH", "FLIP","0xabababababababababababababababababababab", 0]}' \
    http://localhost:9944
```

#### LP

Register an LP account:

```bash
curl -H "Content-Type: application/json" \
    -d '{"id":1, "jsonrpc":"2.0", "method": "lp_register_account", "params": [0]}' \
    http://localhost:9944
```
Register a liquidity refund address:

Before you can deposit liquidity, you need to register a liquidity refund address. This is the address that will receive your liquidity back when you withdraw it.

```bash
curl -H "Content-Type: application/json" \
    -d '{"id":1, "jsonrpc":"2.0", "method": "lp_register_liquidity_refund_address", "params": {"chain": "Ethereum", "address": "0xabababababababababababababababababababab"}}' \
    http://localhost:9944

```

Request a liquidity deposit address:

```bash
curl -H "Content-Type: application/json" \
    -d '{"id":1, "jsonrpc":"2.0", "method": "lp_liquidity_deposit", "params": ["ETH"]}' \
    http://localhost:9944
```

For more details please refer to the [Integrations documentation](https://docs.chainflip.io/integration/liquidity-provision/lp-api).
