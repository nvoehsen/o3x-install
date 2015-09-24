# Switching a Mac OS X powered computer to using ZFS, 2015-09

Here is what I did to boost the storage space in my home desktop computer. My machine is a 2012 Hackintosh, based on a Gigabyte Z77-DS3H board with 16 GB RAM running the latest Mac OS X. It has a 128 GB SSD for the operating system and a 2 TB spinning platter for user files. The latter recently filled up triggering my upgrade instincts.

## Options

I always had a soft spot for the ZFS file system because of its superior data protection features (checksumming), but never got a chance to play with it.

These were the upgrade paths available:

* Get a larger internal HD (e.g. 4 TB), format to HFS+, move the stuff over, be done. While this option is by far the easiest, there is also no fun in it. Additional drawbacks: All the data is on one single hard drive and gone in case the hard drive dies. It also uses a crappy filesystem (HFS+) which shouldn't be trusted with longtime storage of important data like the photo/video collection of my family.
* Get HDs and try out the [O3X](https://www.openzfsonosx.org) ZFS port for Mac OS X
* Get a NAS plus HDs, deploy FreeBSD and ZFS there and access user files over network. This requires buying, operating and maintaining more hardware, so I decided against it. Also, this is a nice fallback option, in case O3X didn't work reliably.

Option 2 is the way to go.

## ZFS setup

ZFS has many cool features, but the most important are:

* You can control the level of protection against hardware failure you want by choosing mirroring or different raid levels.
* The data stored in ZFS is always checksummed, so that data corruption becomes visible. In case of a redundant storage (like mirror or raidZ) this redundant information is also used to correct data which is found to be corrupt.
* Also, all drives can be connected in one logical ZFS pool. The whole storage space can then be made available in different file systems with different sizes independent of the sizes of the HDs.
* Compression leads to efficient storage of compressible data

Since I want to get protection against hardware failure, I chose to go with a three-disk raidZ configuration:

* One 2 TB drive (old)
* Two 3 TB drives

Since raidZ only works on equally sized disks (or partitions or files), I repartitioned the 3 TB drives into one 2TB partition for ZFS and a 1TB partition which is currently free space. The three 2TB slices together form the raidZ which ends up with a usable capacity of 4TB fast and resilient ZFS storage. 2TB are lost for the raidZ redundancy.
When I decide later to upgrade the 2TB drive to a larger size, the ZFS can grow to 3x 3TB without need to migrate the data.

## Setting up ZFS

There are several helpful pages for setting up O3X and ZFS in general, most imporantly the O3X instruction for [installation](https://openzfsonosx.org/wiki/Install) and [setting up a ZFS using O3X on a mac](https://openzfsonosx.org/wiki/Zpool#Creating_a_pool).

Several flags should be set to make ZFS work smooth with Mac OS. My line for creating the zpool was

    sudo zpool create -f -o ashift=12 \
    -O compression=lz4 \
    -O casesensitivity=insensitive \
    -O atime=off \
    -O normalization=formD \
    tank raidz /dev/disk1 /dev/disk2s1 /dev/disk3s1

Creating a ZFS filesystem inside the new pool is easy in comparison:

    zfs create tank/user


## Rsync the data

Rsync is a useful tool for performing the data transfer from HFS+ to ZFS. It is available in homebrew:

    brew tap homebrew/dupes
    brew install rsync

Syncing of files across different filesystems is a complex process and many things can go wrong:

* Resource forks can get lost
* Extended attributes can get lost
* File ownership or permissions can get lost

Therefore it is really important to either get Carbon Copy Cloner or be careful when choosing the rsync flags to be applied. After searching through the internet I used the following configuration:

    rsync -aNHAXx --fileflags --force-change <source> <dest> >& rsyncoutput.txt

The output reveals problems which occured during syncing, in my case a directory in the depths of the iTunes library having a reeeeaaally long filename due to someone entering all full names of the performing artists of an opera in the ID3 meta data.

I found that not all permissions work after transfer. I attribute that to Mac OS ACLs (access control lists) not being transfered. This also required some manual cleanup to make the copy work.

## ZFS status

The ZFS pool status after the data transfer looks fine:

    $ zpool list
    NAME   SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
    tank  5,44T  2,29T  3,14T         -    25%    42%  1.00x  ONLINE  -

A first scrub has also successfully completed:

    zpool status
     pool: tank
    state: ONLINE
     scan: scrub repaired 0 in 3h38m with 0 errors on Sun Sep 20 13:29:54 2015
    config:

     NAME                                            STATE     READ WRITE CKSUM
     tank                                            ONLINE       0     0     0
       raidz1-0                                      ONLINE       0     0     0
         media-XXXXXXXX-XXXX-XXXX-XXXXXXXXXXXX       ONLINE       0     0     0
         media-XXXXXXXX-XXXX-XXXX-XXXXXXXXXXXX       ONLINE       0     0     0
         media-XXXXXXXX-XXXX-XXXX-XXXXXXXXXXXX       ONLINE       0     0     0

## Mac OS specifics

### Disable Finder display of zpool

The finder displays the zpool as an ejectable medium with the "eject me"-button next to it. Since supposedly bad things happen when this button is pressed, I set the property "browse" to off (only needs to be done on the filesystem representing the whole zpool) which makes this volume not appear in the list of storage volumes in finder:

	zfs set com.apple.browse=off tank

There is also a property which helps some apple programs with ZFS, I set it to "on":

	zfs set com.apple.mimic_hfs=on tank

### Reset Spotlight

Spotlight works since O3X 1.3.1. I needed to do some administration tasks to get it to run properly:

* Stop Spotlight:
```bash
	mdutil -i off /Volumes/tank
	mdutil -i off /Volumes/tank/user
```

* Erase spotlight data
```bash
	mdutil -E /Volumes/tank
	mdutil -E /Volumes/tank/user
	cd /Volumes/tank && sudo rm -rf .Spotlight-V100
	cd /Volumes/tank/user && sudo rm -rf .Spotlight-V100
```

* Restart Spotlight
```bash
	mdutil -i on /Volumes/tank/user
```

### Time Machine

Time machine doesn't work on ZFS volumes. But ZFS has everything needed for efficient backups:

* Copy-on-write semantics
* snapshots
* Send and receive capability to transfer snapshots and filesystems to other machines

This leaves the following TODO open for me:

* Get a script that creates (and deletes) snapshots and transfers them to a backup server running ZFS


## Performance

I compared three devices w.r.t disk performance:

* The built in SSD
* A 1TB HFS+ partition located on one of the 3 TB drives to measure single disk performance
* The raidZ spanning three disks

File system performance was measured using [bonnie++](http://www.coker.com.au/bonnie++/). Get it installed and start a test with

```bash
	brew install bonnie++
	/usr/local/sbin/bonnie++ -s 999 -b -m <testname> -d <directory>
```

The results are as follows:

# SSD drive

<table border="3" cellpadding="2" cellspacing="1"><tr><td colspan="2" class="header"><font size=+1><b>Version 1.97</b></font></td><td colspan="6" class="header"><font size=+2><b>Sequential Output</b></font></td><td colspan="4" class="header"><font size=+2><b>Sequential Input</b></font></td><td colspan="2" rowspan="2" class="header"><font size=+2><b>Random<br>Seeks</b></font></td><td colspan="1" class="header"></td><td colspan="6" class="header"><font size=+2><b>Sequential Create</b></font></td><td colspan="6" class="header"><font size=+2><b>Random Create</b></font></td></tr>
<tr><td></td><td>Size</td><td colspan="2">Per Char</td><td colspan="2">Block</td><td colspan="2">Rewrite</td><td colspan="2">Per Char</td><td colspan="2">Block</td><td>Num Files</td><td colspan="2">Create</td><td colspan="2">Read</td><td colspan="2">Delete</td><td colspan="2">Create</td><td colspan="2">Read</td><td colspan="2">Delete</td></tr><tr><td colspan="2"></td><td class="ksec"><font size=-2>K/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>K/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>K/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>K/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>K/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td colspan="1"></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td></tr>
<tr><td rowspan="2" bgcolor="#FFFFFF" class="rowheader"><font size=+1>SSD</td><td class="size" bgcolor="#FFFFFF">999M</td><td>898</td><td>99</td><td>504473</td><td>33</td><td>507579</td><td>22</td><td>2123</td><td>99</td><td>+++++</td><td>+++</td><td>+++++</td><td>+++</td><td class="size" bgcolor="#FFFFFF">16</td><td>26658</td><td>91</td><td>+++++</td><td>+++</td><td>28598</td><td>84</td><td>23456</td><td>88</td><td>+++++</td><td>+++</td><td>15481</td><td>57</td></tr>
<tr><td class="size" bgcolor="#FFFFFF" colspan="1">Latency</td><td colspan="2">10082us</td><td colspan="2">20351us</td><td colspan="2">21497us</td><td colspan="2">4014us</td><td colspan="2">7us</td><td colspan="2">1962us</td><td class="size" bgcolor="#FFFFFF" colspan="1">Latency</td><td colspan="2">1160us</td><td colspan="2">25us</td><td colspan="2">1241us</td><td colspan="2">2175us</td><td colspan="2">12us</td><td colspan="2">13363us</td></tr>
</table>

### Single spinning disk

<table border="3" cellpadding="2" cellspacing="1"><tr><td colspan="2" class="header"><font size=+1><b>Version 1.97</b></font></td><td colspan="6" class="header"><font size=+2><b>Sequential Output</b></font></td><td colspan="4" class="header"><font size=+2><b>Sequential Input</b></font></td><td colspan="2" rowspan="2" class="header"><font size=+2><b>Random<br>Seeks</b></font></td><td colspan="1" class="header"></td><td colspan="6" class="header"><font size=+2><b>Sequential Create</b></font></td><td colspan="6" class="header"><font size=+2><b>Random Create</b></font></td></tr>
<tr><td></td><td>Size</td><td colspan="2">Per Char</td><td colspan="2">Block</td><td colspan="2">Rewrite</td><td colspan="2">Per Char</td><td colspan="2">Block</td><td>Num Files</td><td colspan="2">Create</td><td colspan="2">Read</td><td colspan="2">Delete</td><td colspan="2">Create</td><td colspan="2">Read</td><td colspan="2">Delete</td></tr><tr><td colspan="2"></td><td class="ksec"><font size=-2>K/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>K/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>K/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>K/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>K/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td colspan="1"></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td></tr>
<tr><td rowspan="2" bgcolor="#FFFFFF" class="rowheader"><font size=+1>SingleDrive</td><td class="size" bgcolor="#FFFFFF">999M</td><td>889</td><td>99</td><td>140568</td><td>9</td><td>147368</td><td>6</td><td>2128</td><td>99</td><td>+++++</td><td>+++</td><td>4033</td><td>27</td><td class="size" bgcolor="#FFFFFF">16</td><td>28457</td><td>81</td><td>+++++</td><td>+++</td><td>31097</td><td>77</td><td>6508</td><td>21</td><td>+++++</td><td>+++</td><td>1766</td><td>6</td></tr>
<tr><td class="size" bgcolor="#FFFFFF" colspan="1">Latency</td><td colspan="2">9874us</td><td colspan="2">100ms</td><td colspan="2">90811us</td><td colspan="2">4462us</td><td colspan="2">14us</td><td colspan="2">238ms</td><td class="size" bgcolor="#FFFFFF" colspan="1">Latency</td><td colspan="2">8310us</td><td colspan="2">26us</td><td colspan="2">895us</td><td colspan="2">144ms</td><td colspan="2">11us</td><td colspan="2">191ms</td></tr>
</table>


### RaidZ drive

<table border="3" cellpadding="2" cellspacing="1"><tr><td colspan="2" class="header"><font size=+1><b>Version 1.97</b></font></td><td colspan="6" class="header"><font size=+2><b>Sequential Output</b></font></td><td colspan="4" class="header"><font size=+2><b>Sequential Input</b></font></td><td colspan="2" rowspan="2" class="header"><font size=+2><b>Random<br>Seeks</b></font></td><td colspan="1" class="header"></td><td colspan="6" class="header"><font size=+2><b>Sequential Create</b></font></td><td colspan="6" class="header"><font size=+2><b>Random Create</b></font></td></tr>
<tr><td></td><td>Size</td><td colspan="2">Per Char</td><td colspan="2">Block</td><td colspan="2">Rewrite</td><td colspan="2">Per Char</td><td colspan="2">Block</td><td>Num Files</td><td colspan="2">Create</td><td colspan="2">Read</td><td colspan="2">Delete</td><td colspan="2">Create</td><td colspan="2">Read</td><td colspan="2">Delete</td></tr><tr><td colspan="2"></td><td class="ksec"><font size=-2>K/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>K/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>K/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>K/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>K/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td colspan="1"></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td><td class="ksec"><font size=-2>/sec</font></td><td class="ksec"><font size=-2>% CPU</font></td></tr>
<tr><td rowspan="2" bgcolor="#FFFFFF" class="rowheader"><font size=+1>raidz</td><td class="size" bgcolor="#FFFFFF">999M</td><td>150</td><td>99</td><td>277823</td><td>38</td><td>380660</td><td>50</td><td>421</td><td>99</td><td>+++++</td><td>+++</td><td>1431</td><td>12</td><td class="size" bgcolor="#FFFFFF">16</td><td>91</td><td>5</td><td>+++++</td><td>+++</td><td>92</td><td>1</td><td>87</td><td>4</td><td>+++++</td><td>+++</td><td>91</td><td>1</td></tr>
<tr><td class="size" bgcolor="#FFFFFF" colspan="1">Latency</td><td colspan="2">55520us</td><td colspan="2">3179us</td><td colspan="2">2616us</td><td colspan="2">19770us</td><td colspan="2">90us</td><td colspan="2">324ms</td><td class="size" bgcolor="#FFFFFF" colspan="1">Latency</td><td colspan="2">251ms</td><td colspan="2">58us</td><td colspan="2">281ms</td><td colspan="2">448ms</td><td colspan="2">15us</td><td colspan="2">298ms</td></tr>
</table>


The bandwidth numbers are along the expected results with the RaidZ yielding about twice the throughput of a single disk. The mass file creation/deletion tests did take absurdly long with RaidZ, I suspect that something about the "-b" flag (issues fsync after each write operation) in bonnie++ doesn't play well with the ZFS implementation. Since I don't regularly create thousands of files per second I don't really care.

## Observations and conclusion

During the early ZFS exploration phase, I copied my home directory to a ZFS pool using only an old single 750 GB drive. The ZFS had no compression enabled. Even this single-disk ZFS felt several times faster on some operations than the previous HFS+ setup. Especially Mail (10k plus mails) and iPhoto (20k plus photos) were really fast. And this is without raidz or compression on a rather old and slower hard drive.

So is it worth it for a home setup? Currently, everything seems to work flawlessly. I am still waiting for the first program to crash on startup with a cryptic error message. The gained speed is really nice, but most important for me is the protection against HD failure. So far it seems like a nice improvement for performance-hungry desktop Mac users or hackintoshers. The loss of Time Machine is not really that important since the features can easily be replicated with the built-in ZFS mechanisms. The mod is definitely not something for command-line agnostics.
