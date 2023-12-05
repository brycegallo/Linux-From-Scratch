Using Ubuntu 22.04.3 for Arm64 in Parallels on an M2 Macbook Air

Enter the terminal and use the following commands:
Enter root
```
sudo -i
```

use bash instead of dash
```
dpkg-reconfigure dash
no
```

Add universe and multiverse repositories
```
sudo add-apt-repository universe
sudo add-apt-repository multiverse
sudo apt update
```
use  [2.2-bash-script](https://github.com/brycegallo/Linux-From-Scratch/blob/main/2.2-bash-script) to check software requirements
```
bash /home/parallels/Documents/2.2-bash-script
```
With around 40-50GB of free space on a drive, use GParted to create three partitions as follows:
1. 512mb ext2 - "boot"
2. ~40,000 ext4 - "rootfs"
3. 4,096mb linux-swap - "swap"

Once the partitions are created, open the menu to change flags on the boot partition, set its flag to "boot"
Use ```lsblk``` to check the size of the partitions

Set a variable ```LFS``` with the following command
```
export LFS=/mnt/lfs
```
Check this is set by echoing its value
```
echo $LFS
```

Create and Mount the 
```
mkdir -pv $LFS
mount -v -t ext4 /dev/sda4 $LFS
```
Check that this is mounted properly
```
lsblk
```

Ensure that the swap partition is enabled
```
cd $LFS
/sbin/swapon -v /dev/sda5
```

Prepare a directory all the packages to be stored for this build, then change its permissions so that it can be written to by anyone but only the owner can delete from it
```
mkdir -v $LFS/sources
chmod -v a+wt $LFS/sources
```

To get all the necessary packages for this build we can use wget with the following commands to download them
```
wget https://www.linuxfromscratch.org/~xry111/lfs/view/arm64/wget-list-sysv
wget --input-file=wget-list-sysv --continue --directory-prefix=$LFS/sources
```
To verify that everything has been downloaded, run the following commands
```
wget https://www.linuxfromscratch.org/~xry111/lfs/view/arm64/md5sums

pushd $LFS/sources
  md5sum -c md5sums
popd
```
Any missing packages or patches can be downloaded separately and moved to ```/mnt/lfs/sources```. Some may be present and mistakenly missing from the previous check. A more manual check can be done by going to the sources directory and using ```ls -la``` to view all present packages and patches.

Next, create a limited directory hierarchy with the following commands, ensuring that they are being run as root
```
mkdir -pv $LFS/{etc,var} $LFS/usr/{bin,lib,sbin}

for i in bin lib sbin; do
  ln -sv usr/$i $LFS/$i
done
```
Programs appearing later in this process will be cross compiled using a compiler placed in its own directory, which will be made with the following command
```
mkdir -pv $LFS/tools
```

Create a new non-root user to allow more leniency in case any mistakes are made in the next steps.
```
groupadd lfs
useradd -s /bin/bash -g lfs -m -k /dev/null lfs
passwd lfs
```
