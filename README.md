#  ✅ Cosmos SDK Auto‑Unjail Script created by https://x.com/Coinstampp

- Coinstamp Validator:
  
https://republic-testnet-explorer.vercel.app/validators/raivaloper13swaxn5mecp8ltpcqn4925uz67r3vxajr3zfp8


- Web : https://republicai.io
- X : https://x.com/republicfdn


# What This Does
- Checks your validator every 60 seconds
- Automatically broadcasts tx slashing unjail if the validator is jailed
- Keeps running even if RPC temporarily fails
- Uses --node only if explicitly set
- Rate-limited to prevent tx spam
- Suitable for production validators


## 1. Install Dependencies : 

```bash
sudo apt update
sudo apt install -y jq
```

2. Create the Script
```bash
sudo nano /root/cosmos_auto_unjail.sh
```

Paste the full script below:

- Don't FORGET TO CHANGE THESE 2 PARAMETERS.

- REPUBLIC_VALIDATOR_ADDRESS, REPUBLIC_KEYRING_PASSWORD

- Leave empty to use default RPC
REPUBLIC_NODE=""

- Example (custom local RPC port):
  If your RPC is running on a different port (e.g. 43657), set:
  REPUBLIC_NODE="tcp://localhost:43657"

```bash
#!/usr/bin/env bash

#############################################
# Cosmos SDK Auto-Unjail Monitoring Script
#############################################

# ========== CONFIGURATION ==========
REPUBLIC_BINARY="republicd"
REPUBLIC_WALLET_NAME="wallet"
REPUBLIC_CHAIN_ID="raitestnet_77701-1"
REPUBLIC_GAS_PRICES="1000000000arai"
REPUBLIC_GAS_ADJUSTMENT="1.5"

# Leave empty to use default RPC
REPUBLIC_NODE=""

# REQUIRED: validator operator address
REPUBLIC_VALIDATOR_ADDRESS="PUT_YOUR_VALOPER_ADDRESS_HERE"

# Keyring password (file backend)
REPUBLIC_KEYRING_PASSWORD="PUT_YOUR_KEY_PASSWORD_HERE"

REPUBLIC_CHECK_INTERVAL=60
REPUBLIC_LOG_FILE="/var/log/cosmos_auto_unjail.log"

# Prevent unjail spam (seconds)
REPUBLIC_MIN_SECONDS_BETWEEN_UNJAILS=600
# ===================================

log() {
  echo "[$(date -u '+%Y-%m-%dT%H:%M:%SZ')] $1" | tee -a "$REPUBLIC_LOG_FILE"
}

# Optional --node flag
NODE_FLAG=""
if [ -n "$REPUBLIC_NODE" ]; then
  NODE_FLAG="--node $REPUBLIC_NODE"
fi

# Dependency checks
command -v "$REPUBLIC_BINARY" >/dev/null 2>&1 || { log "ERROR: $REPUBLIC_BINARY not found"; exit 1; }
command -v jq >/dev/null 2>&1 || { log "ERROR: jq not installed"; exit 1; }

log "=========================================="
log "Cosmos Auto-Unjail Monitor STARTED"
log "Validator: $REPUBLIC_VALIDATOR_ADDRESS"
log "Check interval: ${REPUBLIC_CHECK_INTERVAL}s"
log "=========================================="

last_unjail_ts=0

while true; do
  log "Checking validator status..."

  VALIDATOR_INFO=$($REPUBLIC_BINARY q staking validator \
    $REPUBLIC_VALIDATOR_ADDRESS $NODE_FLAG -o json 2>&1)

  if ! echo "$VALIDATOR_INFO" | jq . >/dev/null 2>&1; then
    log "WARN: Failed to fetch validator info"
    sleep "$REPUBLIC_CHECK_INTERVAL"
    continue
  fi

  IS_JAILED=$(echo "$VALIDATOR_INFO" | jq -r '.validator.jailed // false')
  STATUS=$(echo "$VALIDATOR_INFO" | jq -r '.validator.status // "unknown"')

  log "Status: jailed=$IS_JAILED | status=$STATUS"

  now=$(date +%s)

  if [ "$IS_JAILED" = "true" ]; then
    if (( now - last_unjail_ts < REPUBLIC_MIN_SECONDS_BETWEEN_UNJAILS )); then
      log "Jailed but rate-limited, waiting..."
      sleep "$REPUBLIC_CHECK_INTERVAL"
      continue
    fi

    log "🚨 VALIDATOR JAILED! Sending unjail transaction..."

    UNJAIL_OUTPUT=$(echo "$REPUBLIC_KEYRING_PASSWORD" | \
      $REPUBLIC_BINARY tx slashing unjail \
      --from $REPUBLIC_WALLET_NAME \
      --chain-id $REPUBLIC_CHAIN_ID \
      --gas auto \
      --gas-adjustment $REPUBLIC_GAS_ADJUSTMENT \
      --gas-prices $REPUBLIC_GAS_PRICES \
      $NODE_FLAG -y 2>&1)

    if [ $? -eq 0 ]; then
      TX_HASH=$(echo "$UNJAIL_OUTPUT" | grep -oP 'txhash:\s*\K[A-Fa-f0-9]+' || echo "unknown")
      log "✅ Unjail transaction sent. TX hash: $TX_HASH"
      last_unjail_ts=$now
    else
      log "❌ Unjail FAILED: $UNJAIL_OUTPUT"
      last_unjail_ts=$now
    fi
  else
    log "✅ Validator is healthy"
  fi

  sleep "$REPUBLIC_CHECK_INTERVAL"
done
```

Save and exit:

CTRL + X → Y → ENTER

3. Set Permissions
```
sudo chmod 700 /root/cosmos_auto_unjail.sh
```

4. Create systemd Service
```
sudo nano /etc/systemd/system/cosmos-unjail.service
```

Paste this:

```
[Unit]
Description=Cosmos SDK Auto-Unjail Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
ExecStart=/usr/bin/env bash /root/cosmos_auto_unjail.sh
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

5. Enable and Start the Service

```
sudo systemctl daemon-reload
sudo systemctl enable cosmos-unjail.service
sudo systemctl start cosmos-unjail.service
```


6. Monitor Logs

```
sudo journalctl -u cosmos-unjail.service -f
```


Configuration Variables
Edit these in /root/cosmos_auto_unjail.sh:

Variable	Description	Example
- REPUBLIC_BINARY	Your chain binary	republicd, gaiad, osmosisd
- REPUBLIC_WALLET_NAME	Local key name	wallet
- REPUBLIC_CHAIN_ID	Chain ID	raitestnet_77701-1
- REPUBLIC_GAS_PRICES	Gas prices with denom	1000000000arai
- REPUBLIC_VALIDATOR_ADDRESS	Validator operator address	raivaloper1...
- REPUBLIC_KEYRING_PASSWORD	Keyring passphrase	Your password
- REPUBLIC_NODE	Custom RPC (optional)	tcp://localhost:43657

Troubleshooting
Check service status:
```
sudo systemctl status cosmos-unjail.service
```
View full logs:
```
sudo journalctl -u cosmos-unjail.service --no-pager
```
Restart service:
```
sudo systemctl restart cosmos-unjail.service
```
Stop service:
```
sudo systemctl stop cosmos-unjail.service
```


Made with ❤️ for Cosmos validators
