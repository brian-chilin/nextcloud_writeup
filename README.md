# Nextcloud
I was notified that my iCloud was full so I figured I should host my own nas since modern technology makes it easy enough. I'd researched the topic a few years prior, and since I've been even further matured in my general knowledge of CS I figured I could knock this project out in a weekend. I just needed something on my home network that I could easily move files to off my Apple devices. I would settle for either being able to download them on my PC or view them on my iOS devices. I've been stoked that Nextcloud seems performant enough, is easy to learn/deploy, is very feature rich, and allows plugins. The features I'm most excited about are sharing, permissions, security, existing integrations, and how there's so much control and support product support for something I own. It seems matured in how fast I was able to deploy as well. 

# Hardware
I didn't have any hardware set up. I didn't want to spend money on new hardware because I didn't know if I was going to like this solution whatsoever; perhaps a prebuilt nas or paying for a different cloud service would seem more desirable after trying. I had an old CPU+Mobo+RAM working in an old case I was using to clone NVMe drives, and hadn't touched in years. I gutted this other case that had a system that wouldn't post, so I had a PSU waiting for me. 
```
brian@nas1:~$ neofetch
       _,met$$$$$gg.          brian@nas1
    ,g$$$$$$$$$$$$$$$P.       ----------
  ,g$$P"     """Y$$.".        OS: Debian GNU/Linux 12 (bookworm) x86_64
 ,$$P'              `$$$.     Kernel: 6.1.0-13-amd64
',$$P       ,ggs.     `$$b:   Uptime: 2 days, 8 hours, 7 mins
`d$$'     ,$P"'   .    $$$    Packages: 379 (dpkg)
 $$P      d$'     ,    $$P    Shell: bash 5.2.15
 $$:      $$.   -    ,d$$'    Terminal: /dev/pts/2
 $$;      Y$b._   _,d$P'      CPU: AMD A6-7400K Radeon R5 2C+4G (2) @ 3.500GHz
 Y$$.    `.`"Y$$$$P"'         GPU: AMD ATI Radeon R5 Graphics
 `$$b      "-.__              Memory: 777MiB / 6871MiB
  `Y$$
   `Y$$.
     `$$b.
       `Y$$b.
          `"Y$b._
              `"""
```
I scrapped together the following:  
- 1x small noname ssd  
- 2x WD Blue 1TB hard drive  
- 1x WD Blue 2TB hard drive  
This was most of my hardware laying around I was willing to use, and serves as a boot drive and three storage drives.

# Debian
I've been familiarizing with Debian for years now and the reliability and support make it a no-brainer. After using the installer and being logged in as root I set up the following
#### `sudo` and adding my user to the sudoers group
#### `openssh-server`
- I configure my PC's ssh for hostname, ip, identity file, and ensure my PC's public key is in `~/.ssh/authorized_keys`
#### `/etc/hostname`
#### `/etc/network/interfaces` to use static ip
```
brian@nas1:~$ cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug enp1s0
iface enp1s0 inet static
  address 192.168.1.37
  netmask 255.255.255.0
  gateway 192.168.1.1
  dns-nameservers 192.168.1.1 8.8.8.8
```
#### `mdadm` 
I knew I wanted the three hard drives to operate in software raid1 since the hardware step. I value the redundancy so if a drive fails I should have two more copies. With old drives I figure the likelihood of a single drive failing has increased, so this seemed inexpensive to gaurantee. It also gives upgrade paths for the future, suppose I got my hands on three huge hard drives or a couple large SSD's each at least a TB. These are a little expensive for me now, but in the future the only growing pain of upgrading will be transferring up to a TB off my hard drives.
```
$ sudo mdadm --zero-superblock /dev/sdb /dev/sdc /dev/sdd
$ sudo mdadm --create --verbose /dev/md0 --level=1 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd
$ watch -n .5 cat /proc/mdstat
```
After that was all finished (~2 hrs) the following command was successful albeit this output is a couple days later at the time of this writeup:
```
brian@nas1:~$ sudo mdadm --detail /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Sat Nov  8 17:11:36 2025
        Raid Level : raid1
        Array Size : 976630464 (931.39 GiB 1000.07 GB)
     Used Dev Size : 976630464 (931.39 GiB 1000.07 GB)
      Raid Devices : 3
     Total Devices : 3
       Persistence : Superblock is persistent

     Intent Bitmap : Internal

       Update Time : Mon Nov 10 17:58:51 2025
             State : clean
    Active Devices : 3
   Working Devices : 3
    Failed Devices : 0
     Spare Devices : 0

Consistency Policy : bitmap

              Name : nas1:0  (local to host nas1)
              UUID : 1edd3fc2:ca977bbe:3b04b9ea:dcb19e20
            Events : 1526

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
```
`sudo mkfs.ext4 /dev/md0`  
After editing `/etc/fstab` I have it always mount to `/1tbarray`  
#### `neofetch`
#### `docker`
- I knew I wanted to containerize nextcloud
- https://docs.docker.com/engine/install/debian/#install-using-the-repository
- I got to run `sudo docker run hello-world` without issue.

# Nextcloud
- Now that all the prerequisite stuff was out of the way I got to finally got the crux of the solution
- I started by setting the following
- `sudo mkdir -p /1tbarray/docker`
```
brian@nas1:~$ cat /etc/docker/daemon.json
{
  "data-root": "/1tbarray/docker"
}
```
```
brian@nas1:~$ sudo docker info | grep "Root Dir"
 Docker Root Dir: /1tbarray/docker
```
- Now to configure a `nextcloud` container, but first a `mariadb` container for it to store data in
```
brian@nas1:~$ cat /1tbarray/nextcloud/docker-compose.yml
services:
  db:
    image: mariadb:11
    restart: unless-stopped
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    volumes:
      - ./db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=SECRET
      - MYSQL_PASSWORD=SECRET
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud

  app:
    image: nextcloud
    restart: unless-stopped
    ports:
      - 8080:80
    environment:
      - MYSQL_PASSWORD=SECRET
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db
    volumes:
      - ./app:/var/www/html

volumes:
  db:
  app:
```
- At this point I could already log in and make my admin account and start uploading files from within my network!

# kubernetes
- towrite
