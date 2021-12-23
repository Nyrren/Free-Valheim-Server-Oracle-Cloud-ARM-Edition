# Free-Valheim-Server-Oracle-Cloud-ARM-Edition
This is a walkthrough of how to get a Valheim server running with Oracle Cloud's new ARM Ampere always free tier.

Using Oracle Cloud's Ampere instances give us access of up to 4 cores and 24 GB[!!!] of RAM.  The only problem is that Valheim Server doesn't natively have AARCH64/ARM64 support, so we'll have to use tools like [Box86](https://github.com/ptitSeb/box86) and [Box64](https://github.com/ptitSeb/box64) to emulate x86_64 programs.

Thankfully a lot of work has been put into getting Valheim Server to run on the RPi4.  We'll use some of that methodology to get our hosted Ubuntu solution running.

## References

[Akridge's Guide](https://github.com/akridge/Valheim-Free-Game-Server-Setup-Using-Oracle-Cloud) - Article using Oracle Cloud for free using x86_64, using LinuxGSM or AMP as a game server.

[Emmet's Guide for RPI4](https://pimylifeup.com/raspberry-pi-valheim-server/) - Article on how to host Valheim Server on the RPi4 using Steam.

## Setup Instance in Oracle Cloud

- Sign up for [Oracle Cloud](https://www.oracle.com/cloud/free/).

- Create a VM Instance.

![Create Instance](https://github.com/Nyrren/Free-Valheim-Server-Oracle-Cloud-ARM-Edition/blob/main/docs/CreateInstance.jpg)

- Select your OS and your Shape (Ubuntu 20.04, ARM Ampere 2 Cores w/ 12 GB of RAM).

![Select Shape](https://github.com/Nyrren/Free-Valheim-Server-Oracle-Cloud-ARM-Edition/blob/main/docs/SelectShape.jpg)

- Generate a and save your private Key (alternatively you can generate your own it PuTTYgen, but that is beyond the scope of this tutorial).

![Save Key](https://github.com/Nyrren/Free-Valheim-Server-Oracle-Cloud-ARM-Edition/blob/main/docs/SaveKey.jpg)

- Allow transport encryption for the boot volume.

![In Transit Encrypt](https://github.com/Nyrren/Free-Valheim-Server-Oracle-Cloud-ARM-Edition/blob/main/docs/InTransitEncrypt.jpg)

- Your instance will start to provision and soon will look like this:

![Running Instance](https://github.com/Nyrren/Free-Valheim-Server-Oracle-Cloud-ARM-Edition/blob/main/docs/RunningInstance.jpg)

- Open UDP 2456,2457 in Security List for 0.0.0.0/0 (in the Subnet section), there's a link on the subnet line in the screenshot above.

![Ingress Rules](https://github.com/Nyrren/Free-Valheim-Server-Oracle-Cloud-ARM-Edition/blob/main/docs/IngressRules.jpg)

## Login via SSH

- I use PuTTY, you can get it [here](https://www.putty.org/).

- Make note of your public IP address (found in your Oracle Cloud Instance info), and [attach](https://docs.oracle.com/en/cloud/paas/goldengate-cloud/connect-to-cloud-win-putty/) your Private Key that you downloaded when the instance was first setup (Oracle Cloud default user will be ```ubuntu```, so use that).

![Putty SSH](https://github.com/Nyrren/Free-Valheim-Server-Oracle-Cloud-ARM-Edition/blob/main/docs/PuTTySSH.jpg)

- You may get a warning the first time you login with PuTTY, acknowledge it and we should be on our new server.

![Putty SSH](https://github.com/Nyrren/Free-Valheim-Server-Oracle-Cloud-ARM-Edition/blob/main/docs/SSHSession.jpg)


## Update Ubuntu
We'll want to update Ubuntu first.
```
sudo apt update
sudo apt upgrade -y
```

### Open Ports
Next we have to open ports from the server.  Oracle Cloud doesn't utilize UFW for the firewall (disabled by default), it uses the IP tables to open ports.
```
sudo iptables -I INPUT 6 -m state --state NEW -p UDP --dport 2456 -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -p UDP --dport 2457 -j ACCEPT
```
The changes to IP tables must then be saved.
```
sudo netfilter-persistent save
```

### Install Prereqs
Grab the libsdl2-2.0-0 and install git, as we'll need the files to build Box86 and Box64, which are essential for this to work as it provides the x86 to ARM 64-bit emulation. We'll need cmake as well as Box86 and Box64 will need to be compiled.
```
sudo apt install libsdl2-dev
sudo apt install libsdl2-2.0-0
sudo apt install git build-essential cmake
```

### Install [Box86](https://github.com/ptitSeb/box86)
We'll ensure we are in the home directory, then clone the latest Box86 repository.
```
cd ~
git clone https://github.com/ptitSeb/box86
```
Then add the 32bit ARM architecture next.
```
sudo dpkg --add-architecture armhf
sudo apt update
sudo apt install gcc-arm-linux-gnueabihf libc6:armhf libncurses5:armhf libstdc++6:armhf
```
Finally we'll compile Box86.  Yes, we can cmake with the ```-DRPI4ARM64=1``` flag interestingly enough.
```
cd ~/box86
mkdir build
cd build
cmake .. -DRPI4ARM64=1 -DCMAKE_BUILD_TYPE=RelWithDebInfo
make -j$(nproc)
sudo make install
sudo systemctl restart systemd-binfmt
```

### Install [Box64](https://github.com/ptitSeb/box64)
Go back to the home directory and clone the latest Box64 repository.
```
cd ~
git clone https://github.com/ptitSeb/box64.git
```
Next we'll compile Box64. Again, we can cmake with the ```-DRPI4ARM64=1``` flag.
```
cd ~/box64
mkdir build
cd build
cmake .. -DRPI4ARM64=1 -DCMAKE_BUILD_TYPE=RelWithDebInfo
make -j$(nproc)
sudo make install
sudo systemctl restart systemd-binfmt
```

### Reboot
```
sudo reboot
```

### Install Steam
The Steam installation is next, first we'll want to create the install directory.
```
mkdir /home/ubuntu/steamcmd
cd /home/ubuntu/steamcmd
```
Download the necessary files.
```
curl -sqL "https://steamcdn-a.akamaihd.net/client/installer/steamcmd_linux.tar.gz" | tar zxvf -
```
### Install and Verify
Running steamcmd.sh will complete the installation and put you in the steamcmd shell.  You can type quite to exit the shell once it is done.
```
./steamcmd.sh
quit
```

**NOTE:** If you get the following error message: ```Loading Steam API...Failed to init SDL priority manager: SDL not found``` don't fret.  

We have the 64-bit version.  Steam is complaining about the absence of the 32-bit version, but we only need to run ```steamcmd.sh``` to install the Valheim server.  We won't be playing 32-bit games on this server.



### Install Valheim Server
Next is to install the Valheim server components, note that this install will place the install directory in ```/home/ubuntu/valheim_server```.
```
./steamcmd.sh +@sSteamCmdForcePlatformType linux +force_install_dir /home/ubuntu/valheim_server +login anonymous +app_update 896660 validate +quit
```
Before starting the server, modifications are needed to the ```server_server.sh``` script. I use [Nano](https://www.nano-editor.org/dist/latest/nano.html).
```
nano /home/ubuntu/valheim_server/start_server.sh
```
You'll want to replace the line in the server_server.sh script with the one below. 

```<IMPORTANT>``` Change the server info to your own! ```<IMPORTANT>```
```
./valheim_server.x86_64 -nographics -batchmode -port 2456 -public 0 -name "My Server Name" -world "MyWorldName" -password "MySecretPassword" -savedir "/home/ubuntu/valheim_data"
```

From the [Valheim Wiki](https://valheim.fandom.com/wiki/Hosting_Servers) & [Docker Server Notes](https://github.com/lloesche/valheim-server-docker/issues/22):

```-nographics``` When you run this in batch mode, it does not initialize the graphics device. 

```-batchmode``` Run Unity in batch mode. In batch mode, Unity runs command line arguments without the need for human interaction.

```-port``` This is the port that Valheim will connect to the server on.

```-public```  This shows (1) or hides (0) your server from the community server browser.

```-name``` The name of your server as it appears on the in-game community server browser.

```-world``` The name of the Valheim World you wish to host on the server. This is also the name of the world file.

```-password``` The password used to access the Valheim world. Minimum password length is 5 characters

```-savedir``` Overrides the default save path

### Start the Valheim Server
Navigate to the Valheim Server directory and start the server.  There will be errors, and it may take awhile if it's building a new world for the first time.
```
cd /home/ubuntu/valheim_server/
./start_server.sh
```
You'll see the following errors and more, again don't fret.
```
Error loading needed lib /home/ubuntu/valheim_server/valheim_server_Data/Managed/UnityEngine.dll.so
Warning: Cannot dlopen("/home/ubuntu/valheim_server/valheim_server_Data/Managed/UnityEngine.dll.so"/0xa4d19bc0, 101)
Error loading needed lib /home/ubuntu/valheim_server/valheim_server_Data/Managed/mono/aot-cache/amd64/UnityEngine.dll.so
Warning: Cannot dlopen("/home/ubuntu/valheim_server/valheim_server_Data/Managed/mono/aot-cache/amd64/UnityEngine.dll.so"/0xa4d19bc0, 101)
Error loading needed lib /home/ubuntu/valheim_server/valheim_server_Data/Managed/UnityEngine.AIModule.dll.so
Warning: Cannot dlopen("/home/ubuntu/valheim_server/valheim_server_Data/Managed/UnityEngine.AIModule.dll.so"/0xac9f17f0, 101)
Error loading needed lib /home/ubuntu/valheim_server/valheim_server_Data/Managed/mono/aot-cache/amd64/UnityEngine.AIModule.dll.so
Warning: Cannot dlopen("/home/ubuntu/valheim_server/valheim_server_Data/Managed/mono/aot-cache/amd64/UnityEngine.AIModule.dll.so"/0xa3c7d8c0, 101)
Error loading needed lib /home/ubuntu/valheim_server/valheim_server_Data/Managed/UnityEngine.AndroidJNIModule.dll.so
Warning: Cannot dlopen("/home/ubuntu/valheim_server/valheim_server_Data/Managed/UnityEngine.AndroidJNIModule.dll.so"/0xac9f17f0, 101)
Error loading needed lib /home/ubuntu/valheim_server/valheim_server_Data/Managed/mono/aot-cache/amd64/UnityEngine.AndroidJNIModule.dll.so
Warning: Cannot dlopen("/home/ubuntu/valheim_server/valheim_server_Data/Managed/mono/aot-cache/amd64/UnityEngine.AndroidJNIModule.dll.so"/0xa3c7d8c0, 101)
...
```
But eventually you'll see:
```
12/23/2021 03:46:36: Loaded 7384 locations

(Filename: ./Runtime/Export/Debug/Debug.bindings.h Line: 39)
```
Once the world is done building, I've had to quit out of the process to save the world for the 1st time.

<kbd>Ctrl</kbd> + <kbd>C</kbd> to terminate and save world

Relaunch the start_server.sh file, there will still be the Unity.dll errors but eventually you'll get a message that all locations have been loaded.
```
./start_server.sh
```
### Launch Valheim and login!!!
Try to login with your public IP address and password. ```I have arrived!!!```


## Setup Valheim Service
It's possible to have Valheim run on startup, instead of launching the ```start_server.sh``` everytime.  We'll use ```nano``` again to create the file.
```
sudo nano /etc/systemd/system/valheim.service
```
Paste the following contents in the file.  

```<IMPORTANT>```Change the parameters to your own server params!```<IMPORTANT>```
```
[Unit]
Description=Valheim Dedicated Server
Wants=network-online.target
After=network-online.target

[Service]
Environment=SteamAppId=892970
Environment=LD_LIBRARY_PATH=/home/ubuntu/valheim_server/linux64:$LD_LIBRARY_PATH
Type=simple
Restart=on-failure
RestartSec=10
KillSignal=SIGINT
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/valheim_server
ExecStart=/home/ubuntu/valheim_server/valheim_server.x86_64 -nographics -batchmode -port 2456 -public 0 -name "My Server Name" -world "MyWorldName" -password "MySecretPassword" -savedir "/home/ubuntu/valheim_data"

[Install]
WantedBy=multi-user.target
```
Enable and start the service.
```
sudo systemctl enable valheim

sudo systemctl start valheim
```
You can check on the service with the following command:
```
sudo systemctl status valheim
```
