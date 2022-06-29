# PLEASE NOTE:
This is currently a work in progress.

# Setting up RAID 1

Assuming you have 2 additional unmounted disks installed, follow [this guide](https://www.linuxbabe.com/linux-server/linux-software-raid-1-setup)

On step 5, I found the guide wasn't complete. When I removed one of the drives and rebooted, I found that `/mnt/raid1` was empty. When I tried to mount with `~$ sudo mount /dev/md0 /mnt/raid1` I got the following error:
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

# Setting up CLN
Following the github guide [here](https://github.com/ElementsProject/lightning#installation) with a slight modification since we didn't use snap to install bitcoin-core:
```
sudo apt-get install -y software-properties-common
sudo add-apt-repository -u ppa:lightningnetwork/ppa
sudo apt-get install lightningd snapd
```

Next we make a config file to make sure that CLN is running on the raid drives. We use:

`mkdir /mnt/raid1/.lightning`

`nano ~/.lightning/config` 

and write the following:
```
network=bitcoin
bitcoin-rpcuser={username}
bitcoin-rpcpassword={password}
lightning-dir=/mnt/raid1/.lightning/
```
and now to run CLN, we can try:

`lightningd --network=bitcoin --log-level=debug --conf=/home/{username}/.lightning/config`

We can confirm that there is now a db file located in `/mnt/raid1/.lightning/bitcoin/lightningd.sqlite3`

