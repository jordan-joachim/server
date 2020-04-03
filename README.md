# My server project

My description about hosting my own server at home, connected to the Internet by DSL.

I plan to use this machine for several things. First, host our growing collection of photos and videos. Second, host my nextcloud installation to synchronize files, calendar and contacts between our growing collection of computers and mobiles in the family. Third, i'm thinking about hosting my own website. I also would like to switch to a one node kubernetes installation to run this and much more as kube deployments.

My Server hardware:

 - New AMD 3000G processor with integrated graphics, 35W TDP, brand new from end of 2019
 - Mainboard with B450 chipset. These boards are only slightly more expensive than the B320 or B350 boards, but i found a website on Anadtech that says this need 4,8W TDP, compared to 6,8W of a B320/B350 chipset. And the B450 have much more functionality and connectors than the others.
 - 16 GB RAM. Sadly, my board can only use 2 memory modules, but still should be possible to upgrade to 32GB if necessary
 - A 500GB SSD disk i had laying around after switching to a NVMe disk on my main computer for OS install
 - Two WD Red 2TB NAS disks for main mirrored storage
 - bequiet 11 power supply
 - also a cheap wireless logitech keyboard with integrated tochpad.

I connected the server also to my television via HDMI to display and watch the pictures and video collections and connected my color laser printer via usb.

## OS installation

See comments regarding the basic OS installation in 
[linux-installation.md](linux-installation.md)

## failed - nextcloud with docker-compose
I tried then to run nextcloud via docker-compose, but failed.... I just did not get the nginx ingress to work
[nextcloud-docker-compose.md](nextcloud-docker-compose.md) 



