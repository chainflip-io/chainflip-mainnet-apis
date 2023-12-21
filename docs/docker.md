# Docker Setup

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
docker run --platform=linux/amd64 --entrypoint=/usr/local/bin/chainflip-cli chainfliplabs/chainflip-cli:berghain-1.1.1 generate-keys --json > chainflip/lp-keys.json
docker run --platform=linux/amd64 --entrypoint=/usr/local/bin/chainflip-cli chainfliplabs/chainflip-cli:berghain-1.1.1 generate-keys --json > chainflip/broker-keys.json
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

#### Important Note

> 💡 Note: By default, the Node, LP and Broker APIs accept connection from localhost only. This is intentional for added security. However, if you wish to make requests to your LP/Broker API node from another host or over VPN for example, feel free to update the ports in the `docker-compose.yml` file to accept connections from any host.

> This can be achieved by removing the `127.0.0.1:` before the port number. For example:
```yaml
  lp:
    image: chainfliplabs/chainflip-lp-api:berghain
    pull_policy: always
    stop_grace_period: 5s
    stop_signal: SIGINT
    platform: linux/amd64
    restart: unless-stopped
    ports:
      - "10589:80" # <--- This is updated to accept connections from any host
    volumes:
      - ./chainflip/keys/lp:/etc/chainflip/keys
    entrypoint:
      - /usr/local/bin/chainflip-lp-api
    command:
      - --state_chain.ws_endpoint=ws://node:9944
    depends_on:
      - node
```

#### Starting the Node and APIs
Start by starting the node and wait for it to sync:
```bash
docker compose up node -d
docker compose logs -f
```
> 💡 Note: You know that your node is synced once you start seeing logs similar to the following:

```log
chainflip-mainnet-apis-node-1  | 2023-12-14 10:22:24 ✨ Imported #438968 (0x3fba…8e06)
chainflip-mainnet-apis-node-1  | 2023-12-14 10:22:28 ⏩ Block history, #26112 (8 peers), best: #438968 (0x3fba…8e06), finalized #438966 (0x99bd…0628), ⬇ 3.4MiB/s ⬆ 182.8kiB/s
```

Once the node is synced you can start the APIs:
```bash
docker compose up -d
docker compose logs -f
```

If you want to only start the Broker API, you can run:
```bash
docker compose up -d broker
docker compose logs -f broker
```

If you want to only start the LP API, you can run:
```bash
docker compose up -d lp
docker compose logs -f lp
```

### Interacting with the APIs
> Note: The following commands take a little while to respond because it submits and waits for finality.

#### Broker

Register a broker account:

```bash
curl -H "Content-Type: application/json" \
    -d '{"id":1, "jsonrpc":"2.0", "method": "broker_registerAccount"}' \
    http://localhost:10997
```

Request a swap deposit address:

```bash
curl -H "Content-Type: application/json" \
    -d '{"id":1, "jsonrpc":"2.0", "method": "broker_requestSwapDepositAddress", "params": ["ETH", "FLIP","0xabababababababababababababababababababab", 0]}' \
    http://localhost:10997
```

#### LP

Register an LP account:

```bash
curl -H "Content-Type: application/json" \
    -d '{"id":1, "jsonrpc":"2.0", "method": "lp_register_account", "params": [0]}' \
    http://localhost:10589
```
Register a liquidity refund address:

Before you can deposit liquidity, you need to register a liquidity refund address. This is the address that will receive your liquidity back when you withdraw it.

```bash
curl -H "Content-Type: application/json" \
    -d '{"id":1, "jsonrpc":"2.0", "method": "lp_register_liquidity_refund_address", "params": {"chain": "Ethereum", "address": "0xabababababababababababababababababababab"}}' \
    http://localhost:10589

```

Request a liquidity deposit address:

```bash
curl -H "Content-Type: application/json" \
    -d '{"id":1, "jsonrpc":"2.0", "method": "lp_liquidity_deposit", "params": ["ETH"]}' \
    http://localhost:10589
```

For more details please refer to the [Integrations documentation](https://docs.chainflip.io/integration/liquidity-provision/lp-api).
