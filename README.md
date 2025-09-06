# RPC-Node-Guide-Zero-Gravity-0G
## Hardware requirements

Operating System:  Ubuntu 18.04 or later LTS  
Number of CPUs:    8  
RAM:	             64GB  
Storage:           1TB NVMe SSD  
Bandwidth:         100mps for Download / Upload

#
1. Install prerequisites and updates
```bash
sudo apt update
sudo apt install curl git make jq build-essential gcc unzip wget lz4 aria2 -y
```

2. Download galileo repository
```bash
wget https://github.com/0glabs/0gchain-NG/releases/download/v2.0.2/galileo-v2.0.2.tar.gz
```

3. Extract Galileo.tar.gz
```bash
tar -xzf galileo-v2.0.2.tar.gz
rm galileo-v2.0.2.tar.gz
mv galileo-v2.0.2 galileo
cd galileo
```
4. Copy Files and Set Permissions
```bash
sudo mv ./bin/geth /usr/local/bin/geth
sudo mv ./bin/0gchaind /usr/local/bin/0gchaind
sudo chmod +x /usr/local/bin/geth /usr/local/bin/0gchaind
```
5. Initialize geth
```bash
/usr/local/bin/geth init --datadir $HOME/galileo/0g-home/geth-home ./genesis.json
```
6. Initialize 0gchaind
Set Moniker
```bash
read -p "Enter your MONIKER value: " MONIKER
SERVER_IP=$(hostname -I | awk '{print $1}')
```
Init
```bash
/usr/local/bin/0gchaind init "$MONIKER" --home $HOME/galileo/tmp
```
7. Copy files to 0gchaind directory
```bash
cp $HOME/galileo/tmp/data/priv_validator_state.json $HOME/galileo/0g-home/0gchaind-home/data/
cp $HOME/galileo/tmp/config/node_key.json $HOME/galileo/0g-home/0gchaind-home/config/
cp $HOME/galileo/tmp/config/priv_validator_key.json $HOME/galileo/0g-home/0gchaind-home/config/
```
```bash
mkdir -p $HOME/.0gchaind
mv $HOME/galileo/0g-home $HOME/.0gchaind/
```
8. Create 0gchaind service file
```bash
sudo tee /etc/systemd/system/0gchaind.service > /dev/null <<EOF
[Unit]
Description=0GChainD Service
After=network.target

[Service]
User=$USER
WorkingDirectory=$HOME/galileo
ExecStart=/usr/local/bin/0gchaind start     --rpc.laddr tcp://0.0.0.0:26657     --chaincfg.chain-spec devnet     --chaincfg.kzg.trusted-setup-path=$HOME/galileo/kzg-trusted-setup.json     --chaincfg.engine.jwt-secret-path=$HOME/galileo/jwt-secret.hex     --chaincfg.kzg.implementation=crate-crypto/go-kzg-4844     --chaincfg.block-store-service.enabled     --chaincfg.node-api.enabled     --chaincfg.node-api.logging     --chaincfg.node-api.address 0.0.0.0:3500     --pruning=nothing     --home=$HOME/.0gchaind/0g-home/0gchaind-home     --p2p.seeds=85a9b9a1b7fa0969704db2bc37f7c100855a75d9@8.218.88.60:26656     --p2p.external_address=54.38.177.118:26656
Restart=always
RestartSec=5
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
EOF
```
9. Create Geth service file
```bash
sudo tee /etc/systemd/system/geth.service > /dev/null <<EOF
[Unit]
Description=Geth Service for 0GChainD
After=network.target

[Service]
User=$USER
WorkingDirectory=$HOME/galileo
ExecStart=/usr/local/bin/geth --config $HOME/galileo/geth-config.toml \
    --nat extip:$(curl -4 -s ifconfig.me) \
    --bootnodes enode://de7b86d8ac452b1413983049c20eafa2ea0851a3219c2cc12649b971c1677bd83fe24c5331e078471e52a94d95e8cde84cb9d866574fec957124e57ac6056699@8.218.88.60:30303 \
    --datadir $HOME/.0gchaind/0g-home/geth-home \
    --networkid 16601
Restart=always
RestartSec=5
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
EOF
```
10. Reload daemon
```bash
sudo systemctl daemon-reload
sudo systemctl enable geth
sudo systemctl enable 0gchaind
sudo systemctl start geth
sudo systemctl start 0gchaind
```
### Check logs
```bash
journalctl -u 0gchaind -u geth -f
```
### Check latest block
```bash
0gchaind status | jq '{ latest_block_height: .sync_info.latest_block_height, catching_up: .sync_info.catching_up }'
```
### Check block status
```bash
while true; do
  PORT=$(grep -A 3 '^\[rpc\]' $HOME/.0gchaind/0g-home/0gchaind-home/config/config.toml | grep -oP 'laddr = "tcp://[0-9.:]+:\K\d+')
  local=$(curl -s localhost:$PORT/status | jq -r '.result.sync_info.latest_block_height//0')
  network=$(curl -s -X POST https://evmrpc-testnet.0g.ai \
      -H "Content-Type: application/json" \
      -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
      | jq -r '.result' | xargs printf "%d")

  left=$((network - local))
  echo -e "Local: \033[1;34m$local\033[0m | Network: \033[1;36m$network\033[0m | Left: \033[1;31m$left\033[0m"
  sleep 5
done
```

## Delete RPC Node
```bash
cd $HOME
sudo systemctl stop 0gchaind geth
sudo systemctl disable 0gchaind geth
sudo rm /etc/systemd/system/0gchaind.service
sudo rm /etc/systemd/system/geth.service
sudo systemctl daemon-reload
sudo rm -f $(which 0gchaind)
sudo rm -rf $HOME/.0gchaind
sudo rm -rf $HOME/galileo
```









