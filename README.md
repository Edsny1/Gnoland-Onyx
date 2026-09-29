# Gnoland onyx Validator Node Setup

Full guide to run a validator node on the **gno.land onyx** testnet (`onyx-1`).

> **Quick start:** Use the automated setup script — `bash scripts/setup.sh`

Onyx launched on **2026-09-28T00:00:00Z**. It is a fresh chain (not a hardfork of pearl) that runs mainnet's code, one release candidate ahead, and is upgraded whenever mainnet is.

> **Pearl is over.** Do not reuse pearl chain data on onyx. If pearl is still running on the same server, stop it first (or use a separate `GNO_DIR` and different ports).

---

## Auto Install

```bash
wget -O setup.sh https://raw.githubusercontent.com/Edsny1/Gnoland-Onyx/Edsny/scripts/setup.sh
chmod +x setup.sh
bash setup.sh
```

Or with curl:

```bash
curl -o setup.sh https://raw.githubusercontent.com/Edsny1/Gnoland-Onyx/Edsny/scripts/setup.sh
chmod +x setup.sh
bash setup.sh
```

---

## Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| OS | Ubuntu 22.04+ | Ubuntu 24.04 |
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Disk | 200 GB SSD | 500 GB NVMe |
| Go | version required by `go.mod` of the pinned tag | latest |

---

## Version pinning (important)

Binaries are no longer built from a `chain/<name>` branch. Run the version in the **last row** of the onyx upgrade ledger and pin it:

**https://github.com/gnolang/gno/blob/chain/mainnet/misc/deployments/onyx.gno.land/UPGRADES.md**

At launch this is **`v1.5.0`**. The binary changes at every coordinated upgrade, and a node refuses to start a newer version before its halt height. Never build from `master` or from a branch tip — such a binary reaches no consensus with the network.

The examples below use `v1.5.0`; replace it with the last row of the ledger when it changes.

---

## Manual Setup

### 1. Install dependencies

```bash
sudo apt update && sudo apt install -y git make wget curl python3 build-essential
```

Install Go (skip if your installed version satisfies the `go` line in the tag's `go.mod`):

```bash
GO_VERSION="1.24.6"
wget -q "https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go${GO_VERSION}.linux-amd64.tar.gz"
rm "go${GO_VERSION}.linux-amd64.tar.gz"

echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc
```

### 2. Get the binaries

`GNOROOT` must point at a checkout of the **same tag** — the node reads `gnovm/stdlibs` from it.

```bash
git clone --branch v1.5.0 --depth 1 https://github.com/gnolang/gno.git ~/gno
export GNOROOT=~/gno
```

**Option A — build from the tag:**

```bash
cd ~/gno
make -C gno.land install.gnoland install.gnokey
gnoland version    # must print v1.5.0
```

**Option B — native release binary (recommended by the official docs for validators):**

```bash
curl -fsSLO https://github.com/gnolang/gno/releases/download/v1.5.0/gnoland_linux_amd64
curl -fsSLO https://github.com/gnolang/gno/releases/download/v1.5.0/CHECKSUMS.txt
shasum -a 256 gnoland_linux_amd64        # must match the line in CHECKSUMS.txt
chmod +x gnoland_linux_amd64
sudo install -m 0755 gnoland_linux_amd64 /usr/local/bin/gnoland
gnoland version                          # must print v1.5.0

# gnokey is still built from the tag
make -C ~/gno/gno.land install.gnokey
```

**Container image (not recommended for validators):** `ghcr.io/gnolang/gno/gnoland:v1.5.0`. It adds a base image to trust, a registry that must be reachable on restart, a root daemon, and one more way to end up with **two instances of your validator signing at once** (restart policy, `compose up` on a second host, a forgotten container). If you still use it: never `latest`, never a restart policy, and mount `secrets/` read-only.

---

### 3. Download and verify genesis

```bash
cd ~/gno

wget -O genesis.json \
  https://github.com/gnolang/gno/releases/download/chain/onyx/genesis.json

# Verify SHA256 — must match exactly
shasum -a 256 genesis.json
# expected: 4b006fd7ccdec052865accc84dd29b2b76f8b57b2560789a15eedaa88f0e26c5
```

---

### 4. Initialize config and keys

```bash
cd ~/gno
gnoland config init
gnoland secrets init
```

---

### 5. Configure node

Apply required chain-wide settings:

```bash
# Persistent peers (required)
gnoland config set p2p.persistent_peers \
  "g1x5mlj5ava0dw9vkf4j6admjlzswm6f06p44krn@seed-1.onyx.testnets.gno.land:26656,g1grq5zswt0dlwwe7clr4359w70k2ewgse0gcwck@seed-2.onyx.testnets.gno.land:26656"

# Chain-wide consensus settings (must match exactly)
gnoland config set application.prune_strategy syncable
gnoland config set consensus.timeout_commit 3s
gnoland config set consensus.peer_gossip_sleep_duration 10ms
gnoland config set p2p.flush_throttle_timeout 10ms

# Performance (advised)
gnoland config set mempool.size 10000
gnoland config set p2p.max_num_outbound_peers 40
```

Set your node-specific values:

```bash
gnoland config set moniker "YOUR-NODE-NAME"
gnoland config set p2p.external_address "YOUR-SERVER-IP:26656"
gnoland config set p2p.pex true
```

Running a sentry-node setup instead of a standalone node? See the [Sentry-node architecture](https://github.com/gnolang/gno/blob/master/gno.land/cmd/gnoland/README.md#sentry-node-architecture) section of the `gnoland` README.

#### Using custom ports (17xxx)

If port 26xxx is already in use on your server:

```bash
gnoland config set p2p.laddr "tcp://0.0.0.0:17656"
gnoland config set rpc.laddr "tcp://127.0.0.1:17657"
gnoland config set telemetry.prometheus_listen_addr ":17660"
```

> Update `p2p.external_address` to match your P2P port.

---

### 6. Create systemd service

```bash
sudo tee /etc/systemd/system/gnoland.service > /dev/null <<EOF
[Unit]
Description=Gnoland onyx Node
After=network-online.target
Wants=network-online.target

[Service]
User=$USER
WorkingDirectory=$HOME/gno
Environment=GNOROOT=$HOME/gno
Environment=HOME=$HOME
ExecStart=$(which gnoland) start \
  --chainid onyx-1 \
  --genesis $HOME/gno/genesis.json \
  --skip-genesis-sig-verification
Restart=on-failure
RestartSec=5s
LimitNOFILE=65535
StandardOutput=journal
StandardError=journal
SyslogIdentifier=gnoland

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable gnoland
sudo systemctl start gnoland
```

> `--skip-genesis-sig-verification` is required: some genesis transactions
> carry placeholder/intentionally-invalidated signatures (e.g. the
> `names.Enable` call runs with a patched caller), so the node panics on
> startup without it.

Check logs:

```bash
sudo journalctl -u gnoland -f
```

---

### 7. Sync and coordinated upgrades

There is **no community snapshot for onyx** yet; the node syncs from genesis. Wait until it has caught up to the chain tip before registering.

**Syncing from genesis stops at every past upgrade.** Each coordinated halt listed in `UPGRADES.md` fires again during replay: the node stops after committing that halt height. Restart it — with the same binary if it satisfies that row's `halt_min_version`, otherwise with that row's version — and it continues.

If a restart lands between a halt proposal's execution and its halt height while your binary already satisfies that halt's version, the node refuses to start; set `skip_upgrade_height` to that height in `config.toml` for that one restart, or use the previous version until the height.

At launch the ledger has a single row (`v1.5.0`, genesis), so no halts are expected yet.

Check status:

```bash
# Replace 17657 with 26657 if you're using default ports
curl -s http://127.0.0.1:17657/status | python3 -c "
import sys, json
d = json.load(sys.stdin)
info = d['result']['sync_info']
print('Latest block :', info['latest_block_height'])
print('Catching up  :', info['catching_up'])
"
```

Wait until `catching_up: False` before proceeding.

---

### 8. Add operator wallet

Create a new wallet:

```bash
gnokey add YOUR-KEY-NAME
```

Or recover from existing mnemonic:

```bash
gnokey add YOUR-KEY-NAME --recover
```

Get your operator address:

```bash
gnokey list
```

---

### 9. Get testnet GNOT

Request tokens for your `g1...` operator address from the faucet:

**https://onyx.testnets.gno.land/faucet**

Verify balance:

```bash
gnokey query \
  -remote "https://rpc.onyx.testnets.gno.land" \
  auth/accounts/YOUR-G1-ADDRESS
```

---

### 10. Register as validator candidate

Get your consensus public key:

```bash
gnoland secrets get validator_key
# Note the gpub1... value
```

Register on the valoper realm (must be signed by the operator key — the
realm rejects the call if the signer doesn't control the operator address):

```bash
gnokey maketx call \
  --pkgpath gno.land/r/gnops/valopers \
  --func Register \
  --args "YOUR-MONIKER" \
  --args "YOUR-DESCRIPTION" \
  --args "data-center" \
  --args "YOUR-G1-OPERATOR-ADDRESS" \
  --args "YOUR-GPUB1-CONSENSUS-PUBKEY" \
  --gas-fee 1000000ugnot \
  --gas-wanted 50000000 \
  --chainid onyx-1 \
  --remote https://rpc.onyx.testnets.gno.land \
  --broadcast \
  YOUR-KEY-NAME
```

Server type must be one of `cloud`, `on-prem`, `data-center`.

> **Note:** Registration only makes you a **candidate**. A GovDAO member must
> create and pass a proposal to add you to the active validator set
> (via `r/sys/validators/v0`). Once that proposal executes, your node joins the valset.

---

### 11. Update description (optional)

Description limit is **2048 characters**. To update after registration:

```bash
gnokey maketx call \
  --pkgpath "gno.land/r/gnops/valopers" \
  --func "UpdateDescription" \
  --args "YOUR-G1-OPERATOR-ADDRESS" \
  --args "YOUR-NEW-DESCRIPTION" \
  --gas-fee 1000000ugnot \
  --gas-wanted 50000000 \
  --chainid onyx-1 \
  --remote https://rpc.onyx.testnets.gno.land \
  --broadcast \
  YOUR-KEY-NAME
```

---

## Useful commands

```bash
# Service management
sudo systemctl start gnoland
sudo systemctl stop gnoland
sudo systemctl restart gnoland
sudo systemctl status gnoland

# Logs
sudo journalctl -u gnoland -f
sudo journalctl -u gnoland --since "1 hour ago"

# Sync status
curl -s http://127.0.0.1:17657/status | python3 -c \
  "import sys,json; d=json.load(sys.stdin)['result']['sync_info']; print('Height:', d['latest_block_height'], '| Catching up:', d['catching_up'])"

# Installed version (must match the last row of UPGRADES.md)
gnoland version

# Validator key info
gnoland secrets get validator_key

# Wallet list
gnokey list
```

---

## Explorer & resources

| Resource | URL |
|----------|-----|
| Explorer | https://onyx.testnets.gno.land |
| Faucet | https://onyx.testnets.gno.land/faucet |
| Valopers | https://onyx.testnets.gno.land/r/gnops/valopers |
| Active validators | https://onyx.testnets.gno.land/r/sys/validators/v0 |
| RPC | https://rpc.onyx.testnets.gno.land |
| Genesis release | https://github.com/gnolang/gno/releases/tag/chain%2Fonyx |
| Upgrade ledger | https://github.com/gnolang/gno/blob/chain/mainnet/misc/deployments/onyx.gno.land/UPGRADES.md |

---

## Firewall

```bash
# P2P — must be open to public
sudo ufw allow 17656/tcp comment "gnoland P2P"

# RPC — open if you serve public endpoints
sudo ufw allow 17657/tcp comment "gnoland RPC"

# Prometheus — restrict to monitoring server only
sudo ufw allow from YOUR-MONITORING-IP to any port 17660
```

(Use 26656 / 26657 if you kept the default ports.)

---

