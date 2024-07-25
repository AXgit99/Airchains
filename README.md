**install go**
```
cd $HOME
VER="1.21.6"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

**set vars**
```
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export AIRCHAIN_CHAIN_ID="junction"" >> $HOME/.bash_profile
echo "export AIRCHAIN_PORT="19"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binary**
```
cd $HOME
wget -O junctiond https://github.com/airchains-network/junction/releases/download/v0.1.0/junctiond
chmod +x junctiond
mv junctiond $HOME/go/bin/
```

**config and init app**
```
junctiond init $MONIKER --chain-id $AIRCHAIN_CHAIN_ID 
sed -i -e "s|^node *=.*|node = \"tcp://localhost:${AIRCHAIN_PORT}657\"|" $HOME/.junction/config/client.toml
```

**download genesis and addrbook**
```
wget -O $HOME/.junction/config/genesis.json https://testnet-files.itrocket.net/airchains/genesis.json
wget -O $HOME/.junction/config/addrbook.json https://testnet-files.itrocket.net/airchains/addrbook.json
```

**set seeds and peers**
```
SEEDS="04e2fdd6ec8f23729f24245171eaceae5219aa91@airchains-testnet-seed.itrocket.net:19656"
PEERS="47f61921b54a652ca5241e2a7fc4ed8663091e89@airchains-testnet-peer.itrocket.net:19656,38ffaf594a80b88ffaa0ecb3847bf0f77e5c52fe@5.9.87.231:36656,de2e7251667dee5de5eed98e54a58749fadd23d8@34.77.82.65:26656,8d7fceb13fe45bace6c50bb1e14527aa2f1dbdb6@136.243.88.210:19656,999a234d17ef264cdccad3dbbe489404270452c1@65.108.126.173:26656,1db4bf40135b6991b1594d44054aea5f74bb4d47@65.109.113.242:10656,335e9ac0dc613a7b496aff83858174bb4ef16374@37.27.110.190:19656,f2c2b07d9b0afccc8a237b34753f6a24349cfccd@65.108.10.239:60856,5880ddf4518b061c111ae6bf07b1ef76ef2a42af@158.220.100.154:26656,872982e8a521f0e56c237045bbd69dbe97b772b3@65.21.214.117:63656,9ba635344d9c64a4b1d82d7e1138d0216afc27c4@167.235.14.83:34656"
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.junction/config/config.toml
```

**set custom ports in app.toml**
```
sed -i.bak -e "s%:1317%:${AIRCHAIN_PORT}317%g;
s%:8080%:${AIRCHAIN_PORT}080%g;
s%:9090%:${AIRCHAIN_PORT}090%g;
s%:9091%:${AIRCHAIN_PORT}091%g;
s%:8545%:${AIRCHAIN_PORT}545%g;
s%:8546%:${AIRCHAIN_PORT}546%g;
s%:6065%:${AIRCHAIN_PORT}065%g" $HOME/.junction/config/app.toml
```

**set custom ports in config.toml file**
```
sed -i.bak -e "s%:26658%:${AIRCHAIN_PORT}658%g;
s%:26657%:${AIRCHAIN_PORT}657%g;
s%:6060%:${AIRCHAIN_PORT}060%g;
s%:26656%:${AIRCHAIN_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${AIRCHAIN_PORT}656\"%;
s%:26660%:${AIRCHAIN_PORT}660%g" $HOME/.junction/config/config.toml
```

**config pruning**
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.junction/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.junction/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.junction/config/app.toml
```

**set minimum gas price, enable prometheus and disable indexing**
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.001amf"|g' $HOME/.junction/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.junction/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.junction/config/config.toml
```

# create service file
sudo tee /etc/systemd/system/junctiond.service > /dev/null <<EOF
[Unit]
Description=Airchains node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.junction
ExecStart=$(which junctiond) start --home $HOME/.junction
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

# reset and download snapshot
junctiond tendermint unsafe-reset-all --home $HOME/.junction
if curl -s --head curl https://testnet-files.itrocket.net/airchains/snap_airchains.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://testnet-files.itrocket.net/airchains/snap_airchains.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.junction
    else
  echo no have snap
fi

# enable and start service
sudo systemctl daemon-reload
sudo systemctl enable junctiond
sudo systemctl restart junctiond && sudo journalctl -u junctiond -f
