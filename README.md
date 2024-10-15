Hardware Requirements

Minimum

4CPU 8RAM 100GB
Recommended

4CPU 16RAM 200GB
Rent On Hetzner | Rent On OVH
Dependencies Installation

**Install dependencies for building from source**
```
sudo apt update
sudo apt install -y curl git jq lz4 build-essential
```

**Install Go**
```
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.22.7.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.profile
source .profile
Node Installation
```

**Clone project repository**
```
cd && rm -rf ojo
git clone https://github.com/ojo-network/ojo
cd ojo
git checkout v0.1.2
```

**Build binary**
``
make install
```

**Prepare cosmovisor directories**
```
mkdir -p $HOME/.ojo/cosmovisor/genesis/bin
sudo ln -s $HOME/.ojo/cosmovisor/genesis $HOME/.ojo/cosmovisor/current -f
sudo ln -s $HOME/.ojo/cosmovisor/current/bin/ojod /usr/local/bin/ojod -f
```

**Move binary to cosmovisor directory**
```
mv $(which ojod) $HOME/.ojo/cosmovisor/genesis/bin
```

**Set node CLI configuration**
```
ojod config chain-id ojo-devnet
ojod config keyring-backend test
ojod config node tcp://localhost:21657
```

**Initialize the node**
```
ojod init "Your Node Name" --chain-id ojo-devnet
```

**Download genesis and addrbook files**
```
curl -L https://snapshots-testnet.nodejumper.io/ojo/genesis.json > $HOME/.ojo/config/genesis.json
curl -L https://snapshots-testnet.nodejumper.io/ojo/addrbook.json > $HOME/.ojo/config/addrbook.json
```

**Set seeds**
```
sed -i -e 's|^seeds *=.*|seeds = "7186f24ace7f4f2606f56f750c2684d387dc39ac@ojo-testnet-seed.itrocket.net:12656"|' $HOME/.ojo/config/config.toml
```

**Set minimum gas price**
```
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.01uojo"|' $HOME/.ojo/config/app.toml
```

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.ojo/config/app.toml

# Change ports
sed -i -e "s%:1317%:21617%; s%:8080%:21680%; s%:9090%:21690%; s%:9091%:21691%; s%:8545%:21645%; s%:8546%:21646%; s%:6065%:21665%" $HOME/.ojo/config/app.toml
sed -i -e "s%:26658%:21658%; s%:26657%:21657%; s%:6060%:21660%; s%:26656%:21656%; s%:26660%:21661%" $HOME/.ojo/config/config.toml

# Download latest chain data snapshot
curl "https://snapshots-testnet.nodejumper.io/ojo/ojo-testnet_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.ojo"

# Install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.6.0

# Create a service
sudo tee /etc/systemd/system/ojo.service > /dev/null << EOF
[Unit]
Description=Ojo node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.ojo
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.ojo"
Environment="DAEMON_NAME=ojod"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable ojo.service

# Start the service and check the logs
sudo systemctl start ojo.service
sudo journalctl -u ojo.service -f --no-hostname -o cat
Create Validator

# create wallet
ojod keys add wallet

## console output:
#- name: wallet
#  type: local
#  address: ojo1ela8c0jhqgjsj2cq7twu9uhda2n8e6cs8ztxs3
#  pubkey: '{"@type":"/cosmos.crypto.secp256k1.PubKey","key":"Auq9WzVEs5pCoZgr2WctjI7fU+lJCH0I3r6GC1oa0tc0"}'
#  mnemonic: ""

#!!! SAVE SEED PHRASE
kite upset hip dirt pet winter thunder slice parent flag sand express suffer chest custom pencil mother bargain remember patient other curve cancel sweet

#!!! SAVE PRIVATE VALIDATOR KEY
cat $HOME/.ojo/config/priv_validator_key.json

# wait util the node is synced, should return FALSE
ojod status 2>&1 | jq .SyncInfo.catching_up

# Request tokens in discord

# verify the balance
ojod q bank balances $(ojod keys show wallet -a)

## console output:
#  balances:
#  - amount: "10000000"
#    denom: uojo

# create validator
ojod tx staking create-validator \
--amount=9000000uojo \
--pubkey=$(ojod tendermint show-validator) \
--moniker="$NODE_MONIKER" \
--chain-id=ojo-devnet \
--commission-rate=0.1 \
--commission-max-rate=0.2 \
--commission-max-change-rate=0.05 \
--min-self-delegation=1 \
--fees=2000uojo \
--from=wallet \
-y

# make sure you see the validator details
ojod q staking validator $(ojod keys show wallet --bech val -a)

# install pricefeeder
cd || return
rm -rf price-feeder
git clone https://github.com/ojo-network/price-feeder
cd price-feeder || return
git checkout v0.1.1
make install
price-feeder version # version: HEAD-5d46ed438d33d7904c0d947ebc6a3dd48ce0de59

mkdir -p $HOME/.price-feeder
curl -s https://raw.githubusercontent.com/ojo-network/price-feeder/main/price-feeder.example.toml > $HOME/.price-feeder/price-feeder.toml

ojod keys add feeder-wallet --keyring-backend os
ojod tx bank send wallet YOUR_FEEDER_ADDRESS 10000000uojo --from wallet --chain-id ojo-devnet --fees 2000uojo -y
ojod q bank balances $(ojod keys show feeder-wallet --keyring-backend os -a)

CHAIN_ID=ojo-devnet
KEYRING_PASSWORD=YOUR_KEYRING_PASSWORD
WALLET_ADDRESS=$(ojod keys show wallet -a)
FEEDER_ADDRESS=$(ojod keys show feeder-wallet --keyring-backend os -a)
VALIDATOR_ADDRESS=$(ojod keys show wallet --bech val -a)
GRPC="localhost:9090"
RPC="http://localhost:26657"

ojod tx oracle delegate-feed-consent $WALLET_ADDRESS $FEEDER_ADDRESS --from wallet --fees 2000uojo -y
ojod q oracle feeder-delegation $VALIDATOR_ADDRESS

sed -i '/^dir *=.*/a pass = ""' $HOME/.price-feeder/price-feeder.toml
sed -i 's|^address *=.*|address = "'$FEEDER_ADDRESS'"|g' $HOME/.price-feeder/price-feeder.toml
sed -i 's|^chain_id *=.*|chain_id = "'$CHAIN_ID'"|g' $HOME/.price-feeder/price-feeder.toml
sed -i 's|^validator *=.*|validator = "'$VALIDATOR_ADDRESS'"|g' $HOME/.price-feeder/price-feeder.toml
sed -i 's|^backend *=.*|backend = "os"|g' $HOME/.price-feeder/price-feeder.toml
sed -i 's|^dir *=.*|dir = "'$HOME/.ojo'"|g' $HOME/.price-feeder/price-feeder.toml
sed -i 's|^pass *=.*|pass = "'$KEYRING_PASSWORD'"|g' $HOME/.price-feeder/price-feeder.toml
sed -i 's|^grpc_endpoint *=.*|grpc_endpoint = "'$GRPC'"|g' $HOME/.price-feeder/price-feeder.toml
sed -i 's|^tmrpc_endpoint *=.*|tmrpc_endpoint = "'$RPC'"|g' $HOME/.price-feeder/price-feeder.toml
sed -i 's|^global-labels *=.*|global-labels = [["chain_id", "'$CHAIN_ID'"]]|g' $HOME/.price-feeder/price-feeder.toml

sudo tee /etc/systemd/system/price-feeder.service > /dev/null << EOF
[Unit]
Description=Ojo Price Feeder
After=network-online.target
[Service]
User=$USER
ExecStart=$(which price-feeder) $HOME/.price-feeder/price-feeder.toml --log-level debug
Restart=on-failure
RestartSec=10
LimitNOFILE=10000
Environment="PRICE_FEEDER_PASS=$KEYRING_PASSWORD"
[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable price-feeder
sudo systemctl start price-feeder
sudo journalctl -u price-feeder -f --no-hostname -o cat
Secure Server Setup (Optional)

# generate ssh keys, if you don't have them already, DO IT ON YOUR LOCAL MACHINE
ssh-keygen -t rsa

# save the output, we'll use it later on instead of YOUR_PUBLIC_SSH_KEY
cat ~/.ssh/id_rsa.pub
# upgrade system packages
sudo apt update
sudo apt upgrade -y

# add new admin user
sudo adduser admin --disabled-password -q

# upload public ssh key, replace YOUR_PUBLIC_SSH_KEY with the key above
mkdir /home/admin/.ssh
echo "YOUR_PUBLIC_SSH_KEY" >> /home/admin/.ssh/authorized_keys
sudo chown admin: /home/admin/.ssh
sudo chown admin: /home/admin/.ssh/authorized_keys

echo "admin ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# disable root login, disable password authentication, use ssh keys only
sudo sed -i 's|^PermitRootLogin .*|PermitRootLogin no|' /etc/ssh/sshd_config
sudo sed -i 's|^ChallengeResponseAuthentication .*|ChallengeResponseAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PasswordAuthentication .*|PasswordAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PermitEmptyPasswords .*|PermitEmptyPasswords no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PubkeyAuthentication .*|PubkeyAuthentication yes|' /etc/ssh/sshd_config

sudo systemctl restart sshd

# install fail2ban
sudo apt install -y fail2ban

# install and configure firewall
sudo apt install -y ufw
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow 9100
sudo ufw allow 26656

# make sure you expose ALL necessary ports, only after that enable firewall
sudo ufw enable

# make terminal colorful
sudo su - admin
source <(curl -s https://raw.githubusercontent.com/nodejumper-org/cosmos-scripts/master/utils/enable_colorful_bash.sh)

# update servername, if needed, replace YOUR_SERVERNAME with wanted server name
sudo hostnamectl set-hostname YOUR_SERVERNAME

# now you can logout (exit) and login again using ssh admin@YOUR_SERVER_IP
