Using Ubuntu 22.04.3 for Arm64 in Parallels on an M2 Macbook Air

Enter the terminal and use the following commands:
Enter root
```
sudo -i
```

use bash instead of dash
```
dpkg-reconfigure das
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

```
mkdir -pv $LFS
```
