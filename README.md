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

SSH in to your mainnet failover node (we're setting it up as testnet first to practice failing over).

Be sure to setup a non-root user before you get rolling. I user 2 non-root users, one w/ sudo access and a sol user w/out sudo. This is not strictly necessary, but make sure you're not using root at a minimum.

Add user helpers:
```
$ sudo adduser <username> # adds a non-sudo user
$ sudo adduser sol <username> # adds a sudo user 
$ sudo usermod -aG sudo <username> # grants `sudo` to an existing user
```




