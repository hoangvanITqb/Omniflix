Dependencies Installation

**Install dependencies for building from source**
```
sudo apt update
sudo apt install -y curl git jq lz4 build-essential
```

# Install Go
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.22.7.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.profile
source .profile
Node Installation

Node Name

Your Node Name
Port prefix

169
# Clone project repository
cd && rm -rf omniflixhub
git clone https://github.com/OmniFlix/omniflixhub
cd omniflixhub
git checkout v5.2.0

# Build binary
make install

# Prepare cosmovisor directories
mkdir -p $HOME/.omniflixhub/cosmovisor/genesis/bin
ln -s $HOME/.omniflixhub/cosmovisor/genesis $HOME/.omniflixhub/cosmovisor/current -f

# Copy binary to cosmovisor directory
cp $(which omniflixhubd) $HOME/.omniflixhub/cosmovisor/genesis/bin

# Set node CLI configuration
omniflixhubd config chain-id omniflixhub-1
omniflixhubd config keyring-backend file
omniflixhubd config node tcp://localhost:16957

# Initialize the node
omniflixhubd init "Your Node Name" --chain-id omniflixhub-1

# Download genesis and addrbook files
curl -L https://snapshots.nodejumper.io/omniflixhub/genesis.json > $HOME/.omniflixhub/config/genesis.json
curl -L https://snapshots.nodejumper.io/omniflixhub/addrbook.json > $HOME/.omniflixhub/config/addrbook.json

# Set seeds
sed -i -e 's|^seeds *=.*|seeds = "babc3f3f7804933265ec9c40ad94f4da8e9e0017@seed.rhinostake.com:16956,ebc272824924ea1a27ea3183dd0b9ba713494f83@omniflixhub-mainnet-seed.autostake.com:27306,20e1000e88125698264454a884812746c2eb4807@seeds.lavenderfive.com:16956,9aa8a73ea9364aa3cf7806d4dd25b6aed88d8152@omniflix.seed.mzonder.com:10656,6b0ffcce9b59b91ceb8eea5d4599e27707e3604a@seeds.stakeup.tech:10215,8542cd7e6bf9d260fef543bc49e59be5a3fa9074@seed.publicnode.com:26656,574b37cc6e80663e70673cbe848147c2643ca48e@35.240.187.174:26656,ebc272824924ea1a27ea3183dd0b9ba713494f83@omniflixhub-mainnet-peer.autostake.com:27306,d8e371758cdb310906bc32ba0bb922642bb33536@65.21.91.99:26756,82feb443470ff81afa830e15fea387cac4849aac@mainnet.omniflix.peers.stakr.space:36656"|' $HOME/.omniflixhub/config/config.toml

# Set minimum gas price
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.001uflix"|' $HOME/.omniflixhub/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.omniflixhub/config/app.toml

# Enable prometheus
sed -i -e 's|^prometheus *=.*|prometheus = true|' $HOME/.omniflixhub/config/config.toml

# Change ports
sed -i -e "s%:1317%:16917%; s%:8080%:16980%; s%:9090%:16990%; s%:9091%:16991%; s%:8545%:16945%; s%:8546%:16946%; s%:6065%:16965%" $HOME/.omniflixhub/config/app.toml
sed -i -e "s%:26658%:16958%; s%:26657%:16957%; s%:6060%:16960%; s%:26656%:16956%; s%:26660%:16961%" $HOME/.omniflixhub/config/config.toml

# Download latest chain data snapshot
curl "https://snapshots.nodejumper.io/omniflixhub/omniflixhub_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.omniflixhub"

# Install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.7.0

# Create a service
sudo tee /etc/systemd/system/omniflixhub.service > /dev/null << EOF
[Unit]
Description=OmniFlix node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.omniflixhub
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.omniflixhub"
Environment="DAEMON_NAME=omniflixhubd"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable omniflixhub.service

# Start the service and check the logs
sudo systemctl start omniflixhub.service
sudo journalctl -u omniflixhub.service -f --no-hostname -o cat
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
