<div align="center">

# ⛓️ Safrochain Mainnet Full Node & Validator Setup Guide

**A complete guide to running a Safrochain mainnet full node and registering as a validator**  
*Build from source, configuration, statesync, systemd service, and validator creation — step by step.*

[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04+-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![Safrochain](https://img.shields.io/badge/Safrochain-Mainnet-00C2A8?style=flat-square)](https://safrochain.com)
[![Version](https://img.shields.io/badge/Node%20Version-v0.2.2-brightgreen?style=flat-square)](https://docs.safrochain.com)
[![Chain ID](https://img.shields.io/badge/Chain%20ID-safrochain--1-blue?style=flat-square)](https://docs.safrochain.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Author:** HazenNetworkSolutions  
> **Network:** Safrochain Mainnet (Chain ID: safrochain-1)  
> **Version:** v0.2.2  
> **Last Updated:** June 2026

---

## Table of Contents

- [Hardware Requirements](#hardware-requirements)
- [Network Endpoints](#network-endpoints)
- [Step 1 — System Verification](#step-1--system-verification)
- [Step 2 — System Update and Dependencies](#step-2--system-update-and-dependencies)
- [Step 3 — Install Go](#step-3--install-go)
- [Step 4 — Build safrochaind from Source](#step-4--build-safrochaind-from-source)
- [Step 5 — Set Up Cosmovisor](#step-5--set-up-cosmovisor)
- [Step 6 — Initialize the Node](#step-6--initialize-the-node)
- [Step 7 — Download Genesis](#step-7--download-genesis)
- [Step 8 — Configure Peers (config.toml)](#step-8--configure-peers-configtoml)
- [Step 9 — Configure Custom Ports](#step-9--configure-custom-ports)
- [Step 10 — App-Level Settings (app.toml)](#step-10--app-level-settings-apptoml)
- [Step 11 — Set CLI Defaults (client.toml)](#step-11--set-cli-defaults-clienttoml)
- [Step 12 — Statesync (Optional)](#step-12--statesync-optional)
- [Step 13 — Create Systemd Service](#step-13--create-systemd-service)
- [Step 14 — Start the Node](#step-14--start-the-node)
- [Step 15 — Verify Sync](#step-15--verify-sync)
- [Step 16 — Create a Wallet](#step-16--create-a-wallet)
- [Step 17 — Register as a Validator](#step-17--register-as-a-validator)
- [Useful Commands](#useful-commands)

---

## Hardware Requirements

| Component        | Minimum          | Recommended      |
| ---------------- | ---------------- | ---------------- |
| Operating System | Ubuntu 22.04+    | Ubuntu 24.04     |
| CPU              | 4 cores          | 8+ cores         |
| RAM              | 16 GB            | 32 GB            |
| Disk             | 500 GB NVMe SSD  | 1 TB NVMe SSD    |
| Network          | 100 Mbps         | 1 Gbps           |

---

## Network Endpoints

| Type     | Endpoint                                  |
| -------- | ----------------------------------------- |
| RPC      | https://rpc.safrochain.network            |
| REST     | https://api.safrochain.network            |
| gRPC     | https://grpc.safrochain.network           |
| gRPC-Web | https://grpc-web.safrochain.network       |
| Status   | https://status.safrochain.network         |
| Docs     | https://docs.safrochain.com               |

---

## Step 1 — System Verification

After SSH-ing into your server, verify the system meets requirements:

```bash
lsb_release -a
uname -r
lscpu | grep -E "Model name|CPU\(s\)|Thread|Socket|Core"
free -h
df -h
```

---

## Step 2 — System Update and Dependencies

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git make jq build-essential wget curl htop tmux \
  lz4 gcc unzip screen btop iotop nethogs
```

---

## Step 3 — Install Go

Safrochain mainnet requires **Go 1.25.8**:

```bash
cd $HOME
wget https://go.dev/dl/go1.25.8.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.25.8.linux-amd64.tar.gz
rm go1.25.8.linux-amd64.tar.gz

[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> ~/.bash_profile
echo 'export GOPATH=$HOME/go' >> ~/.bash_profile
source $HOME/.bash_profile

[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

Verify the installation:

```bash
go version
```

Expected output:
```
go version go1.25.8 linux/amd64
```

---

## Step 4 — Build safrochaind from Source

Clone the official repository and build `v0.2.2`:

```bash
git clone https://github.com/Safrochain-Org/safrochain-node ~/safrochain-node
cd ~/safrochain-node
git fetch --tags
git checkout v0.2.2
make install
```

After a successful build you will see the install complete banner:

```
┌──────────────────────────────────────────────────────────────────
│   ✨  INSTALL COMPLETE
├──────────────────────────────────────────────────────────────────
│   ●  safrochaind   v0.2.2
│   ●  Cosmos SDK    v0.50.14
│   ●  CometBFT      v0.38.21
│   ●  Go runtime    go1.25.8 linux/amd64
│   ●  Build tags    netgo,ledger
└──────────────────────────────────────────────────────────────────
```

Verify:

```bash
safrochaind version
# safrochaind v0.2.2

safrochaind version --long | grep -E '^version:|^cosmos_sdk_version:|^go:|^build_tags:|^commit:'
```

Expected output:
```
build_tags: netgo,ledger
cosmos_sdk_version: v0.50.14
go: go version go1.25.8 linux/amd64
version: v0.2.2
```

---

## Step 5 — Set Up Cosmovisor

Install cosmovisor and configure the directory structure:

```bash
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.6.0
```

Create the cosmovisor upgrade directory and place the binary:

```bash
mkdir -p $HOME/.safrochain/cosmovisor/upgrades/v0.2.2/bin
cp $HOME/go/bin/safrochaind $HOME/.safrochain/cosmovisor/upgrades/v0.2.2/bin/
```

Create symlinks so cosmovisor points to the current version:

```bash
sudo ln -sfn $HOME/.safrochain/cosmovisor/upgrades/v0.2.2 $HOME/.safrochain/cosmovisor/current
sudo ln -sfn $HOME/.safrochain/cosmovisor/current/bin/safrochaind /usr/local/bin/safrochaind
```

Verify:

```bash
cosmovisor version
safrochaind version
# safrochaind v0.2.2
```

---

## Step 6 — Initialize the Node

Set your moniker and initialize the home directory:

```bash
MONIKER="YOUR_MONIKER"

safrochaind init "$MONIKER" --chain-id safrochain-1 --home ~/.safrochain

echo "export MONIKER=$MONIKER" >> $HOME/.bash_profile
echo "export SAFRO_CHAIN_ID=\"safrochain-1\"" >> $HOME/.bash_profile
echo "export SAFRO_HOME=\"$HOME/.safrochain\"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

> ℹ️ Replace `YOUR_MONIKER` with your own node display name.

---

## Step 7 — Download Genesis

Pull the canonical mainnet genesis and verify the SHA-256 checksum:

```bash
curl -L https://raw.githubusercontent.com/Safrochain-Org/mainnet-genesis/main/genesis.json \
  -o ~/.safrochain/config/genesis.json
```

Verify:

```bash
( cd ~/.safrochain/config && \
  echo "c05ac5aec1918df9edb257e8e0eea184d73edc51370eb4aa9f0b4f0aad615c4d  genesis.json" \
  | shasum -a 256 -c - )
# Expected: genesis.json: OK
```

> ⚠️ If the checksum does not match, do not continue. Re-download the genesis file.

---

## Step 8 — Configure Peers (config.toml)

Set the foundation seeds and P2P settings:

```bash
sed -i.bak 's|^seeds *=.*|seeds = "bc772fdc9749e6dfd200a9428f07d86fe4fd34ec@seed.safrochain.network:26666,d323d296ba55e89fb6ce1a724f8da1740bd8cbb0@seed2.safrochain.network:26670"|' "$SAFRO_HOME/config/config.toml"
sed -i 's|^persistent_peers *=.*|persistent_peers = ""|' "$SAFRO_HOME/config/config.toml"
sed -i 's|^addr_book_strict *=.*|addr_book_strict = true|' "$SAFRO_HOME/config/config.toml"
sed -i 's|^pex *=.*|pex = true|' "$SAFRO_HOME/config/config.toml"
```

Optional — advertise your public IP for inbound P2P:

```bash
# Replace <PUBLIC_IP> with your actual server IP
sed -i.bak -e "s|^external_address *=.*|external_address = \"<PUBLIC_IP>:26656\"|" "$SAFRO_HOME/config/config.toml"
```

Verify:

```bash
grep -E '^(seeds|persistent_peers|pex|addr_book_strict|external_address) ' "$SAFRO_HOME/config/config.toml"
```

---

## Step 9 — Configure Custom Ports

Setting a custom port prefix avoids conflicts when running multiple nodes on the same server.

Enter a **2-digit prefix** (e.g. `46`) and the commands below will rewrite every port in both `config.toml` and `app.toml`:

```bash
read -p "Enter your PORT prefix (2-digit, e.g. 46): " SAFRO_PORT
echo "export SAFRO_PORT=$SAFRO_PORT" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

Apply to `app.toml`:

```bash
sed -i.bak -e "s%:1317%:${SAFRO_PORT}317%g" \
           -e "s%:8080%:${SAFRO_PORT}080%g" \
           -e "s%:9090%:${SAFRO_PORT}090%g" \
           -e "s%:9091%:${SAFRO_PORT}091%g" \
           -e "s%:8545%:${SAFRO_PORT}545%g" \
           -e "s%:8546%:${SAFRO_PORT}546%g" \
           -e "s%:6065%:${SAFRO_PORT}065%g" \
           "$SAFRO_HOME/config/app.toml"
```

Apply to `config.toml`:

```bash
sed -i.bak -e "s%:26658%:${SAFRO_PORT}658%g" \
           -e "s%:26657%:${SAFRO_PORT}657%g" \
           -e "s%:6060%:${SAFRO_PORT}060%g" \
           -e "s%:26656%:${SAFRO_PORT}656%g" \
           -e "s%^external_address *=.*%external_address = \"$(wget -qO- eth0.me):${SAFRO_PORT}656\"%" \
           -e "s%:26660%:${SAFRO_PORT}660%g" \
           "$SAFRO_HOME/config/config.toml"
```

Verify:

```bash
grep -E ":(${SAFRO_PORT})" "$SAFRO_HOME/config/config.toml" | head -5
grep -E ":(${SAFRO_PORT})" "$SAFRO_HOME/config/app.toml" | head -5
```

---

## Step 10 — App-Level Settings (app.toml)

Apply the recommended app settings:

```bash
# Minimum gas price
sed -i.bak 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.05usaf"|' "$SAFRO_HOME/config/app.toml"
```

Additional settings via sed:

```bash
# Prometheus metrics (config.toml)
sed -i -e "s/prometheus = false/prometheus = true/" "$SAFRO_HOME/config/config.toml"

# Disable indexer (saves disk)
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" "$SAFRO_HOME/config/config.toml"

# Pruning
sed -i -e "s/^pruning *=.*/pruning = \"default\"/" "$SAFRO_HOME/config/app.toml"
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" "$SAFRO_HOME/config/app.toml"
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"10\"/" "$SAFRO_HOME/config/app.toml"
```

Full recommended `app.toml` block:

```toml
minimum-gas-prices = "0.05usaf"

[api]
enable = true
swagger = false
address = "tcp://0.0.0.0:1317"

[grpc]
enable = true
address = "0.0.0.0:9090"

[grpc-web]
enable = true
address = "0.0.0.0:9091"

[telemetry]
enabled = true
prometheus-retention-time = 60

# use "nothing" for archive nodes
pruning             = "default"
pruning-keep-recent = "100"
pruning-interval    = "10"
```

---

## Step 11 — Set CLI Defaults (client.toml)

```bash
safrochaind config set client chain-id safrochain-1 --home "$SAFRO_HOME"
safrochaind config set client node tcp://127.0.0.1:${SAFRO_PORT}657 --home "$SAFRO_HOME"
safrochaind config set client keyring-backend file --home "$SAFRO_HOME"
safrochaind config set client broadcast-mode sync --home "$SAFRO_HOME"
safrochaind config set client output json --home "$SAFRO_HOME"
```

---

## Step 12 — Statesync (Optional)

If you want to speed up the initial sync by skipping block replay, statesync is available. See the official statesync guide at:  
https://docs.safrochain.com/run-a-node/statesync

---

## Step 13 — Create Systemd Service

```bash
sudo bash -c "cat > /etc/systemd/system/safrochaind.service" << EOF
[Unit]
Description=safrochaind node service
After=network-online.target
Wants=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.safrochain"
Environment="DAEMON_NAME=safrochaind"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.safrochain/cosmovisor/current/bin"
Environment="LD_LIBRARY_PATH=$HOME/.safrochain"

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable safrochaind
```

---

## Step 14 — Start the Node

```bash
sudo systemctl start safrochaind
sudo journalctl -u safrochaind -f --no-pager -o cat
```

Verify the service is running:

```bash
sudo systemctl status safrochaind --no-pager
```

Expected output:
```
● safrochaind.service - Safrochain mainnet node
     Active: active (running) since ...
```

---

## Step 15 — Verify Sync

```bash
curl -s http://127.0.0.1:${SAFRO_PORT}657/status \
  | jq '.result.sync_info | {latest_block_height, catching_up}'
```

Compare against the public RPC:

```bash
curl -s https://rpc.safrochain.network/status \
  | jq '.result.sync_info.latest_block_height'
```

> ⚠️ Wait until `catching_up` is `false` before creating a validator.

---

## Step 16 — Create a Wallet

```bash
safrochaind keys add wallet --keyring-backend file --home ~/.safrochain
```

> ⚠️ **CRITICAL:** Save your mnemonic phrase in a secure location. Without it, you cannot recover your wallet.

To recover an existing wallet:

```bash
safrochaind keys add wallet --recover --keyring-backend file --home ~/.safrochain
```

List wallets and get your address:

```bash
safrochaind keys list --keyring-backend file --home ~/.safrochain
```

Check your balance:

```bash
safrochaind query bank balances $(safrochaind keys show wallet -a --keyring-backend file --home ~/.safrochain) \
  --node https://rpc.safrochain.network:443
```

---

## Step 17 — Register as a Validator

> The node must be **fully synced** (`catching_up: false`) before creating a validator.

### Get your consensus public key:

```bash
safrochaind comet show-validator --home ~/.safrochain
```

### Create validator JSON:

```bash
cat > $HOME/validator.json << EOF
{
  "pubkey": $(safrochaind comet show-validator --home ~/.safrochain),
  "amount": "1000000usaf",
  "moniker": "YOUR_MONIKER",
  "identity": "YOUR_KEYBASE_ID",
  "website": "https://your-website.com",
  "security": "your@email.com",
  "details": "Your validator description",
  "commission-rate": "0.10",
  "commission-max-rate": "0.20",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1"
}
EOF
```

### Submit the transaction:

```bash
safrochaind tx staking create-validator $HOME/validator.json \
  --from wallet \
  --chain-id safrochain-1 \
  --gas auto \
  --gas-adjustment 1.4 \
  --fees 500usaf \
  --keyring-backend file \
  --home ~/.safrochain \
  -y
```

### Verify your validator:

```bash
safrochaind query staking validator \
  $(safrochaind keys show wallet --bech val -a --keyring-backend file --home ~/.safrochain) \
  --node https://rpc.safrochain.network:443
```

---

## Useful Commands

### Service Management

```bash
sudo systemctl start safrochaind
sudo systemctl stop safrochaind
sudo systemctl restart safrochaind
sudo systemctl status safrochaind
```

### Logs & Sync

```bash
# Live logs
sudo journalctl -u safrochaind -f --no-pager -o cat

# Logs from last hour
sudo journalctl -u safrochaind --since "1 hour ago"

# Sync status
curl -s http://127.0.0.1:${SAFRO_PORT}657/status | jq '.result.sync_info | {latest_block_height, catching_up}'

# Connected peers
curl -s http://127.0.0.1:${SAFRO_PORT}657/net_info | jq '.result.n_peers'
```

### Wallet & Balance

```bash
# List wallets
safrochaind keys list --keyring-backend file --home ~/.safrochain

# Show address
safrochaind keys show wallet -a --keyring-backend file --home ~/.safrochain

# Check balance
safrochaind query bank balances $(safrochaind keys show wallet -a --keyring-backend file --home ~/.safrochain) \
  --node https://rpc.safrochain.network:443
```

### Staking

```bash
# Delegate tokens
safrochaind tx staking delegate \
  $(safrochaind keys show wallet --bech val -a --keyring-backend file --home ~/.safrochain) 1000000usaf \
  --from wallet --chain-id safrochain-1 \
  --gas auto --gas-adjustment 1.4 --fees 500usaf \
  --node tcp://127.0.0.1:${SAFRO_PORT}657 \
  --keyring-backend file --home ~/.safrochain -y

# Redelegate tokens
safrochaind tx staking redelegate \
  $(safrochaind keys show wallet --bech val -a --keyring-backend file --home ~/.safrochain) \
  <NEW_VALOPER> 1000000usaf \
  --from wallet --chain-id safrochain-1 \
  --gas auto --gas-adjustment 1.4 --fees 500usaf \
  --node tcp://127.0.0.1:${SAFRO_PORT}657 \
  --keyring-backend file --home ~/.safrochain -y

# Undelegate tokens
safrochaind tx staking unbond \
  $(safrochaind keys show wallet --bech val -a --keyring-backend file --home ~/.safrochain) 1000000usaf \
  --from wallet --chain-id safrochain-1 \
  --gas auto --gas-adjustment 1.4 --fees 500usaf \
  --node tcp://127.0.0.1:${SAFRO_PORT}657 \
  --keyring-backend file --home ~/.safrochain -y
```

### Rewards

```bash
# Withdraw all rewards
safrochaind tx distribution withdraw-all-rewards \
  --from wallet --chain-id safrochain-1 \
  --gas auto --gas-adjustment 1.4 --fees 500usaf \
  --node tcp://127.0.0.1:${SAFRO_PORT}657 \
  --keyring-backend file --home ~/.safrochain -y

# Withdraw commission
safrochaind tx distribution withdraw-rewards \
  $(safrochaind keys show wallet --bech val -a --keyring-backend file --home ~/.safrochain) \
  --commission \
  --from wallet --chain-id safrochain-1 \
  --gas auto --gas-adjustment 1.4 --fees 500usaf \
  --node tcp://127.0.0.1:${SAFRO_PORT}657 \
  --keyring-backend file --home ~/.safrochain -y
```

### Governance

```bash
# List proposals
safrochaind query gov proposals --node https://rpc.safrochain.network:443

# Vote on a proposal
safrochaind tx gov vote 1 yes \
  --from wallet --chain-id safrochain-1 \
  --gas auto --gas-adjustment 1.4 --fees 500usaf \
  --node tcp://127.0.0.1:${SAFRO_PORT}657 \
  --keyring-backend file --home ~/.safrochain -y
```

### Validator Operations

```bash
# Edit validator
safrochaind tx staking edit-validator \
  --new-moniker "NEW_MONIKER" \
  --identity "YOUR_KEYBASE_ID" \
  --website "https://your-website.com" \
  --security-contact "your@email.com" \
  --details "Your validator description" \
  --from wallet --chain-id safrochain-1 \
  --gas auto --gas-adjustment 1.4 --fees 500usaf \
  --node tcp://127.0.0.1:${SAFRO_PORT}657 \
  --keyring-backend file --home ~/.safrochain -y

# Unjail validator
safrochaind tx slashing unjail \
  --from wallet --chain-id safrochain-1 \
  --gas auto --gas-adjustment 1.4 --fees 500usaf \
  --node tcp://127.0.0.1:${SAFRO_PORT}657 \
  --keyring-backend file --home ~/.safrochain -y

# Check signing info
safrochaind query slashing signing-info \
  $(safrochaind comet show-validator --home ~/.safrochain) \
  --node https://rpc.safrochain.network:443
```

### Firewall

```bash
# P2P — must be open to the public
sudo ufw allow ${SAFRO_PORT}656/tcp comment "safrochain P2P"

# RPC — open only if you serve public endpoints
sudo ufw allow ${SAFRO_PORT}657/tcp comment "safrochain RPC"

# REST API
sudo ufw allow ${SAFRO_PORT}317/tcp comment "safrochain REST"

# gRPC
sudo ufw allow ${SAFRO_PORT}090/tcp comment "safrochain gRPC"
```

---

## About the Author

This guide was prepared by **HazenNetworkSolutions**.  
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
