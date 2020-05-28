# trustbox-19.03 build process

## [board webpage](https://www.grapeboard.com/#!/downloads)

build image instructions. Based on [Scalys Grapeboard BSP user guide](https://www.grapeboard.com/wp-content/uploads/2019/09/Scalys_Grapeboard-bsp-user-guide_20190902.pdf)

## preparation

### on host machine (ubuntu) [or docker]

    sudo apt-get install curl git gawk wget gcc-aarch64-linux-gnu docker.io \
    device-tree-compiler libncursesw5-dev libncurses5-dev make

### setup username and email for git if not already done

    git config --global user.name <username>
    git config --global user.email <email@address.com>

## prepare for building RFS

### setup docker

    sudo addgroup --system docker
    sudo usermod -aG docker <accountname>
    sudo gpasswd -a <accountname> docker
    sudo service docker restart
    #relog to apply changes

### create folder in your home directory and clone bsp-lsdk repository

    mkdir -p ~/trustbox/ && cd ~/trustbox
    git clone https://github.com/serek4/bsp-lsdk.git
    cd bsp-lsdk && git checkout trustbox-1903.1

### download [flexbuild_lsdk1903.tgz](https://www.nxp.com/design/software/embedded-software/linux-software-and-development-tools/layerscape-software-development-kit:LAYERSCAPE-SDK?&tab=Design_Tools_Tab#nogo), NXP account needed (1903 version is in previous tab), move it to bsp-lsdk directory

    mv flexbuild_lsdk1903.tgz ~/trustbox/bsp-lsdk/flexbuild_lsdk1903.tgz

### run `construct.sh` script. It will unpack the flexbuild_lsdk1903.tgz and apply the Grapeboard support patches

    ./construct.sh


## building RFS

    source setup.env    # (on host machine)
    flex-builder docker
    source setup.env    # (in docker environment)

Modify `configs/ubuntu/packages_trustbox` file to add/remove custom packages


### build RFS

    flex-builder -i mkrfs -m trustbox -r ubuntu:main -B packages_trustbox -a arm64

### build linux kernel

    flex-builder -c linux -m trustbox

### compiling apps

    flex-builder -c apps -m trustbox -a arm64

### merge components into RFS

    flex-builder -i merge-component -m trustbox -a arm64

before next step you can modify files to include changes in compressed image (network settings, root ssh access etc.)

### compress image

    flex-builder -i packrfs -a arm64 -m trustbox

### done.

    exit


you now can unpack compressed image, or copy build/rfs/rootfs_lsdk_19.03_LS_arm64/ folder, to sdcard

## prepare sdcard
### on host system

    sudo fdisk -l
    #replace /dev/<sdX> with path to your sdcard
    sudo parted -a optimal /dev/<sdX> mkpart primary 0% 100%
    sudo mkfs.ext4 -L trustbox /dev/<sdX1>
    sudo mkdir /mnt/sdcard
    sudo mount /dev/<sdX1> /mnt/sdcard

## copy to sdcard

### from files

    cd build/rfs/rootfs_lsdk_19.03_LS_arm64/
    sudo rsync -avx --progress ./ /mnt/sdcard # This will take several minutes depending on your system and sdcard.
    sync

### from comperssed image

    sudo tar xvzf ~/trustbox/bsp-lsdk/build/images/rootfs_lsdk_19.03_LS_arm64.tgz -C /mnt/sdcard

### unmount sdcard and put it in board

    sudo umount /mnt/sdcard

## serial connection if can't ssh yet
115200 8 1 none none<br>
user `root`<br>
pass `root`

### allow root ssh login, uncomment `#PermitRootLogin yes`

    vi /etc/ssh/sshd_config
    reboot

### network config `/etc/netplan/config.yaml`

    network:
      version: 2
      renderer: networkd
      ethernets:
        eth0:
          dhcp4: yes
        eth1:
          dhcp4: yes


## ssh connection
### change root password

    passwd root

### set timezone

    timedatectl
    timedatectl set-timezone Europe/Warsaw


### remove user to free up uid 1000

    deluser --remove-home user
    
### add new user with sudo privileges

    adduser -shell /bin/bash <username>
    usermod -aG sudo <username>

# Updating board

## building firmware components

    source setup.env    # (on host machine)
    flex-builder docker
    source setup.env    # (in docker environment)

### composite ﬁrmware (rcw, pbi, u-boot, EthernetMAC/PHY ﬁrmware and PPAimage)

    flex-builder -i mkfw -m trustbox -b qspi 


### or separately

uboot

    flex-builder -c uboot -m trustbox -b qspi 

PFE ﬁrmware

    flex-builder -c qoriq-engine-pfe-bin -m trustbox 

PPA ﬁrmware

    flex-builder -c ppa-generic -m trustbox

### copy files to root of sdcard (in this case),<br>boot board in u-boot by interrupting system boot on start and<br>run this commands

    run update_mmc_uboot_qspi_nor
    run update_mmc_pfe_qspi_nor
    run update_mmc_ppa_qspi_nor
    restart

### reset env to default and save to permanent memory

    env default -a
    saveenv
    restart
