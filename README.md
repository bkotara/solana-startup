# Solana Startup

Some other super useful links:
- [Official Docs](https://docs.anza.xyz/)
- [Educational Workshop](https://www.youtube.com/playlist?list=PLilwLeBwGuK6jKrmn7KOkxRxS9tvbRa5p)
- [Helius Setup Guide](https://www.helius.dev/blog/how-to-set-up-a-solana-validator)

## The Basic Idea

### What to Expect

This walkthorugh is intended to get you setup with operational testnet and maninet nodes as well as a mainnet failover.

It's not intended to be exhaustive. And the expectation is you understand things such as:
- how to setup a remote server (obviously)
- how to use SSH
- the general basics of navigating a remote server with the command line
- bash scripting

Also, always feel free to turn to the [Solana Discord](https://solana.com/discord) when in doubt - there are many helpful people there.

### The Setup Flow

1. First, you need to have 3 servers ready to go
    - One will be your mainnet node (this should have the best specs of the three)
    - One will be your mainnet failover node
    - One will be your testnet node (this can have the worst specs, but it needs to be able to keep up with the network)
    - I, personally, recommend having these 3 servers in different geographical regions.
    - [Here](https://docs.anza.xyz/operations/requirements/) are the official requirements.
2. We're going to start on the failover node - we're going to configure it as a testnet validator. The reason we're doing this is to force practicing failover.
3. Once we have the mainnet failover node running a testnet validator, we're going to setup the testnet node, too.
4. Then we're going to failover to the testnet node.
5. Sanity check all is well...
6. Then we'll pretty much nuke the failover node (not completely) and get it setup for mainnet.
7. Finally, we'll setup your primary mainnet node.
8. At this point, you'll have 3 nodes running with one mainnet failover target for when needed (upgrades, etc).

## Testnet

### Setup Keys

Quick guide link [here](https://docs.anza.xyz/operations/setup-a-validator#create-keys).

Be sure to keep your authorized withrawer key safe! Official [docs](https://docs.anza.xyz/operations/best-practices/security#do-not-store-your-withdrawer-key-on-your-validator-machine) here.

### Setup Your Failover Node for Testnet

SSH to your mainnet failover node (we're setting it up as testnet first to practice failing over).

#### User Setup

Be sure to setup a non-root user before you get rolling. I user 2 non-root users, one w/ sudo access and a sol user w/out sudo. This is not strictly necessary, but make sure you're not using root at a minimum.

Add user helpers:
```bash
sudo adduser <username> # adds a non-sudo user
sudo adduser sol <username> # adds a sudo user 
sudo usermod -aG sudo <username> # grants `sudo` to an existing user
```

#### General Setup

Make sure you're node is up to date and has the proper packages (note: this step requires `sudo`):
```bash
sudo apt update
sudo apt upgrade
sudo apt install libssl-dev libudev-dev pkg-config zlib1g-dev llvm clang cmake make libprotobuf-dev protobuf-compiler
```

#### Security Recommendations

It's recommended to use [fail2ban](https://github.com/fail2ban/fail2ban) out of the box.
```bash
sudo apt install fail2ban
```

It's also recommended to only open the necessary ports for operation w/ a firewall - ufw is a great option. DigitalOcean has a great guide for ufw [here](https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands).
```bash
sudo apt install ufw
```


Here's a quick cheat sheet for what the validator needs:
```bash
sudo ufw allow 22/tcp # We need ssh to work!
sudo ufw allow 8000:10000/tcp # Really, we only need to allow the port range the validator is using; i.e. 8000:8020 (depending on your run script)
sudo ufw allow 8000:10000/udp # Same as the above
# sudo ufw allow 8900,8899/tcp # If you're allowing RPC access - not recommended for mainnet
sudo ufw enable
```

You can always check the status of ufw with:
```bash
sudo ufw status
```

#### Hard Drive Setup

Good guide on this [here](https://docs.anza.xyz/operations/setup-a-validator#hard-drive-setup).

Consolidated commands (assuming you have two NVMEs attached):
```bash
sudo mkfs -t ext4 /dev/nvme0n1
sudo mkfs -t ext4 /dev/nvme1n1
sudo mkdir -p /mnt/ledger
sudo mkdir -p /mnt/accounts
sudo mount /dev/nvme0n1 /mnt/ledger
sudo mount /dev/nvme1n1 /mnt/accounts

# Make your sol user the owner
sudo chown -R sol:sol /mnt/ledger
sudo chown -R sol:sol /mnt/accounts
```

#### Optimizations

Again, this can be found [here](https://docs.anza.xyz/operations/setup-a-validator#optimize-sysctl-knobs)

No need to worry about [this](https://docs.anza.xyz/operations/setup-a-validator#increase-systemd-and-session-file-limits) as we'll be setting up a service for the validator script.

#### Validator Install

Be sure to copy your validator identity and vote-account keypairs to your faiilover node. Reminder: your authorized withdrawer keypair should never be on your validator node - there is no need and it only adds security concerns.

It's recommended to build from source - and it's good to know how to - so that's what we'll do now.

Switch to your sol user:
```bash
su - sol
```

Make sure you have Rust installed for the `sol` user:
```bash
curl https://sh.rustup.rs -sSf | sh
source $HOME/.cargo/env
rustup component add rustfmt
rustup update
```

I like keeping things clean, so I put all of my repos and source code in `~/developer` - do as you please.
```bash
mkdir developer
cd developer
```

Grab the release tag you want to run - these can be found [here](https://github.com/anza-xyz/agave/releases).
```bash
export VERSION="2.0.18" # example
wget "https://github.com/anza-xyz/agave/archive/refs/tags/v$VERSION.tar.gz"
tar -xvzf "v$VERSION.tar.gz" && rm "v$VERSION.tar.gz"
cd "agave-$VERSION"
scripts/cargo-install-all.sh --validator-only ~/.local/share/solana/install/releases/v"$VERSION"
```

Now let's set the active solana release:
```bash
ln -snf "/home/sol/.local/share/solana/install/releases/v$VERSION" /home/sol/.local/share/solana/install/active_release
```

The above command links the release you just installed to the `active_release` path. This is just a convenient way to manage multiple releases - for upgrades etc.

Finally, let's add the `active_release` path to your sol user's path:
```bash
echo 'export PATH="/home/sol/.local/share/solana/install/active_release/bin":"$PATH"' >> ~/.bashrc
source ~/.bashrc
```

Now you can run `solana --version` and you should see whatever version you exported above.
```
solana-cli 2.0.18 (src:00000000; feat:607245837, client:Agave) # For the 2.0.18 example above
```

If you're instead planning on running Jito you can modify the above commands to checkout the proper git release (instead of downloading/unzipping) and build from there. [Here](https://jito-foundation.gitbook.io/mev/jito-solana/building-the-software#initial-setup) are Jito's offical docs.

#### Validator Service

Create a startup script as shown [here](https://docs.anza.xyz/operations/setup-a-validator#create-a-validator-startup-script).

*Note that this example script has `--rpc-port 8899` and does not have `--private-rpc` - if you plan on running like this in testnet, be sure you have ports 8899 and 8900 open in ufw.*

*Also note that this script is setting `--dynamic-port-range 8000-8020` so your ufw config only needs to open those up for upd/tcp instead of the 8000:10000 (shown [above](#security-recommendations)).*

It's a good idea to confirm your script runs by [executing it directly](https://docs.anza.xyz/operations/setup-a-validator#verifying-your-validator-is-working) and [checking the logs](https://docs.anza.xyz/operations/setup-a-validator#verifying-your-validator-is-working).

Now, let's make a [systemd unit](https://docs.anza.xyz/operations/guides/validator-start/#systemd-unit) to ensure the script runs in the background.

Put this content in `/etc/systemd/system/sol.service`:
```
[Unit]
Description=Solana Validator
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=1
User=sol
LimitNOFILE=1000000
LogRateLimitIntervalSec=0
Environment="PATH=/bin:/usr/bin:/home/sol/.local/share/solana/install/active_release/bin"
ExecStart=/home/sol/bin/validator.sh

[Install]
WantedBy=multi-user.target
```

To start it:
```bash
sudo systemctl enable --now sol
```

*Note that Environment here is pointing to your active release (set above).*

[Here's](https://www.digitalocean.com/community/tutorials/how-to-use-systemctl-to-manage-systemd-services-and-units) a good DigitalOcean guide on how to use systemctl.

Finally, configure log rotation as shown [here](https://www.digitalocean.com/community/tutorials/how-to-use-systemctl-to-manage-systemd-services-and-units).

At this point, you should have a running validator (via systemd) with log rotation enabled. You can tail logs to see it running or use the monitor command:
```bash
agave-validator --ledger /mnt/ledger/ monitor
```

You can also use the `catchup` command to watch your validator catch up to the network:
```bash
solana catchup -ut --our-localhost 8899  # be sure to use `-ut` here since this is a testnet validator
```