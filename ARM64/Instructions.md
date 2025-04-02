Using Ubuntu 22.04.3 for Arm64 in Parallels on an M2 Macbook Air
Last Updated: 2023-December-05

# Preparing the Linux Environment

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

Create and Mount the lfs directory
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

Make lfs the owner of all directories under /mnt/lfs so it has full access to them
```
chown -v lfs $LFS/{usr{,/*},lib,var,etc,bin,sbin,tools}
```

Then either start a new shell as user lfs or use the following command to switch to lfs in the current shell
```
su - lfs
```

Create a new bash profile with the following command
```
cat > ~/.bash_profile << "EOF"
exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
EOF
```

Create a bashrc file to define the settings for new bash shell instances.
MAKEFLAGS can be changed to reflect the number of cores available on the host system, the default here is 4
```
cat > ~/.bashrc << "EOF"
set +h
umask 022
LFS=/mnt/lfs
LC_ALL=POSIX
LFS_TGT=$(uname -m)-lfs-linux-gnu
PATH=/usr/bin
if [ ! -L /bin ]; then PATH=/bin:$PATH; fi
PATH=$LFS/tools/bin:$PATH
CONFIG_SITE=$LFS/usr/share/config.site
export LFS LC_ALL LFS_TGT PATH CONFIG_SITE
export LFS=/mnt/lfs
export MAKEFLAGS='-j4'
EOF
```

Remove any undocumented instantations of .bashrc that might be included with some Linux distributions and potentially cause conflicts when building critical packages
```
[ ! -e /etc/bash.bashrc ] || mv -v /etc/bash.bashrc /etc/bash.bashrc.NOUSE
```

Force the bash shell to read this newly created profile
```
source ~/.bash_profile
```

# Compiling a Cross-Toolchain
## Building Software from Source
Ensure all packages and patches are in the directory ```/mnt/lfs/sources/```
1. Use ```tar``` to extract each one
2. ```cd``` to the newly created directory after extraction
3. Follow the specific instructions for each package.
4. ```cd``` back to the sources directory after building
5. Delete the original, un-extracted file unless instructed otherwise

### GNU Binutils 2.41
Extract the tar file and then work inside the directories as instructed
```
tar -xvf binutils-2.41.tar.xz
cd binutils-2.41
mkdir -v build
cd build
```

Use the following command to configure a binutils makefile for build and installation
```
../configure --prefix=$LFS/tools \
             --with-sysroot=$LFS \
             --target=$LFS_TGT   \
             --disable-nls       \
             --enable-gprofng=no \
             --disable-werror
```
Then build and install with the following command, which is wrapped in ```time { } ``` to provide a baseline estimate of how long each build and install should take. This is not necessary but provides us a "Standard Build Unit" (SBU) of time, which LFS has provided for each build.
```
time { make && make install; }
```
After building, optionally check the output of the make process.
```
make -k check
```
Then to help prevent conflicts in future builds, in cases where the same software will be built more than once, return to the sources folder and remove the directory, confirming the removal of any files if the shell prompts.
```
cd $LFS/sources/
rm -r binutils-2.41
```


### GNU Compiler Collection
```
tar -xvf gcc-13.2.0.tar.xz
cd gcc-13.2.0
```

Unpack and rename GCC's MPFR, GMP, and MPC packages so the GCC build process will use them automatically
```
tar -xf ../mpfr-4.2.1.tar.xz
mv -v mpfr-4.2.1 mpfr
tar -xf ../gmp-6.3.0.tar.xz
mv -v gmp-6.3.0 gmp
tar -xf ../mpc-1.3.1.tar.gz
mv -v mpc-1.3.1 mpc
```

This LFS build is being made on an ARM64 host, so the following command sets the default directory name for 64-bit libraries to ```lib```
```
sed -e '/lp64=/s/lib64/lib/' \
    -i.orig gcc/config/aarch64/t-aarch64-linux
```

Following the same instruction as for binutils regarding building to a specific ```build``` directory
```
mkdir -v build
cd build
```

GCC makefile configuration command
```
../configure                  \
    --target=$LFS_TGT         \
    --prefix=$LFS/tools       \
    --with-glibc-version=2.38 \
    --with-sysroot=$LFS       \
    --with-newlib             \
    --without-headers         \
    --enable-default-pie      \
    --enable-default-ssp      \
    --disable-nls             \
    --disable-shared          \
    --disable-multilib        \
    --disable-threads         \
    --disable-libatomic       \
    --disable-libgomp         \
    --disable-libquadmath     \
    --disable-libssp          \
    --disable-libvtv          \
    --disable-libstdcxx       \
    --enable-languages=c,c++
```

Make and install
```
time { make && make install; }
```

Create a full internal system header which will be needed for later builds
```
cd ..
cat gcc/limitx.h gcc/glimits.h gcc/limity.h > \
  `dirname $($LFS_TGT-gcc -print-libgcc-file-name)`/include/limits.h
```

Again clean up after the make and install
```
rm -r gcc-13.2.0
```

### Linux 6.5.1 kernel
```
tar -xvf linux-6.5.1.tar.xz
cd linux-6.5.1
```
Expose an Application Programming Interface (API) for the system's C library (Glibc in LFS) to use
```
make mrproper
```

Extract the user-visible kernel headers from the source
```
make headers
find usr/include -type f ! -name '*.h' -delete
cp -rv usr/include $LFS/usr
```

Clean up
```
cd ..
rm -r linux-6.5.1
```

### Glibc
Extract the tar file, cd into the directory created in extraction, apply the patch, make a build directory, cd into it
```
tar -xvf glibc-2.38.tar.xz
cd glibc-2.38
patch -Np1 -i ../glibc-2.38-fhs-1.patch
mkdir -v build
cd build
```
Ensure that the ldconfig and sln utilities are installed into /usr/sbin:
```
echo "rootsbindir=/usr/sbin" > configparms
```

Configure the makefile
```
../configure                             \
      --prefix=/usr                      \
      --host=$LFS_TGT                    \
      --build=$(../scripts/config.guess) \
      --enable-kernel=4.14               \
      --with-headers=$LFS/usr/include    \
      libc_cv_slibdir=/usr/lib
```

There are some reports that this package fails when using ```make``` with multiple cores, so either use one core to be safe, or use multiple cores as normal and rerun the file with just one core if it fails
```
time make -j1
```

Install the newly built Glibc to the LFS system.
Ensure that ```$LFS``` is set to ```/mnt/lfs``` or the following command might render your current Linux system unusable
```
make DESTDIR=$LFS install
```

Fix a hard coded path to the executable loader in the ldd script
```
sed '/RTLDLIST=/s@/usr@@g' -i $LFS/usr/bin/ldd
```

Now check to make sure the following commands produce the accompanying output
```
echo 'int main(){}' | $LFS_TGT-gcc -xc -
readelf -l a.out | grep ld-linux
```
Output:
```
[Requesting program interpreter: /lib/ld-linux-aarch64.so.1]
```
Then clean up the test file and sources directory
```
rm -v a.out
cd ../..
rm -r glibc-2.38
```

### Libstdc++ from GCC-13.2.0 
Extract the tar file, cd into the directory created in extraction, make a build directory, cd into it
```
tar -xvf gcc-13.2.0.tar.xz
mkdir -v build
cd build
```

Configure the makefile
```
../libstdc++-v3/configure           \
    --host=$LFS_TGT                 \
    --build=$(../config.guess)      \
    --prefix=/usr                   \
    --disable-multilib              \
    --disable-nls                   \
    --disable-libstdcxx-pch         \
    --with-gxx-include-dir=/tools/$LFS_TGT/include/c++/13.2.0
```

Compile Libstdc++ and time the operation
```
time make
```

Install the library
```
make DESTDIR=$LFS install
```

Remove the libtool archive files because they are harmful for cross-compilation:
```
rm -v $LFS/usr/lib/lib{stdc++,stdc++fs,supc++}.la
```

Clean up
```
cd ../..
rm -r gcc-13.2.0
```

# Cross Compiling Temporary Tools

### m4-1.4.19

Extract the tar file, cd into the directory created in extraction
```
tar -xvf m4-1.4.19.tar.xz
cd m4-1.4.19
```

Configure the makefile
```
./configure --prefix=/usr   \
            --host=$LFS_TGT \
            --build=$(build-aux/config.guess)
```

Compile Libstdc++ and time the operation
```
time make
```

Install the library
```
make DESTDIR=$LFS install
```

Leave directory and clean up
```
cd ..
rm -r m4-1.4.19
```

### Ncurses-6.4
Extract the tar file, cd into the directory created in extraction
```
tar -xvf ncurses-6.4.tar.xz
cd ncurses-6.4
```

First, ensure that gawk is found first during configuration:
```
sed -i s/mawk// configure
```

Build the “tic” program on the build host:
```
mkdir build
pushd build
  ../configure
  make -C include
  make -C progs tic
popd
```

Configure the makefile
```
./configure --prefix=/usr                \
            --host=$LFS_TGT              \
            --build=$(./config.guess)    \
            --mandir=/usr/share/man      \
            --with-manpage-format=normal \
            --with-shared                \
            --without-normal             \
            --with-cxx-shared            \
            --without-debug              \
            --without-ada                \
            --disable-stripping          \
            --enable-widec
```

Compile Ncurses-6.4 and time the operation
```
time make
```

Install the package:
```
make DESTDIR=$LFS TIC_PATH=$(pwd)/build/progs/tic install
echo "INPUT(-lncursesw)" > $LFS/usr/lib/libncurses.so
```

Clean up
```
cd ..
rm -r ncurses-6.4
```

### Bash-5.2.15
Extract the tar file, cd into the directory created in extraction
```
tar -xvf bash-5.2.15.tar.gz
cd bash-5.2.15
```

Configure the makefile
```
./configure --prefix=/usr                      \
            --build=$(sh support/config.guess) \
            --host=$LFS_TGT                    \
            --without-bash-malloc
```

Compile Bash-5.2.15 and time the operation
```
time make
```

Install the package
```
make DESTDIR=$LFS install
```

Make a link for the programs that use sh for a shell:
```
ln -sv bash $LFS/bin/sh
```

Clean up
```
cd ..
rm -r bash-5.2.15
```

### Coreutils-9.4
Extract the tar file, cd into the directory created in extraction
```
tar -xvf coreutils-9.4.tar.xz
cd coreutils-9.4
```

Configure the makefile
```
./configure --prefix=/usr                     \
            --host=$LFS_TGT                   \
            --build=$(build-aux/config.guess) \
            --enable-install-program=hostname \
            --enable-no-install-program=kill,uptime
```

Compile Coreutils-9.4 and time the operation
```
time make
```

Install the package
```
make DESTDIR=$LFS install
```

Move programs to their final expected locations
```
mv -v $LFS/usr/bin/chroot              $LFS/usr/sbin
mkdir -pv $LFS/usr/share/man/man8
mv -v $LFS/usr/share/man/man1/chroot.1 $LFS/usr/share/man/man8/chroot.8
sed -i 's/"1"/"8"/'                    $LFS/usr/share/man/man8/chroot.8
```

Clean up
```
cd ..
rm -r Coreutils-9.4
```

### Diffutils-3.10
Extract the tar file, cd into the directory created in extraction
```
tar -xvf diffutils-3.10.tar.xz
cd diffutils-3.10
```

Configure the makefile
```
./configure --prefix=/usr   \
            --host=$LFS_TGT \
            --build=$(./build-aux/config.guess)
```

Compile Diffutils-3.10 and time the operation
```
time make
```

Install the package
```
make DESTDIR=$LFS install
```

Clean up
```
cd ..
rm -r Diffutils-3.10
```

### File-5.45 
Extract the tar file, cd into the directory created in extraction
```
tar -xvf file-5.45.tar.gz
cd file-5.45
```

Make a temporary copy of the file command
```
mkdir build
pushd build
  ../configure --disable-bzlib      \
               --disable-libseccomp \
               --disable-xzlib      \
               --disable-zlib
  make
popd
```

Configure the makefile
```
./configure --prefix=/usr --host=$LFS_TGT --build=$(./config.guess)
```

Compile File-5.45 and time the operation
```
time make FILE_COMPILE=$(pwd)/build/src/file
```

Install the package
```
make DESTDIR=$LFS install
```

Remove the libtool archive file because it is harmful for cross compilation:
```
rm -v $LFS/usr/lib/libmagic.la
```

Clean up
```
cd ..
rm -r file-5.45
```