# Nextcloud
I was notified that my iCloud was full so I figured I should host my own NAS since modern technology makes it easy enough. I'd researched the topic a few years prior, and since I've matured even further with general knowledge of CS I figured I could knock this project out in a weekend. I just needed something on my home network that I could easily move files to, off my Apple devices. I would settle for either being able to download them on my PC or view them on my iOS devices. I've been stoked that Nextcloud seems performant enough, is easy to deploy, easy to learn, very feature rich, and allows plugins. The features I'm most excited about are sharing, permissions, security, existing integrations like being an available widget to share to in iOs Photos. It seems matured because of the good documentation/support and how painless deployment was. 

# Hardware
I didn't have any hardware set up. I didn't want to spend money on new hardware because I didn't know if I was going to like this solution whatsoever; perhaps a prebuilt nas or paying for a different cloud service would seem more desirable after trying. I had an old CPU+Mobo+RAM working in an old case I was using to clone NVMe drives, and hadn't touched in years. I gutted another case that had a system that wouldn't post, so I had a PSU waiting for me in there. 
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
### sudo
- adding my user to the sudoers group
### openssh-server
- I configure my PC's ssh for hostname, ip, identity file, and ensure my PC's public key is in `~/.ssh/authorized_keys`
### /etc/hostname
### /etc/network/interfaces
- to use static ip:
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
### mdadm 
I knew I wanted the three hard drives to operate in software raid1 since the hardware step. I value the redundancy so if a drive fails I should have two more copies. With old drives I figure the likelihood of a single drive failing has increased, so this seemed inexpensive to gaurantee. It also gives upgrade paths for the future, suppose I got my hands on three huge hard drives or a couple large SSD's each at least a TB. These are a little expensive for me now, but in the future the only growing pain of upgrading will be transferring up to a TB off my hard drives. Half of the 2TB drive will be totally lost because I don't care to do the formatting, however I still felt I was making a good use of what I had laying around - I don't value that 1TB lost over the simplicity yet redundancy of my RAID array. 
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
### neofetch
### htop
### docker
- I knew I wanted to containerize nextcloud
- https://docs.docker.com/engine/install/debian/#install-using-the-repository
- I got to run `sudo docker run hello-world` without issue.

# Nextcloud
- Now that all the prerequisite stuff was out of the way I got to finally got the crux of the solution
- I started by making a space on the RAID array for docker, really the only thing I intend to use this harware for:  
- `sudo mkdir -p /1tbarray/docker`
```
brian@nas1:~$ cat /etc/docker/daemon.json
{
  "data-root": "/1tbarray/docker"
}
```
- verify:
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
- My cluster was already working with other stuff, and my cluster takes the traffic from my router and reverse proxies it to the correct service via ingress nginx
- https://github.com/kubernetes/ingress-nginx
- Getting this set up is non-trivial for a beginner, but is well documented and does occasionally require special steps when developers of software release patches that require intricate configuration of other software, as well as being an extremely variable and flexible system.
- I started by registering a subdomain to point to my ip: `nextcloud.brian2002.com`

- https://github.com/brian-chilin/live-k8s/blob/main/pe410a/nextcloud.yaml  
-  features four resources:
   1. A `Deployment` consisting of a single nginx container. It simply proxies the traffic from the ingress controller to my NAS outside of the cluster. This is basically a second proxy and seems convoluted, however it seems troublesome to have the ingress controller directly reverse proxy to external resources while the nginx container is extremely useful and simple enough if you are already familiar with nginx.  
   2. The configuration for this nginx running inside a container is specified in a `ConfigMap`. It's pretty barebones so here's the special lines:
      - `proxy_pass http://192.168.1.37:8080;` A system on my network could already directly access http://192.168.1.37:8080 to use Nextcloud
      - `client_max_body_size 8192M;` Lacking this makes transferring larger files difficult. 8GB was enough for me to get going with my images (typically 2 to 5MB) that were failing on nginx's default of 1MB. Perhaps it's enough for some short videos as well. This was one of the last configuration changes I made in the entire process this md describes as I was already uploading files from inside my network before setting up any kubernetes resources and I hadn't realized this field was wrong until testing uploads from outside my network
   3. A `Service` to make that configured deployment available within the cluster. It's super barebones.
   4. An `Ingress` to match my domain name to the service. I opted to use my existing cluster issuer to terminate ssl, visible in the ingress. At first I was using a staging issuer, as this required a bit of time configuring. The next simplest way to configure ssl termination would probably be inside the nginx container. Here are some of the lines concerning ssl termination:   
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nextcloud-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - nextcloud.brian2002.com
    secretName: nextcloud-brian2002-tls
```
- This wasn't exactly the order everything was implemented, but while coming up with the k8s yaml I was also tweaking some fields for Nextcloud's configuration back on the nas
  - `brian@nas1:~$ sudo nano /1tbarray/nextcloud/app/config/config.php`:
  - https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/config_sample_php_parameters.html
    - `trusted-domains` was tweaked rather early appending `nextcloud.brian2002.com` to the array
    - the following four were added after I was trying to log in and would hit a 'authorize browser' button and be left hanging on a spinning loading circle
```
  'trusted_proxies' =>
  array (
    0 => '192.168.1.202',
  ),
  'overwritehost' => 'nextcloud.brian2002.com',
  'overwriteprotocol' => 'https',
  'overwrite.cli.url' => 'https://nextcloud.brian2002.com',
```