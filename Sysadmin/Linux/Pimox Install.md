Once you have Bookworm installed on all 3 Pis, configure them to use a consecutive range of IP numbers. Not a requirement, but just convenient during setup/troubleshooting. Here’s the host file from one of the Raspberries:

192.168.1.26     pimox1
192.168.1.27     pimox2
192.168.1.28     pimox3

Edit your host files (/etc/hosts) to contain just your 3 Pis and their IP addresses before you go any further. 
Remove everything else from the files. 
I also added these IP addresses to dns, so I can browse to them by name but it’s still important to add them to your /etc/hosts file because once we cluster them, they’ll be really chatty.

Next, we’ll need to add the Proxmox port repo. You can use the following commands.

First, add the gpg key for the repo:

curl https://mirrors.apqa.cn/proxmox/debian/pveport.gpg -o /etc/apt/trusted.gpg.d/pveport.gpg

Create the pveport.list file and populate with mirror info:

echo "deb [arch=arm64] https://global.mirrors.apqa.cn/proxmox/debian/pve bookworm port">/etc/apt/sources.list.d/pveport.list

Then you’ll need to update the repo cache. Let’s run an upgrade, too, just to make sure Bookworm is up to date:

apt update
apt -y upgrade
Now that we’re ready, let’s install the Proxmox port. You can do this on all 3 Pis concurrently.

apt install -y proxmox-ve 
Once the installation is completed, you’ll need to reboot all 3 hosts.

reboot
Once the Pis are back online, try the URL of the first Proxmox node at port 8006:

https://192.168.1.26:8006
The login is root and your root password. Hopefully, you should see the main Proxmox screen being served by your Raspberry!


We did it! One tiny bit of housekeeping while we’re thinking about it. For some reason, the repo definition needs to have the architecture defined. The install changes it back from the one we put in, so you might need to change it back.

vi /etc/apt/sources.list.d/pveport.list
and ensure that the square bracketed part is intact:

deb [arch=arm64] https://global.mirrors.apqa.cn/proxmox/debian/pve bookworm port
The next port will look at getting these new Proxmox nodes clustered and talking to each other.
