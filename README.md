# Installation and operating a Namada Full Node (Testnet)
Lastest version: v0.28.0

Testnet chain ID: coming soon

Namada is a layer 1 protocol that uses Proof-of-Stake consensus and aims to provide interchain privacy for various assets. 
Namada is the initial fractal instance of Anoma and is presently being developed by Heliax, a public goods lab.

The guide below helps you install/update and operate a full node to become a validator in the Namada blockchain network. 
Note that Namada has 2 types of validators, Pre-genesis Validator and Post-genesis Validator. The instructions below are for the Post-genesis Validator.

# Quick Jump
[1. Install new Namada Node](#1-installing-new-namada-full-node)

[2. Config Validator Address](#2config-validator-address)

[3. Operation Commands](#3-node-operation)

[4. QA and Error Handle](#4-qa-and-error-handle)

[5. Update Namada Validator Node](#5-update-namada-validator-node-post-genesis-validator-only)

# Prepare the environment
* OS: Ubuntu 22.04 (Tested), Ubuntu 20.04
* Ports required: 26656, 26657
* CPU:	x86_64 or arm64 processor with at least 4 physical cores (must support AVX/SSE instruction set)
* RAM:	16GB DDR4
* Storage:	at least 1000GB SSD (NVMe SSD is recommended. HDD will also work.)

# 1. Installing new Namada Full Node & Validator
### 1.1 ENV Params
```
cd ~ 
NAMADA_TAG="v0.28.0"
NAMADA_CHAIN_ID="public-testnet-7.0.3c5a38dc983"
TM_HASH="v0.1.4-abciplus"
VALIDATOR_ALIAS="BitberiMem"
```
### 1.2 Install and update dependency
```
sudo apt update -y
sudo apt upgrade -y
sudo apt install curl make clang pkg-config libssl-dev build-essential git jq ncdu bsdmainutils  -y
apt-get install protobuf-compiler -y
apt install -y pkg-config libusb-1.0-0-dev libftdi1-dev -y
```
### 1.3 Installing Rustc
```
cd ~ 
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
#Press 1 to install
source "$HOME/.cargo/env"
rustup show
```
### 1.4 Installing Go (required by Tendermint)
```
goVer="1.19.4"
cd ~
wget "https://golang.org/dl/go$goVer.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$goVer.linux-amd64.tar.gz"
rm "go$goVer.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile
source ~/.bash_profile
go version
```

### 1.5 Installing Tendermint
```
cd ~ 
sudo rm -rf tendermint 
git clone https://github.com/heliaxdev/tendermint && cd tendermint && git checkout $TM_HASH
make build
sudo mv build/tendermint /usr/local/bin/
tendermint version
```
### 1.6 Installing Namada
```
cd ~ && sudo rm -rf $HOME/namada 
git clone https://github.com/anoma/namada 
cd namada 
git checkout $NAMADA_TAG
make build-release
sudo mv target/release/namada /usr/local/bin/
sudo mv target/release/namada[c,n,w] /usr/local/bin/
namada -V
```
### 1.7 Config Namada Service
```
echo "[Unit]
Description=Namada Node
After=network.target

[Service]
User=$USER
WorkingDirectory=$HOME/.namada
Type=simple
ExecStart=/usr/local/bin/namada --base-dir=$HOME/.namada node ledger run
Environment=NAMADA_TM_STDOUT=true
RemainAfterExit=no
Restart=always
RestartSec=5s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target" > $HOME/namadad.service
sudo mv $HOME/namadad.service /etc/systemd/system
sudo tee <<EOF >/dev/null /etc/systemd/journald.conf
Storage=persistent
EOF
sudo systemctl restart systemd-journald
sudo systemctl daemon-reload
sudo systemctl enable namadad
```
### 1.8 Join Testnet 15
```
cd ~
namada client utils join-network --chain-id $NAMADA_CHAIN_ID
```
### 1.9 Start Namada Service
```
sudo systemctl start namadad
```
# 2.Config Validator Address
You will have 3 Address: Main Namada Address(private) > consensus-key address > validator-key address (public)
### Create new Main Namada Address
```
cd ~
namada wallet address gen --alias yourMainWalletAliasName
```
### Initialising Validator account (Validator-key and Consesum-key address)
```
cd ~
namada client init-validator \
  --alias $VALIDATOR_ALIAS_NAME \
  --source yourMainWalletAliasName \
  --commission-rate 0.1 \
  --max-commission-rate-change 0.1
```
### Get tokens from the faucet
```
namadac transfer \
  --token NAM \
  --amount 1000 \
  --source faucet \
  --target $VALIDATOR_ALIAS_NAME \
  --signer $VALIDATOR_ALIAS_NAME
```
You can faucet for any Namada address by alias name or specific address of that address. In this example I use validator address.

You can faucet as many times as you want to get as many test tokens as you want

### Bond token to your validator
```
namada client bond \
  --validator $VALIDATOR_ALIAS_NAME \
  --amount 1000
```
By default, if you do not specify the source, it will be taken from the validator address. 
Of course you can also specify from any other address you own eg Your main Nameda address or something else
```
namada client bond \
  --source $WALLET \
  --validator $VALIDATOR_ALIAS \
  --amount 3000
```
* If you get an error, please try again in a few minutes
### Done ! 
If all goes well, you have now become a validator.
# 3. Node Operation
### Node
Check Node Info
```
curl -s localhost:26657/status
```
Check Log
```
journalctl -u namadad -f
```
Check current height
```
sudo journalctl -u namadad -n 10000 -f -o cat | grep height
```
Restart service 
```
Systemctl restart namadad
```
Check free disk space
```
df -hT / --total | awk '/^total/ {print "Total disk size: "$3" | Used:" $4 " | Avaliable:" $5 " | Usage:" $6}'
```
### Account
Check the ballance
```
namada client balance --owner $WALLET_ALIAS_NAME_OR_FULL_ADDRESS --token NAM
```
Check your Votepower
```
namada client bonded-stake | grep $VALIDATOR_ADDRESS
```
Show all validator in Namada Blockchain
```
namada client bonded-stake
```
Check total bonds (Bonds require 2 epoch to be activated)
```
namadac bonds --validator $VALIDATOR_ALIAS
```
Check current Epoch
```
namadac epoch
```
Unbonds
```
namada client unbond \
  --source $WALLET \
  --validator $VALIDATOR_ALIAS \
  --amount 1.2
```
Send token to another address
```
namada client transfer \
  --source $WALLET_ALIAS \
  --target $RECIVE_WALLET_ADDRESS \
  --token NAM \
  --amount 1000
```

# 4. QA and Error Handle
```
The address doesn’t belong to any known validator account
```
* Check your node is synced or not. Double check that the addresses you entered are correct. If everything is correct wait 1 - 2 epoch then try again.
```
INFO namada_apps::cli::context: Chain ID: namada-internal.00000000000000
The application panicked (crashed).
Message:  called `Result::unwrap()` on an `Err` value:
   0: couldn't read genesis config file from .namada/namada-internal.00000000000000.toml
   1: No such file or directory (os error 2)
Location:
   apps/src/lib/config/genesis.rs:687
```
* Let's go back to the home directory "cd ~"

# 5. Update Namada Validator Node (Post-genesis Validator Only)
Currently, Namada Testnet version allows export of private keys but does not support automatic import of private keys. 
There is also no recommendation from Namada Team to request the use of addresses from the previous testnet as proof of contribution. 
So simple to update to the new version, you can delete the old version then reinstall new version.
Of course there is still a manual way to do it if you have a need to do so please leave an issue in this Repo. 
We will add a new article on how to manually export and import Namada accounts (if it's a real need).

To delete and reinstall Namada do the following:

Stop Namadad service
```
systemctl stop namadad
```
Remove Namada Data
```
cd ~
sudo rm -rf .namada
```
Pull new Namada version
```
NAMADA_TAG="v0.15.1"
NAMADA_CHAIN_ID="public-testnet-7.0.3c5a38dc983"
cd ~/namada
git fetch
git checkout $NAMADA_TAG
make build-release
sudo mv target/release/namada /usr/local/bin/
sudo mv target/release/namada[c,n,w] /usr/local/bin/
namada -V
cd ~
```
Then rejoin new Testnet and config your validator + faucet [From Step 1.8 #Join Testnet 7](#18-join-testnet-7)
### Done !

# Reference
https://docs.namada.net/testnets/post-genesis-validator.html

# Note
If you encounter any problems or difficulties, feel free to create an issue in this repo. 

Thanks

