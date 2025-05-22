**Hardware Requirements**
Minimum
```
3CPU 4RAM 80GB
```

**Recommended**
```
4CPU 8RAM 160GB
Rent On Hetzner | Rent On OVH
Dependencies Installation
```

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
```

**Node Installation**
**Clone project repository**
```
cd && rm -rf paloma
git clone https://github.com/palomachain/paloma
cd paloma
git checkout v2.4.2
```
**Install libwasmvm**
```
curl -L https://github.com/CosmWasm/wasmvm/releases/download/v1.5.0/libwasmvm.x86_64.so > libwasmvm.x86_64.so
sudo mv -f libwasmvm.x86_64.so /usr/lib/libwasmvm.x86_64.so
```

**Build binary**
```
make install
```

**Prepare cosmovisor directories**
```
mkdir -p $HOME/.paloma/cosmovisor/genesis/bin
ln -s $HOME/.paloma/cosmovisor/genesis $HOME/.paloma/cosmovisor/current -f
```

**Copy binary to cosmovisor directory**
```
cp $(which palomad) $HOME/.paloma/cosmovisor/genesis/bin
```

**Set node CLI configuration**
```
palomad config chain-id tumbler
palomad config keyring-backend file
palomad config node tcp://localhost:12157
```

**Initialize the node**
```
palomad init "Your Node Name" --chain-id tumbler
```

**Download genesis and addrbook files**
```
curl -L https://snapshots.nodejumper.io/paloma/genesis.json > $HOME/.paloma/config/genesis.json
curl -L https://snapshots.nodejumper.io/paloma/addrbook.json > $HOME/.paloma/config/addrbook.json
```

**Set seeds**
```
sed -i -e 's|^seeds *=.*|seeds = "400f3d9e30b69e78a7fb891f60d76fa3c73f0ecc@paloma.rpc.kjnodes.com:11059,ab6875bd52d6493f39612eb5dff57ced1e3a5ad6@95.217.229.18:10656,9581fadb9a32f2af89d575bb0f2661b9bb216d41@93.190.141.218:26656,874ccf9df2e4c678a18a1fb45a1d3bb703f87fa0@65.109.172.249:26656"|' $HOME/.paloma/config/config.toml
```

**Set minimum gas price**
```
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.01ugrain"|' $HOME/.paloma/config/app.toml
```

**Set pruning**
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.paloma/config/app.toml
```

**Enable prometheus**
```
sed -i -e 's|^prometheus *=.*|prometheus = true|' $HOME/.paloma/config/config.toml
```

**Change ports**
```
sed -i -e "s%:1317%:12117%; s%:8080%:12180%; s%:9090%:12190%; s%:9091%:12191%; s%:8545%:12145%; s%:8546%:12146%; s%:6065%:12165%" $HOME/.paloma/config/app.toml
sed -i -e "s%:26658%:12158%; s%:26657%:12157%; s%:6060%:12160%; s%:26656%:12156%; s%:26660%:12161%" $HOME/.paloma/config/config.toml
```

**Download latest chain data snapshot**
```
curl "https://snapshots.nodejumper.io/paloma/paloma_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.paloma"
```

**Install Cosmovisor**
```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.7.0
```

**Create a service**
```
sudo tee /etc/systemd/system/paloma.service > /dev/null << EOF
[Unit]
Description=Paloma node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.paloma
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.paloma"
Environment="DAEMON_NAME=palomad"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable paloma.service
```

**Start the service and check the logs**
```
sudo systemctl start paloma.service
sudo journalctl -u paloma.service -f --no-hostname -o cat
Secure Server Setup (Optional)
```

**generate ssh keys, if you don't have them already, DO IT ON YOUR LOCAL MACHINE**
```
ssh-keygen -t rsa
```

**save the output, we'll use it later on instead of YOUR_PUBLIC_SSH_KEY**
```
cat ~/.ssh/id_rsa.pub
```

**upgrade system packages**
```
sudo apt update
sudo apt upgrade -y
```

**add new admin user**
```
sudo adduser admin --disabled-password -q
```

**upload public ssh key, replace YOUR_PUBLIC_SSH_KEY with the key above**
```
mkdir /home/admin/.ssh
echo "YOUR_PUBLIC_SSH_KEY" >> /home/admin/.ssh/authorized_keys
sudo chown admin: /home/admin/.ssh
sudo chown admin: /home/admin/.ssh/authorized_keys

echo "admin ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
```

**disable root login, disable password authentication, use ssh keys only**
```
sudo sed -i 's|^PermitRootLogin .*|PermitRootLogin no|' /etc/ssh/sshd_config
sudo sed -i 's|^ChallengeResponseAuthentication .*|ChallengeResponseAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PasswordAuthentication .*|PasswordAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PermitEmptyPasswords .*|PermitEmptyPasswords no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PubkeyAuthentication .*|PubkeyAuthentication yes|' /etc/ssh/sshd_config

sudo systemctl restart sshd
```

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
