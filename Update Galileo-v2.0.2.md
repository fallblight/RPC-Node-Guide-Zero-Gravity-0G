#
1. Stop the service
```bash
systemctl stop 0gchaind geth
```

2. Download and extract galileo v2.0.2
```bash
wget https://github.com/0glabs/0gchain-NG/releases/download/v2.0.2/galileo-v2.0.2.tar.gz
tar -xvzf galileo-v2.0.2.tar.gz
cd $HOME/galileo-v2.0.2/
```

3. Give permission
```bash
chmod +x ./bin/0gchaind
chmod +x ./bin/geth
```
4. Copy geth and 0gchaind to your correct path/directory
```bash
cp ./bin/0gchaind /usr/local/bin/0gchaind
cp ./bin/geth /usr/local/bin/geth
```
5. Edit 0gchaind.service
Validator Nodes:

After upgrading, when running your consensus client, you must add the following CLI options for the restaking RPC configuration:
```bash
--chaincfg.restaking.enabled \
--chaincfg.restaking.symbiotic-rpc-dial-url ${ETH_RPC_URL} \
--chaincfg.restaking.symbiotic-get-logs-block-range ${BLOCK_NUM}
```
Note:

Adjust --chaincfg.restaking.symbiotic-rpc-dial-url to your preferred Ethereum RPC provider.
Set --chaincfg.restaking.symbiotic-get-logs-block-range based on your RPC’s performance and limitations (default: 1, meaning 1 block per request). Higher values may speed up syncing but can cause failures if your RPC cannot handle larger ranges.
6. Reload and restart services
```bash
sudo systemctl daemon-reload
sudo systemctl restart 0gchaind
sudo systemctl restart geth
```
