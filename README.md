# Solana Startup

Some other super useful links:
- [Official Docs](https://docs.anza.xyz/)
- [Educational Workshop](https://www.youtube.com/playlist?list=PLilwLeBwGuK6jKrmn7KOkxRxS9tvbRa5p)
- [Helius Setup Guide](https://www.helius.dev/blog/how-to-set-up-a-solana-validator)
- [Identity Transition](https://pumpkins-pool.gitbook.io/pumpkins-pool)

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

### Setup your Failover Node for Testnet

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
# sudo ufw allow 8900,8899/tcp # If you're allowing RPC access and using a narrow range above - RPC is not recommended for mainnet
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

Be sure to copy your validator identity and vote-account keypairs to your failover node. Reminder: your authorized withdrawer keypair should never be on your validator node - there is no need and it only adds security concerns.

Also, at this point we can begin integrating the failover requirements, one of which is having a junk identity on both your primay and failover node. [Here](https://pumpkins-pool.gitbook.io/pumpkins-pool#generating-junk-identities) are the docs for this. Note that each node should have its own junk identity.

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

We'll want to modify the script to have the `--authorized-voter` flag set as shown [here](https://pumpkins-pool.gitbook.io/pumpkins-pool#validator-startup-script-modifications).

We'll also need to setup the necessary symlink for our primary identity as shown [here](https://pumpkins-pool.gitbook.io/pumpkins-pool#creating-identity-symlinks). Because we're going to use this as testnet primary and then failover to our actual testnet node, be sure to set your actual identity as the identity.json at this time.

[Here's](./testnet-validator.sh) an example run script for testnet with the failover modifications.

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

#### Monitoring your Validator

At this point, you should have a running validator (via systemd) with log rotation enabled. You can tail logs to see it running or use the monitor command:
```bash
agave-validator --ledger /mnt/ledger/ monitor
```

*Note: when you run the above command, the output will show you your active identity - this should match your actual testnet identity public key.*

You can also use the `catchup` command to watch your validator catch up to the network:
```bash
solana catchup -ut --our-localhost 8899  # be sure to use `-ut` here since this is a testnet validator
```

## Failover

#### Configure your Testnet Node

At this point, much of your work is rinse/repeat. Start from [here](#setup-your-failover-node-for-testnet) and get your testnet node ready to run.

This time, instead of symlinking your primary identity to the identity.json file, use the [unstaked identity](https://pumpkins-pool.gitbook.io/pumpkins-pool#creating-identity-symlinks); i.e. the junk identity.
```bash
ln -sf /home/sol/unstaked-identity.json /home/sol/identity.json
```

Once you have the node operational (the same run script should work) you can check the validator's status with the `monitor` command. This time, the identity should be that of your unstaked identity; i.e. the pub key of the junk identity we created on this node.

You can validate this against this command:
```bash
solana-keygen pubkey unstaked-identity.json # note that identity.json should work here, too, since it's a symlink to unstaked-identity.json
```

#### Setup the Failover Scripts

You can follow [these](https://pumpkins-pool.gitbook.io/pumpkins-pool#transition-process) example scripts to do the actual failover.

Note that these scripts will require SSH access from node to node as the `sol` user. You can always modify these scripts to run from your local box if you prefer.

On both nodes you should have `initiate_failover.sh` and `complete_failover.sh` scripts so you can switch both directions.

#### Failover to your Testnet Node

Before running the failover, it's important to check if it's caught up - otherwise you'll experience downtime. You can do this with the `catchup` command shown [above](#monitoring-your-validator).

If your testnet node is all caught up, you can initiate failover by first by running `./initiate_failover.sh` on your active node and then (sequentially) `./complete_failover.sh` on your inactive node.

Once these scripts have completed you can use the `monitor` command on each node to confirm the new identities.

You can also check your validator's IP in the `gossip` command (it should match your testnet node):
```bash
solana gossip -ut | grep $(solana-keygen pubkey validator-keypair.json)
```

If all is well, then you've successsfully failed over to your testnet node!

## Mainnet

On to mainnet...

#### Cleanup your Failover Node

Now, let's go back to your failover node and tear down the testnet validator.

We can stop the service:
```bash
sudo systemctl stop sol.service
```

Then, we can delete our testnet keypairs from this node - and the unstaked identity (we'll make a new one for mainnet).

Finally, we can cleanup the attached drives:
```bash
rm -rf /mnt/ledger/*
rm -rf /mnt/accounts/*
```

Remember to copy over your mainnet vote account and identity keypairs before proceeding. These should be different than your testnet keypairs!

#### Setup Jito-Solana

For mainnet, we're going to run [Jito-Solana](https://jito-foundation.gitbook.io/mev/jito-solana/building-the-software). Be sure to grab the latest mainnet release from [here](https://github.com/jito-foundation/jito-solana/releases) and run the build via the git checkout flow as shown [here](https://jito-foundation.gitbook.io/mev/jito-solana/building-the-software#initial-setup).

It's always good to sanity check your solana version after running an install and `active_release` update:
```bash
solana --version
```

The output should confirm you have jito-solana installed:
```
solana-cli 2.0.18 (src:2f2ef44d; feat:607245837, client:JitoLabs)
```

For your run script, you'll need a few updates for mainnet and jito. An example can be found [here](./mainnet-validator.sh). Be sure to add in actual [known validators](https://docs.anza.xyz/operations/guides/validator-start/#known-validators)!

You can use `agave-validator --help` to see various other run flags and descriptions. Values for the jito flags can be found [here](https://jito-foundation.gitbook.io/mev/jito-solana/command-line-arguments).

For completeness, I recommend starting your failover node as your primary mainnet validator (just like we did for testnet; i.e. using the primary identity instead of the junk one) and failing over to your actual mainnet node - practice is good - knowing things work is good.

Once you have your script as you like, you can restart the systemd service:
```bash
sudo systemctl start sol.service
```

*Note that grabbing an initial snapshot and the general boot process may take much longer on mainnet.*

#### Setup your Mainnet Node

Once again, we're at a rinse and repeat stage, but you're almost done.

All you need to do is follow the setup on your mainnet node, but using the same version of Jito-Solana you chose [above](#setup-jito-solana).

Be sure to use the same mainnet run script here, too. The only difference is you'll need to set your identity.json to the unstaked identity you create on the mainnet node.

Follow the steps in the [failover section](#failover) to get your failover scripts setup on your mainnet node. Be sure to modify the scripts on your failover node so they point to your mainnet node instead of your testnet node.

Once you have the scripts set and both nodes are caught up, you can transition identities on the two nodes.

From this point on you have 2 nodes, one with your actual identity set, running in the mainnet cluster and you're off to the races of collecting more stake!

## What Next?

You can check your validator's performance on various sites:
- [STAKEWIZ](https://stakewiz.com/)
- [Vortex](https://app.vx.tools/)

*Note that your validator may not show up until you have some stake and have made it into the leader schedule.*

If you'd like to stake with your validator, you can do so via the command line as shown [here](https://docs.anza.xyz/cli/examples/delegate-stake) or via the STAKEWIZ UI.