# PLEASE NOTE:
This is currently a work in progress.

# Setting up RAID 1

Assuming you have 2 additional unmounted disks installed, follow [this guide](https://www.linuxbabe.com/linux-server/linux-software-raid-1-setup)

On step 5, I believe that assumes you already set up your `/etc/fstab` (which further down below in the guide). If you haven't already done that, you might find that `/mnt/raid1` is empty (i.e. `/dev/md0` isn't mounted there). When I tried to mount with `~$ sudo mount /dev/md0 /mnt/raid1` I got the following error:
`mount: /mnt/raid1: can't read superblock on /dev/md0.`

To get around it, I did [the following](https://superuser.com/questions/993259/why-is-my-raid-1-disk-inactive):

`sudo mdadm --stop /dev/md0`
Then I noticed it was nvme0n1p1 that was still attatched when running `sudo mdadm --examine /dev/nvme0n1p1 /dev/nvme1n1p1`, so I did:

`sudo mdadm --assemble --force /dev/md0 /dev/nvme0n1p1`

After that, I was able to mount with:

`sudo mount /dev/md0 /mnt/raid1`

Now, I was able to complete step 5 and recover the txt file.

Editing the fstab file gave me some trouble so I skipped that part.

The final thing to do is change the ownership of the directory:

`sudo chown -R {username}: /mnt/`


# Setting Up bitcoind

## Method 1:
Install bitcoin daemon following [this guide](https://bitcoin.org/en/full-node#linux-instructions)

## Method 2:
Follow the guide on the [CLN github](https://github.com/ElementsProject/lightning#installation) which uses snap. After installing bitcoind, you can refer to step 1 for basic comands with `bitcoind`

## Running bitcoind
After running `bitcoind -daemon`, run `bitcoin-cli stop`, then (assuming you installed bitcoin in your home directory) `nano ~/.bitcoin/bitcoin.conf` and put this in:
```
[main]
txindex=1
datadir=/home/{username}/.bitcoin
dbcache=3000
rpcuser={username}
rpcpassword={password}
zmqpubrawblock=tcp://127.0.0.1:28332
zmqpubrawtx=tcp://127.0.0.1:28333
```

## Setting up bitcoind service
At the bottom of the tutorial in Method 2, it recommends starting bitcoind via a crontab. However we will do it through a service.

We will create a file called `/etc/systemd/system/bitcoind.service` and copy paste the the [bitcoind.service](/bitcoind.service) file while editing the {username} with your username.

To enable the service on startup, use

`sudo systemctl enable bitcoind.service `

Todo: update bitcoind.service script by adapting [bitcoin/bitcoin/blob/master/contrib/init/bitcoind.service](https://github.com/bitcoin/bitcoin/blob/master/contrib/init/bitcoind.service).

# Setting up CLN
Following the github guide [here](https://github.com/ElementsProject/lightning#installation) with a slight modification since we didn't use snap to install bitcoin-core:
```
sudo apt-get install -y software-properties-common
sudo add-apt-repository -u ppa:lightningnetwork/ppa
sudo apt-get install lightningd snapd
```

Next we make a config file to make sure that CLN is running on the raid drives. We use:

`mkdir /mnt/raid1/.lightning`

`nano /mnt/raid1/.lightning/config` 

and write the following:
```
network=bitcoin
bitcoin-rpcuser={username}
bitcoin-rpcpassword={password}
lightning-dir=/mnt/raid1/.lightning/
log-file=/mnt/raid1/.lightning/lightningd.log
```
and now to run CLN, we can try:

`lightningd --network=bitcoin --log-level=debug --conf=/mnt/raid1/.lightning/config`

We can confirm that there is now a db file located in `/mnt/raid1/.lightning/bitcoin/lightningd.sqlite3`

## Setting up lightningd service
Create a file called `/etc/systemd/system/lightningd.service` and copy paste the the [lightningd.service](/lightningd.service) file while editing the {username} with your username.

To enable the service on startup, use

`sudo systemctl enable lightningd.service`

## Setting up `hsm_secret` from a BIP39 word list

The `hsm_secret` is created when you first create the node. To back it up, see the section below. If you would like a pencil & paper copy, follow the steps below:

1. Use [trezor's mnemonic repo](https://github.com/trezor/python-mnemonic) (or any other tool you like) to generate 12-24 words. You can use the following script:
```
from mnemonic import Mnemonic

mnemo = Mnemonic("english")
words = mnemo.generate(strength=128)

print(words)
```
2. run `lightning-hsmtool generatehsm hsm_secret`
3. mv the `hsm_secret` file to  `/mnt/raid1/.lightning/bitcoin/`
4. `chmod 0400 /mnt/raid1/.lightning/bitcoin/hsm_secret`


## Backing up your CLN node

Follow the [CLN backup guide](https://github.com/ElementsProject/lightning/blob/master/doc/BACKUP.md)

Note that: 
> Recovery of the `hsm_secret` is sufficient to recover any onchain funds. Recovery of the `hsm_secret` is necessary, but insufficient, to recover any in-channel funds. 

There are 2 main things to back up: the `hsm_secret` and the `lightningd.sqlite3` database.

For the `hsm_secret`, I did the following steps:

```
mkdir -p  ~/.lightning/backups/bitcoin/
cp /mnt/raid1/.lightning/bitcoin/hsm_secret ~/.lightning/backups/bitcoin/hsm_secret
```

For the channel backup, I added the following line to the bottom of the `/mnt/raid1/.lightning/config`:

```
wallet=sqlite3:///mnt/raid1/.lightning/bitcoin/lightningd.sqlite3:/home/{username}/.lightning/backups/bitcoin/lightningd.sqlite3
```

This backup strategy combined with our raid1 setup means that in order to lose our funds, all 3 of our disks would have to fail.

# Useful CLN commands:

to confirm your current configs:

`lightning-cli listconfigs` 

# Useful CLN links:
* [Backing up your CLN node](https://github.com/ElementsProject/lightning/blob/master/doc/BACKUP.md)
* [Setting up TOR](https://github.com/ElementsProject/lightning/blob/master/doc/TOR.md)