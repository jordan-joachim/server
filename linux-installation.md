# Basic Server installation

## Linux installation

Installation of Ubuntu 18.04 was straight forward, rebooted, but after grub, screen went blank.

After some tries, figured out that Ubuntu has a problem with the new graphics chip in the AMD 3000G.

First workaround was to add `nodemodeset` to the kernel line in `/etc/default/grub`

```
sudo sed -i "s/GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"/GRUB_CMDLINE_LINUX_DEFAULT="quiet nomodeset splash"/" /etc/default/grub
sudo update-grub
```

This allowed the Linux to boot in basic graphics mode 800x600.

## Getting the graphics to work

After that, i did some google research and found some indicators that only the newest kernel are getting support for the graphic part in the AMD 3000G CPU and installed the laster 5.5.8 kernel via ukuu

```
sudo add-apt-repository ppa:teejee2008/ppa
sudo apt-get update
sudo apt-get install ukuu
sudo ukuu --install v5.5.8 # or latest kernel
```

Then copied over the latest firmware available in the git repo

```
sudo apt-get -y install git
git clone https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git
cd linux-firmware/amdgpu/
cp raven* /lib/firmware/amdgpu/
```

Finally, i installed the AMD graphics driver. You won't find a driver for the AMD 3000G CPU, there is only a driver for Windows, so i went for the graphics driver suite that is offered for `Graphics`->`2nd generation Vega`->`Radeon VII` and downloaded the driver 19.50 for ubuntu package

```
untar latest amdgpu driver
./amdgpu-install
```

The install had some problems compiling drivers for both the 5.5.8 kernel and the HWE kernels of Ubuntu 18.04.4, but it installed the libraries without a problem.

After that, i removed the `nomodeset` from `/etc/default/grub` and finally, i got full nice graphics support on my HDMI connection.

## Preparing the server

First, installing some tools and ssh server

```
sudo apt-get -y install jq openssh-server aptitude joe curl vim axel lvm2
```

### Secure ssh for admin user

The next commands allow the admin user only to connect via ssh with keys. This disables ssh login with password.

On workstation:
```
scp .ssh 192.168.178.5:
ssh 192.168.178.5
```
On server:
```
sudo echo -ne "Match User <admin username>\n       PasswordAuthentication no" >> /etc/ssh/sshd_config
cd ~/.ssh; rm know*; cat id_rsa.pub > authorized_keys
```
## Setup LVM for storage disks

The two NAS disks i want to use in a mirrored LVM. I decided against using the BIOS raid functionality because that needs additional drivers in the OS and makes me dependant on the board and BIOS version. If i have to move the disk to a new hardware, LVM will be much easier

Ssh to the server:

This writes the partition table and prepares partitions to be used by LVM. My SSD is /dev/sda
```
sudo echo -e "o\nn\np\n1\n\n\nt\n8e\nw\n" | fdisk /dev/sdb  
sudo echo -e "o\nn\np\n1\n\n\nt\n8e\nw\n" | fdisk /dev/sdc
```
Create the LVM physical volumes on both partitions
```
sudo pvcreate /dev/sdb1 /dev/sdc1
```
Create a volume group from the two partitions
```
sudo vgcreate mirrorvg /dev/sdb1 /dev/sdc1
```
Create a logical volume as mirrored for the full size
```
sudo lvcreate -l100%VG -m1 -n mirrorlv mirrorvg
```
Check that everything is created correctly
```
sudo lvs # check logical volume
sudo lvdisplay -v # check logical volume
```
Look for the `LV Path` value in the output, like `/dev/mirrorvg/mirrorlv`. You need it in the next command
Format the logical volume
```
sudo mkfs.ext4 /dev/mirrorvg/mirrorlv
```
Create mount point, add entry in `/etc/fstab` and mount
```
sudo mkdir /mirror
export dm=$(realpath /dev/mirrorvg/mirrorlv | cut -d "/" -f 3)
export uuid=$(ls -l /dev/disk/by-uuid/ | grep "$dm" | cut -d " " -f 10)
sudo echo -ne "UUID=$uuid /mirror               ext4    errors=remount-ro 0       1\n" >> /etc/fstab
sudo mount -a
```

## Setup Samba

I want to use Samba to allow Kodi and other apps to access the picture and video collection on my storage

Install samba
```
sudo apt-get -y install samba samba-common  
```
Add two users for access and set passwords
```
sudo adduser --no-create-home --disabled-login --shell /bin/false <full access user>
sudo adduser --no-create-home --disabled-login --shell /bin/false <read only access user>
sudo pdbedit -L -v #list users
sudo smbpasswd -a <full access user>
sudo smbpasswd -a <read only access user>
```
Create shares and make the samba user owner
```
sudo mkdir -p /mirror/shares/media
sudo chown -R heimnetz: /mirror/shares/media
```

Add shares to samba config and recylce samba service
```
sudo cat <<EOF >> /etc/samba/smb.conf 
[media]
comment = Media
path = /mirror/shares/media
write list = <full access user>
valid users = <full access user>,<read only access user>
force user = <full access user>
EOF
sudo systemctl restart smbd.service
```

### TODO: get X11 remote access to work
```
aptitude install tightvncserver
vncserver #fill in password
vncserver -kill :1 #stops server
```

## Install printer SW and allow remote printing

I have an old Samsung color laser printer, but it's working and actually, new toner cartridges are fairly cheap.

Download driver, best have one on your disk...

Unpack and install
```
tar zxvf uld_V1.00.39_01.17.tar.gz
cd uld
bash install.sh
```
Allow remote access on cupsd
```
vim /etc/cups/cupsd.conf #add values
Listen 192.168.178.5:631 
systemctl restart cups
```

## Setup ufw

Install ufw (should probably already installed)
```
sudo aptitude -y install ufw
```
Setup basic rules for ssh, samba and CUPS
```
sudo ufw allow from 192.168.178.0/24 to any app Samba
sudo ufw allow from 192.168.178.0/24 to any app CUPS
sudo ufw allow from 192.168.178.0/24 to any app OpenSSH
sudo ufw default drop incoming
sudo ufw default allow outgoing
```

## Setup fail2ban

Install fail2ban, enable and start
```
sudo aptitude -y install fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```
### Todo: increasing ban times
still not working - To get increasing ban times, i added this filter config

```
cat <<EOF > /etc/fail2ban/filter.d/f2b-loop.conf
# Fail2Ban configuration file for subsequent bans
#
[INCLUDES]
before = common.conf
[Definition]
failregex = \]\s+Ban\s+<HOST>
ignoreregex = \[f2b-loop.*\]\s+Ban\s+<HOST>
EOF
```
now the jail.conf has to be changed
```
cat <<EOF > /etc/fail2ban/jail.local
[DEFAULT]
banaction=ufw
bantime  = 10800 ;3 hours
findtime  = 86400 ;1 day
maxretry = 5

[f2b-loop2]
enabled = true
filter = f2b-loop
bantime = 86400 ;1 day
findtime = 604800 ;1 week
logpath = /var/log/fail2ban.log
maxretry = 2

[f2b-loop3]
enabled = true
filter = f2b-loop
bantime	= 604800 ;1 week
findtime = 2592000 ;1 month
logpath = /var/log/fail2ban.log
maxretry = 3

[f2b-loop4]
enabled = true
filter = f2b-loop
bantime = 2592000 ;1 month
findtime = 15552000 ;6 months
logpath = /var/log/fail2ban.log
maxretry = 6

[f2b-loop5]
enabled = true
filter = f2b-loop
bantime = 15552000 ;6 months
findtime = 31536000 ;1 year
logpath = /var/log/fail2ban.log
maxretry = 9
EOF
```

# Final words

With that, the basic server is set up. Next step is to install nextcloud