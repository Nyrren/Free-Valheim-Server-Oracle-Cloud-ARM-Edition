# Free-Valheim-Server-Oracle-Cloud-ARM-Edition
This is a walkthrough of how to get a Valheim server running with Oracle Cloud's new ARM Ampere always free tier.

## References

[Akridge's Guide](https://github.com/akridge/Valheim-Free-Game-Server-Setup-Using-Oracle-Cloud)

[Emmet's Guide for RPI4](https://pimylifeup.com/raspberry-pi-valheim-server/)

## Setup Instance in Oracle Cloud

Ubuntu 20.04 Shape
2 Cores - 12 GB of RAM

Open UDP 2456,2457 in Security List

## Update Ubuntu
```
sudo apt update
sudo apt upgrade -y
```

### Open Ports
```
sudo iptables -I INPUT 6 -m state --state NEW -p UDP --dport 2456 -j ACCEPT
sudo netfilter-persistent save
```

### Install Prereqs
```
sudo apt install libsdl2-dev
sudo apt install libsdl2-2.0-0
sudo apt install git build-essential cmake
```

### Install Box 86
```
git clone https://github.com/ptitSeb/box86

sudo dpkg --add-architecture armhf
sudo apt update
sudo apt install gcc-arm-linux-gnueabihf libc6:armhf libncurses5:armhf libstdc++6:armhf

cd ~/box86
mkdir build
cd build
cmake .. -DRPI4ARM64=1 -DCMAKE_BUILD_TYPE=RelWithDebInfo
make -j$(nproc)
sudo make install
sudo systemctl restart systemd-binfmt
```

### Install Box 64
```
cd ~
git clone https://github.com/ptitSeb/box64.git

cd ~/box64
mkdir build
cd build
cmake .. -DRPI4ARM64=1 -DCMAKE_BUILD_TYPE=RelWithDebInfo
make -j$(nproc)
sudo make install
sudo systemctl restart systemd-binfmt
```

### Reboot


### Install Steam
```
mkdir /home/ubuntu/steamcmd
cd /home/ubuntu/steamcmd
```
```
curl -sqL "https://steamcdn-a.akamaihd.net/client/installer/steamcmd_linux.tar.gz" | tar zxvf -
```
### Verify
```
./steamcmd.sh
quit
```

### Install Valheim Server
```
./steamcmd.sh +@sSteamCmdForcePlatformType linux +force_install_dir /home/ubuntu/valheim_server +login anonymous +app_update 896660 validate +quit
```
```
nano /home/ubuntu/valheim_server/start_server.sh
```

```
./valheim_server.x86_64 -nographics -batchmode -port 2456 -public 0 -name "My Server Name" -world "MyWorldName" -password "MySecretPassword" -savedir "/home/ubuntu/valheim_data"
```

```
cd /home/ubuntu/valheim_server/
./start_server.sh
```

```
--Ctrl+C terminate and save world
```

```
./start_server.sh
```

### Setup Valheim Service

```
sudo nano /etc/systemd/system/valheim.service
```
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
```
sudo systemctl enable valheim

sudo systemctl start valheim

sudo systemctl status valheim
```
