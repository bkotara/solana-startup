# Solana Startup

Some other super useful links:
- [Official Docs](https://docs.anza.xyz/)
- [Educational Workshop](https://www.youtube.com/playlist?list=PLilwLeBwGuK6jKrmn7KOkxRxS9tvbRa5p)
- [Helius Setup Guide](https://www.helius.dev/blog/how-to-set-up-a-solana-validator)

## The Basic Idea

### What to Expect

This walkthorugh is intended to get you setup with operational testnet and maninet nodes as well as a mainnet failover.

It's not intended to be exhaustive. And the expectation is you understand things such as:
- how to setup a remote server (obviosuly)
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

### Setup Your Failover Node for Testnet

SSH to your mainnet failover node (we're setting it up as testnet first to practice failing over).

#### User Setup

Be sure to setup a non-root user before you get rolling. I user 2 non-root users, one w/ sudo access and a sol user w/out sudo. This is not strictly necessary, but make sure you're not using root at a minimum.

Add user helpers:
```
$ sudo adduser <username> # adds a non-sudo user
$ sudo adduser sol <username> # adds a sudo user 
$ sudo usermod -aG sudo <username> # grants `sudo` to an existing user
```

#### General Setup

Make sure you're node is up to date and has the proper packages (note: this step requires `sudo`)
```
$ sudo apt update
$ sudo apt upgrade
$ sudo apt install libssl-dev libudev-dev pkg-config zlib1g-dev llvm clang cmake make libprotobuf-dev protobuf-compiler
```

#### Security Recommendations

It's recommended to use [fail2ban](https://github.com/fail2ban/fail2ban) out of the box.
```
$ sudo apt install fail2ban
```

It's also recommended to only open the necessary ports for operation w/ a firewall - ufw is a great option. DigitalOcean has a great guide for ufw [here](https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands).
```
$ sudo apt install ufw
```


Here's a quick cheat sheet for what the validator needs:
```
$ sudo ufw allow 22/tcp # We need ssh to work!
$ sudo ufw allow 8000:10000/tcp # Really, we only need to allow the port range the validator is using; i.e. 8000:8020 (depending on your run script)
$ sudo ufw allow 8000:10000/udp # Same as the above
# sudo ufw allow 8900,8899/tcp # If you're allowing RPC access - not recommended for mainnet
$ sudo ufw enable
```

You can always check the status of ufw with:
```
$ sudo ufw status
```

#### Hard Drive Setup

Good guide on this [here](https://docs.anza.xyz/operations/setup-a-validator#hard-drive-setup).

Consolidated commands (assuming you have two NVMEs attached):
```
$ sudo mkfs -t ext4 /dev/nvme0n1
$ sudo mkfs -t ext4 /dev/nvme1n1
$ sudo mkdir -p /mnt/ledger
$ sudo mkdir -p /mnt/accounts
$ sudo mount /dev/nvme0n1 /mnt/ledger
$ sudo mount /dev/nvme1n1 /mnt/accounts

# Make your sol user the owner
$ sudo chown -R sol:sol /mnt/ledger
$ sudo chown -R sol:sol /mnt/accounts
```


#### Optimizations

Again, this can be found [here](https://docs.anza.xyz/operations/setup-a-validator#optimize-sysctl-knobs)

No need to worry about [this](https://docs.anza.xyz/operations/setup-a-validator#increase-systemd-and-session-file-limits) as we'll be setting up a service for the validator script.