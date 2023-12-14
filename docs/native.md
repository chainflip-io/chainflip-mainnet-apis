# Running the Binaries (Native)

## Pre-requisites
- JQ (https://stedolan.github.io/jq/download/)

## Setup
### Clone the repo

```bash
git clone https://github.com/chainflip-io/chainflip-mainnet-apis.git
cd chainflip-mainnet-apis
```
### Getting the Binaries

Update and upgrade your system:
```bash
sudo apt update
sudo apt upgrade -y
```

Import Chainflip's GPG key:
```bash
sudo gpg --keyserver hkp://keyserver.ubuntu.com --recv-keys 4E506212E4EF4E0D3E37E568596FBDCACBBCDD37
```

Verify the key's authenticity:
```bash
sudo gpg --export 4E506212E4EF4E0D3E37E568596FBDCACBBCDD37 | sudo gpg --show-keys
```

The output should look like this:
```bash
pub   rsa4096 2023-11-06 [SC]
    4E506212E4EF4E0D3E37E568596FBDCACBBCDD37
uid                      Chainflip Labs GmbH (Releaser Master Key) <security@chainflip.io>
sub   rsa4096 2023-11-06 [S] [expires: 2024-11-05]
sub   rsa4096 2023-11-06 [S] [expires: 2024-11-05]
sub   rsa4096 2023-11-06 [S] [expires: 2024-11-05]
```

Add apt repository:
```bash
sudo mkdir -p /etc/apt/trusted.gpg.d
sudo gpg --export 4E506212E4EF4E0D3E37E568596FBDCACBBCDD37 | sudo tee /etc/apt/trusted.gpg.d/chainflip-mainnet.gpg
echo "deb [arch=amd64 signed-by=/etc/apt/trusted.gpg.d/chainflip-mainnet.gpg] https://pkgs.chainflip.io/ubuntu/ jammy berghain" | sudo tee /etc/apt/sources.list.d/chainflip-berghain-mainnet.list
```

Download the binaries:
```bash
sudo apt update
sudo apt update
sudo apt install -y chainflip-cli chainflip-node chainflip-broker-api chainflip-lp-api
```

### Start the node

```bash
sudo systemctl enable --now chainflip-rpc-node
```

Check the status of the node:

```bash
sudo systemctl status chainflip-rpc-node
journalctl -u chainflip-rpc-node -f
```

If the node is synced you should see something like this:

```bash
Dec 14 12:00:00 chainflip-node[1234]: 2021-12-14 12:00:00  ✨ Imported #1234 (0x1234…)
```

### Generating Keys

> ⛔️ Please make sure you backup your keys. If you lose your keys, you will lose access to your funds. ⛔️

```bash
chainflip-cli generate-keys --json --path /etc/chainflip/keys > lp-keys.json
chainflip-cli generate-keys --json --path /etc/chainflip/keys > broker-keys.json
```

### Fund Accounts

> Note: The minimum funding amount for registering as a Broker or LP role is technically 1 FLIP. However, we recommend funding your accounts with at least 5 FLIP to account for transaction fees.

1. Get the public key of the Broker or LP account:

```bash
# Broker
cat broker-keys.json | jq -r '.signing_account_id'

# LP
cat lp-keys.json | jq -r '.signing_account_id'
```

2. Then head to the [Auctions Web App](https://auctions.chainflip.io/nodes)
3. Connect your wallet
4. Click "Add Node"
5. Follow the instructions to fund the account

### Running the APIs

#### Starting the Node and APIs

To start the Broker API and LP API, run the following command:

```bash
sudo systemctl enable --now chainflip-broker-api
```

Then you can check the by running:

```bash
sudo systemctl status chainflip-broker-api
journalctl -u chainflip-broker-api -f
```

Similarly, to start the LP API, run the following command:

```bash
sudo systemctl enable --now chainflip-lp-api
```

Then you can check the by running:

```bash
sudo systemctl status chainflip-lp-api
journalctl -u chainflip-lp-api -f
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
