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
wget -O galileo.tar.gz https://github.com/0glabs/0gchain-NG/releases/download/v1.1.1/galileo-v1.1.1.tar.gz
```

3. Extract Galileo.tar.gz
```bash
tar -xzvf galileo.tar.gz -C $HOME
```
4. Copy Files and Set Permissions
```bash
sudo chmod 777 $HOME/galileo/bin/geth
sudo chmod 777 $HOME/galileo/bin/0gchaind
cp $HOME/galileo/bin/geth $HOME/go/bin/geth
cp $HOME/galileo/bin/0gchaind $HOME/go/bin/0gchaind
```
```bash
source $HOME/.bash_profile
geth version
0gchaind version
```
### Replace MONIKER and WALLET with your own name of choice
```bash
echo "export WALLET='WALLET'" >> $HOME/.bash_profile
echo "export MONIKER='MONIKER'" >> $HOME/.bash_profile
source $HOME/.bash_profile
```
5. Initialize Geth
```bash
mkdir -p $HOME/.0gchaind
cp -r $HOME/galileo/0g-home $HOME/.0gchaind
```
```bash
geth init --datadir $HOME/.0gchaind/0g-home/geth-home $HOME/galileo/genesis.json
```
6. Initialize 0gchaind
```bash
0gchaind init "MONIKER" --home $HOME/.0gchaind/tmp
```
```bash
#copy node files to 0gchaind-home
cp $HOME/.0gchaind/tmp/data/priv_validator_state.json $HOME/.0gchaind/0g-home/0gchaind-home/data/
cp $HOME/.0gchaind/tmp/config/node_key.json $HOME/.0gchaind/0g-home/0gchaind-home/config/
cp $HOME/.0gchaind/tmp/config/priv_validator_key.json $HOME/.0gchaind/0g-home/0gchaind-home/config/
#remove temporary file
rm -rf $HOME/.0gchaind/tmp
```
7. Create 0gchaind service file
```bash
cat > /etc/systemd/system/0gchaind.service << EOF
[Unit]
Description=0G Chain Daemon
After=network-online.target

[Service]
User=root
ExecStart=$HOME/go/bin/0gchaind start \\
    --rpc.laddr tcp://0.0.0.0:26657 \\
    --chain-spec devnet \\
    --kzg.trusted-setup-path=$HOME/galileo/kzg-trusted-setup.json \\
    --engine.jwt-secret-path=$HOME/galileo/jwt-secret.hex \\
    --kzg.implementation=crate-crypto/go-kzg-4844 \\
    --block-store-service.enabled \\
    --node-api.enabled \\
    --node-api.logging \\
    --node-api.address 0.0.0.0:3500 \\
    --pruning=nothing \\
    --home $HOME/.0gchaind/0g-home/0gchaind-home \\
    --p2p.seeds 85a9b9a1b7fa0969704db2bc37f7c100855a75d9@8.218.88.60:26656 \\
    --p2p.external_address $(curl -s http://ipv4.icanhazip.com):8656
Environment=CHAIN_SPEC=devnet
WorkingDirectory=$HOME/galileo
Restart=always
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```
7. Create Geth service file
```bash
cat > /etc/systemd/system/geth.service << EOF
[Unit]
Description=Go Ethereum Client
After=network-online.target
Wants=network-online.target

[Service]
User=root
ExecStart=$HOME/go/bin/geth \\
    --config $HOME/galileo/geth-config.toml \\
    --datadir $HOME/.0gchaind/0g-home/geth-home \\
    --networkid 16601 \\
    --port 30303 \\
    --http.port 8645 \\
    --ws.port 8646 \\
    --authrpc.port 8551
Restart=always
WorkingDirectory=$HOME/galileo
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```
8. Reload daemon
```bash
sudo systemctl daemon-reload && \
sudo systemctl enable 0gchaind
sudo systemctl enable geth.service
```
9. Start 0gchaind and Geth
```bash
sudo systemctl start 0gchaind
sudo systemctl status 0gchaind
```
```bash
sudo systemctl start geth
sudo systemctl status geth
```
### Check logs
```bash
sudo journalctl -fu 0gchaind -o cat
sudo journalctl -fu geth -o cat
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
  network=$(curl -s http://152.53.102.226:27657/status | jq -r '.result.sync_info.latest_block_height//0')
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









