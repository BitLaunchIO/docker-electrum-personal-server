# Docker EPS

## **Table of Contents**
 - [Installation](#installation)
 - [Configuration](#configuration)
 - [Start the containers](#start-the-containers)
 - [Connecting Electrum to EPS](#connecting-electrum-to-eps)
 - [Important Notes](#important-notes)

---
## **Installation**
Clone the dockerised electrum personal server repo.
```sh
git clone https://github.com/BitLaunchIO/docker-electrum-personal-server.git
```

Install docker and docker-compose:
```sh
apt update -y && apt install docker docker-compose -y
```

If you are running this on a different machine to Electrum, it’s recommended to download Tor, otherwise skip to Configuration. Connecting via Tor will ensure your IP address won’t be linked to your BTC addresses.

Install Tor on both machines:
```sh
apt update -y && apt install tor -y
```

Note: If you are not using the root user, you will need to add your user to the `debian-tor` or `tor` group then logout/login.
```sh
sudo usermod -aG debian-tor MyUser
```

**docker-eps host machine**

The `torrc` file set up with a hidden service will allow you to connect to the EPS over Tor. Copy this config file to `/etc/tor/torrc` and restart the Tor service.
```sh
cp docker-eps/torrc /etc/tor/torrc
systemctl restart tor
```

**electrum client machine**

Likewise with the docker-eps host machine, the electrum client machine needs a `torrc` file.
```sh
sudo nano /etc/tor/torrc
```

Add the following lines to the newly created `torrc` file. Or replace any existing lines with the following.
```
User tor
ControlPort 9051
CookieAuthentication 1
CookieAuthFileGroupReadable 1
```

Save the changes then restart the tor service:
```sh
sudo systemctl restart tor
```

---
## **Configuration**
Add your Electrum wallet <u>public</u> key(s) to `config.ini` this is so that the EPS can scan for transactions and find your addresses.
```
mywallet = xpubkey
myotherwallet = zpubkey
```

Bitcoin core will need some nodes to be able to sync to the blockchain. Add some nodes  by editing the `docker-eps/bitcoin/bitcoin.conf` file. You can either google some bitcoin full-node onion addresses or add some from the wiki:

https://en.bitcoin.it/wiki/Fallback_Nodes#Tor_nodes

https://en.bitcoinwiki.org/wiki/Fallback_Nodes#Tor_nodes

```
addnode=btcnode1.onion
addnode=btcnode2.onion
```

---
## **Start the containers**
Once you have configured everything, start the docker containers.

This might take a while the first time as bitcoin core has to sync the blockchain. The time it takes to sync depends on hardware and internet speed, it could take a couple days or weeks to fully sync as each block has to be verified. Even when running in pruned mode you still need to download the full chain, the pruning will just remove the blocks that are no longer needed.

You may also see errors in the EPS logs as it won’t be able to connect to the bitcoin core node until it has finished syncing to the blockchain.
```sh
docker-compose up --build -d
```

Once the <u>bitcoind</u> container is running, you will need to create a wallet. EPS will import all your electrum addresses this wallet and check for incoming transactions to the wallet.
```sh
docker exec -t docker-eps_bitcoind_1 bitcoin-cli createwallet electrumpersonalserver
```

---
## **Connecting Electrum to EPS**
To connect Electrum to EPS, you will need the onion address.
```sh
cat /var/lib/tor/bitcoin-service/hostname
```

Start Electrum with this command to make sure it only connects to your EPS and routes all traffic over your Tor proxy.

If running locally:
```sh
electrum --oneserver --server 127.0.0.1:50002:s
```

If running on another machine:
```sh
electrum --oneserver --server myeps.onion:57283:s -p socks5:127.0.0.1:9050
```

---
## **Important Notes**
- If you are installing EPS on a different machine to Electrum, such as a VPS, you will want to connect to it over Tor.
- You NEED to add Tor nodes for bitcoind to sync.
- After the initial bitcoind sync you NEED to create a wallet.
- If your wallet has historical transactions, EPS will need to rescan for them (only once). To do this pruning needs to be disabled in the `bitcoin.conf` file. The `Dockerfile.eps` file also needs editing. Once the rescan is complete, revert the changes back to what they were.
```dockerfile
CMD [“electrum-personal-server”, “--rescan-date”, “<DD/MM/YYYY or block-height>”, “config.ini”]
```
- The time it takes for bitcoind to initially sync to the blockchain depends on internet speed and hardware. It could take up-to a week.
