Using Ubuntu 22.04.3 for Arm64 in Parallels on an M2 Macbook Air

Enter the terminal and use the following commands
```
# enter root
sudo -i
# use bash instead of dash
dpkg-reconfigure dash
no

# Add universe and multiverse repositories
sudo add-apt-repository universe
sudo add-apt-repository multiverse
sudo apt update

# use 2.2-bash-script to check software requirements
```
