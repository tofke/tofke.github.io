---
layout: default
title: Basic VPS setup
---
# VPS-for-MN
Virtual Private Server setup for hosting "Dash/PIVX" like Master Nodes.

This step by step guide is primarly focusing on Ubuntu systems for now.

I'll update it to get you information on other Linux flavours over time (Debain and RedHat based for example).

## 1. The first step is to get yourself a decent server
See <a href=https://github.com/tofke/VPS-for-MN/wiki/Choose-a-VPS-provider>this Wiki page for some examples of VPS providers</a>.

## 2. When you connect to your new server, you should update the basic OS ! 
Most freshly deployed preinstalled operating systems are prebuild images wich are probably not running the latest versions of software packages. On a debian based system (like Ubuntu or Mint), the upgrade process is made like this (as root) : 
```
apt-get update && apt-get upgrade -y
```
After this first update/upgrade process, you should reboot your server, as chances are you downloaded a new Linux kernel ! 
As the root user, just type "<b>reboot</b>", and you will be disconnected. Reboots on virtual machines are very fast, as there are no hardware checks to be done (BIOS POST, RAID firmware ans so on). You can generally reconnect after less than a minute or so.
```
reboot
```

## 3. Add swap space (virtual memory on disk like 'pagefile.sys' on Windows) : 
### 3.1 Create a swapfile if you don't have a dedicated partition for that : 
```
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```
Depending on your needs, you could add more or less swap : 4G stands for 4 GigaBytes, so if for example you just want a 2 Gb swapfile you would have typed "fallocate -l 2G /swapfile" instead.
### NOTE : type this last command as root, not with "sudo" !
Become root : 
```
sudo su -
```
Add the swapfile in your fstab to have it enabled automatically after a reboot : 
```
echo "/swapfile   none    swap    sw    0   0" >> /etc/fstab
```

### 3.2 Tell Linux not to swap unless necessary, for example in case of RAM usage over 90% :  
Check current settings with "sysctl -a|grep swappiness", and if needed change it (as root) :
```
sysctl -a|grep swappiness
```
For example, you could have this kind of output : 

(this is an example, don't paste these lines)
```
root@mine:~# sysctl -a|grep swappiness
sysctl: lecture de la clé « net.ipv6.conf.all.stable_secret »
sysctl: lecture de la clé « net.ipv6.conf.default.stable_secret »
sysctl: lecture de la clé « net.ipv6.conf.enp4s0.stable_secret »
sysctl: lecture de la clé « net.ipv6.conf.lo.stable_secret »
vm.swappiness = 60
```
(ignore the other messages beginning with "sysctl:" here)

--> let's change that in "/etc/sysctl.conf" to "vm.swappiness=10", and run "sysctl -p" to reload : 
```
echo "vm.swappiness=10" >> /etc/sysctl.conf && sysctl -p
```
This last command should answer with the changed parameter, for example : 

(again, this is just an example, don't paste "root@mine:#" in your shell)
```
root@mine:~# echo "vm.swappiness=10" >> /etc/sysctl.conf && sysctl -p
vm.swappiness = 10
```


## 4. Secure access to your server :
Some masternode "experts" will tell you in their documentation or Discord channel to install and run their stuff as root ... i <ai>STRONGELY</ai> discourage you to do so ! You should take a bit more precautions with your server and create a dedicated user for each task. For example, i am running +10 masternodes on a single VPS (2 Gb of RAM and 2 CPU treads). No need to "destroy the server and redo a full installation" for a simple blockchain syncronisation problem ! If they knew what the're doing, they should not just give you such dummy advises. This sounds so unprofessional to me ... needless to say i don't listen to such advices as i run many masternodes on a single VPS ... but well you do what you want, if it is easyer for you to just run an installation script. I suggest to learn basics of Linux administration however to understand what commands you type in !

### 4.1 Create a user : 
```
useradd -m -s /bin/bash $USERNAME
```
Of course, in the line above you replace "<i>$USERNAME</i>" with whatever you want (without the $ sign), like "admin", "operator", "god", "me" ... 

### 4.2 Add this new user in the "adm", "systemd-journal" and "sudo" groups : 

This particular system group will let this user do administrative tasks like installing software, starting services, administer the firewall rules and so on. Running a command with the word "sudo" before it is like "becomming root" in short.
```
usermod -a -G adm,systemd-journal,sudo $USERNAME
```
Again, in the line above replace "<i>$USERNAME</i>" with the name you just created previouly.

### 4.3 Once this new user is created, set a password for him with the passwd command : 
```
passwd username
```
Again, in the line above you replace "<i>username</i>" with the name you just created previouly. This command will ask you to enter the new password twice (don't worry if you see no stars or dots or whatever in the prompt as you type a password, this is normal behaviour or this command on Unix systems). An example (not to be pasted in your terminal ;p) of the above 3 steps : 
```
root@vps554524:~# useradd -m -s /bin/bash tof
root@vps554524:~# usermod -a -G adm,systemd-journal,sudo tof
root@vps554524:~# passwd tof
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully
root@vps554524:~# groups tof
tof : tof adm sudo systemd-journal
```
As you can see, nothing apears after the "password:" prompts.

<B>Information on groups : </B>
* adm - allows access to log files in /var/log without using sudo
* systemd-jounral - allows access to the log via journalctl without using sudo
* sudo - allows access to run commands as the super user

### 4.4 check connection with that user (connect with ssh from wherever you want)

An even more secure option would be to enable remote connections with an ssh key ... i'll explain that later when i'll have more time to update this documentation. Basically, on a MAC or Linux at home (not on a public computer, do this only on your own), create a keypair with "ssh-keygen" and then install it to the remote user's home folder with "ssh-copy-id user@remotehost" ... same syntax as an ssh connection : "ssh user@host". I recommend using MobaXterm for Windows users, as this software let's you keep your settings if you enable a local persistent home (this will be documented here too one day).

### 4.5 Optionally (recommended), disable remote root login :

Do this only if the connexion with your new user works ! 
```
sudo vi /etc/ssh/sshd_config
```
--> change "PermitRootLogin" to "no" \
--> change "PasswordAuthentication" to "no" \
--> change "ChallengeResponseAuthentication" to "no" \
(you can use nano instead of vi if you prefer)

Restart the ssh server : 
```
sudo systemctl restart sshd
```

## 5. install dependencies for bitcoin : 
(All coins out there are forks of the original Bitcoin Core software stack)
### 5.1 Add some components used by most of the masternode software 
Unless you compile yourself, then you probably know what you do and won't need this documentation. 

Always refer to the dev team install notice when possible for needed dependencies.
```
sudo apt install software-properties-common
sudo add-apt-repository ppa:bitcoin/bitcoin
sudo apt update && sudo apt upgrade
sudo apt install libssl1.0.0 libboost-system1.58.0 libboost-filesystem1.58.0 libboost-program-options1.58.0 libboost-thread1.58.0 libdb4.8++  libzmq5 libminiupnpc10
```
### 5.2 optionnal (but recommended) :
To enable usage of the QT graphical user interfaces.
```
sudo apt install qt protobuf libqrencode 
```
### 5.3 optional : add dev libs and compilation tools
Only if you want to compile things yourself ... 
```
sudo apt-get install build-essential libtool autotools-dev autoconf pkg-config libssl-dev libevent-dev
sudo apt install libssl-dev libboost-all-dev libminiupnpc-dev libzmq-dev libdb4.8++-dev 
```
section to be competed ... 

## 6. Add some more tools to ease administration : 
```
sudo apt install htop glances byobu jq -y
```
(did you notice i use sudo again : connect as root only when needed)
* htop : a better top with easy viewable ressource usage
* glances : an even better tool to view ressources "at a glance" (hence the name, comes from HP-UX 'glance' tool)
* byobu : a better 'screen' (a terminal multiplexer, wich allow to keep a detached session in backgroud or share your screen with multiple sessions)
* jq : jq is like sed for JSON data (a usefull tool to parse JSON data)

### Tools for Windows (for remote administration of your VPS)
* MobaXterm : the best Xwindows / SSH client (and much more) i know for Windows : https://mobaxterm.mobatek.net/download.html

### Feel free to consider a donation if this helped
<b>Donation = motivation</b> :100: 
* BTC : 1FMXSYsd1aDLEBAZ65c8jpoJovNusn5aQb
* ETH : 0x2Bd3fb2040341eeed54fD7e5ffc73Ca49B5Bf0F5
* LTC : ltc1q4epe4vp4n9a84ltc8jxy8j67xphntyn78gme99
* ZEN : zncRXepm3mCcxh1jSySYUQ55HYvZrWK9FAV

Thank you for reading this guide ! 

Have fun with crypto ;-)
