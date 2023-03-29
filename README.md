# Namada Blockchain Installation Guide
Instructions for installing and operating Node Namada Ubuntu 22.x (Recommended), Ubuntu 20.x
# Install Dependency
Update Ubuntu
```
apt update -y
apt upgrade -y
```
Install build Dependency
```
sudo apt install -y make curl git-core libssl-dev pkg-config clang libclang-12-dev build-essential bsdmainutils jq chrony liblz4-tool ncdu uidmap dbus-user-session
```

install Rustc (Via Rustup for lastest update)
```
curl https://sh.rustup.rs -sSf | sh
```
Press 1 to install
```
1) Proceed with installation (default)
```
Add Rust to System PATH
```
source "$HOME/.cargo/env"
```
Verify Rustc the Installation
```
rustup show
```
Ko biet can doan nay ko
```
if ! [ -x "$(command -v go)" ]; then
  ver="1.19.4"
  cd $HOME
  wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
  sudo rm -rf /usr/local/go
  sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
  rm "go$ver.linux-amd64.tar.gz"
  echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile
  source ~/.bash_profile
fi
```
```
echo "export NAMADA_TAG=v0.14.3" >> ~/.bash_profile
echo "export TM_HASH=v0.1.4-abciplus" >> ~/.bash_profile
echo "export CHAIN_ID=public-testnet-6.0.a0266444b06" >> ~/.bash_profile
echo "export WALLET=wallet" >> ~/.bash_profile

#***CHANGE parameters !!!!!!!!!!!!!!!!!!!!!!!!!!!!***
echo "export VALIDATOR_ALIAS=YOUR_MONIKER" >> ~/.bash_profile

source ~/.bash_profile
```

```
cd $HOME && git clone https://github.com/anoma/namada && cd namada && git checkout $NAMADA_TAG
make build-release
```

# Step by step to install Namada Testnet V 0.14.2

https://github.com/systemd-run/manuals/tree/main/namada#readme
https://docs.namada.net/user-guide/install/from-source.html

#install update and libs

cd $HOME

sudo apt update && sudo apt upgrade -y

sudo apt install curl tar wget clang pkg-config libssl-dev libclang-dev -y

sudo apt install jq build-essential bsdmainutils git make ncdu gcc git jq chrony liblz4-tool -y

sudo apt install -y uidmap dbus-user-session


cd $HOME
  sudo apt update
  sudo curl https://sh.rustup.rs -sSf | sh -s -- -y
  . $HOME/.cargo/env
  curl https://deb.nodesource.com/setup_16.x | sudo bash
  sudo apt install cargo nodejs -y < "/dev/null"
  cargo --version
  
if ! [ -x "$(command -v go)" ]; then
  ver="1.19.4"
  cd $HOME
  wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
  sudo rm -rf /usr/local/go
  sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
  rm "go$ver.linux-amd64.tar.gz"
  echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile
  source ~/.bash_profile
fi

#Setting up vars

echo "export NAMADA_TAG=v0.14.2" >> ~/.bash_profile
echo "export TM_HASH=v0.1.4-abciplus" >> ~/.bash_profile
echo "export CHAIN_ID=public-testnet-5.0.d25aa64ace6" >> ~/.bash_profile
echo "export WALLET=wallet" >> ~/.bash_profile

#***CHANGE parameters !!!!!!!!!!!!!!!!!!!!!!!!!!!!***
echo "export VALIDATOR_ALIAS=YOUR_MONIKER" >> ~/.bash_profile

source ~/.bash_profile

cd $HOME && git clone https://github.com/anoma/namada && cd namada && git checkout $NAMADA_TAG
make build-release


cd $HOME && git clone https://github.com/heliaxdev/tendermint && cd tendermint && git checkout $TM_HASH
make build

cd $HOME && cp $HOME/tendermint/build/tendermint  /usr/local/bin/tendermint && cp "$HOME/namada/target/release/namada" /usr/local/bin/namada && cp "$HOME/namada/target/release/namadac" /usr/local/bin/namadac && cp "$HOME/namada/target/release/namadan" /usr/local/bin/namadan && cp "$HOME/namada/target/release/namadaw" /usr/local/bin/namadaw

tendermint version
namada --version

#run fullnode
cd $HOME && namada client utils join-network --chain-id $CHAIN_ID

cd $HOME && wget "https://github.com/heliaxdev/anoma-network-config/releases/download/public-testnet-5.0.d25aa64ace6/public-testnet-5.0.d25aa64ace6.tar.gz"
tar xvzf "$HOME/public-testnet-5.0.d25aa64ace6.tar.gz"

```
#Make service
sudo tee /etc/systemd/system/namadad.service > /dev/null <<EOF
[Unit]
Description=namada
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.namada
Environment=NAMADA_LOG=debug
Environment=NAMADA_TM_STDOUT=true
ExecStart=/usr/local/bin/namada --base-dir=$HOME/.namada node ledger run 
StandardOutput=syslog
StandardError=syslog
Restart=always
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

sudo systemctl daemon-reload
sudo systemctl enable namadad
sudo systemctl start namadad

#add  peers
#cd $HOME
#sudo systemctl stop namadad
#rm "$HOME/.namada/public-testnet-5.0.d25aa64ace6/tendermint/config/addrbook.json"
#curl -s https://raw.githubusercontent.com/systemd-run/manuals/main/namada/addrbook.json > $HOME/.namada/public-testnet-5.0.d25aa64ace6/tendermint/config/addrbook.json
#sudo systemctl restart namadad && sudo journalctl -u namadad -f -o cat

#waiting full synchronization

# check "catching_up": false  --- is OK
curl -s localhost:26657/status

#Make wallet and run validator

cd $HOME
namada wallet address gen --alias $WALLET

namada client transfer \
  --source faucet \
  --target $WALLET \
  --token NAM \
  --amount 1000 \
  --signer $WALLET

cd $HOME
namada client init-validator --alias $VALIDATOR_ALIAS --source $WALLET --commission-rate 0.05 --max-commission-rate-change 0.01 --gas-limit 10000000

#enter pass

cd $HOME
namada client transfer \
    --token NAM \
    --amount 1000 \
    --source faucet \
    --target $VALIDATOR_ALIAS \
    --signer $VALIDATOR_ALIAS
	
#use faucet again because min stake 1000 and you need some more NAM
namada client transfer \
    --token NAM \
    --amount 1000 \
    --source faucet \
    --target $VALIDATOR_ALIAS \
    --signer $VALIDATOR_ALIAS
	
#check balance
namada client balance --owner $VALIDATOR_ALIAS --token NAM

#stake your funds
namada client bond \
  --validator $VALIDATOR_ALIAS \
  --amount 1500 \
  --gas-limit 10000000
  
#print your validator address

RAW_ADDRESS=`cat "$HOME/.namada/$CHAIN_ID/wallet.toml" | grep address`
WALLET_ADDRESS=$(echo -e $RAW_ADDRESS | sed 's|.*=||' | sed -e 's/^ "//' | sed -e 's/"$//')
echo $WALLET_ADDRESS
echo "export WALLET_ADDRESS=$WALLET_ADDRESS" >> ~/.bash_profile
source ~/.bash_profile

#waiting more than 2 epoch and check your status
namada client bonded-stake | grep $WALLET_ADDRESS
namada client bonds | grep $WALLET_ADDRESS

#check only height logs
sudo journalctl -u namadad -n 10000 -f -o cat | grep height

#DELETE NODE!!!
systemctl stop namadad && systemctl disable namadad
rm /etc/systemd/system/namadad* -rf
rm $(which namadad) -rf
rm $HOME/.namada* -rf
rm $HOME/namada -rf
rm $HOME/tendermint -rf
