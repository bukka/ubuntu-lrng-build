# Ubuntu LRNG build

This is a build for Ubuntu with kernel patched with the [Linux Random Number Generator](https://github.com/smuellerDD/lrng).

## Noble build

To build noble, following steps should be done:

```sh
git clone https://github.com/bukka/ubuntu-lrng-build

# Add noble sources
sudo cp ubuntu-lrng-build/noble/etc/apt/sources.list.d/ubuntu-noble.sources /etc/apt/sources.list.d/ubuntu-noble.sources

sudo apt update
sudo apt upgrade -y

sudo apt build-dep -y linux linux-image-unsigned-$(uname -r)

# currently linux-cloud-tools-common install Hyper-V KVP Protocol Daemon which delays build due to the bug
# - https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1820063
sudo apt install -y linux-cloud-tools-common wireless-regdb libncurses-dev gawk flex bison openssl libssl-dev dkms libelf-dev libudev-dev libpci-dev libiberty-dev autoconf llvm fakeroot

# fetch lrng patches
git clone https://github.com/smuellerDD/lrng.git
# move cover latter to prevent failure
mv lrng/kernel_patches/v6.8/v54-0000-cover-letter.patch lrng/kernel_patches/v6.8/XXXXv54-0000-cover-letter.patch

# clone latest linux-aws source
git clone --depth 1 --branch master-next git://git.launchpad.net/~canonical-kernel/ubuntu/+source/linux-aws/+git/noble linux-aws/noble
cd linux-aws/noble

# apply lrng patches
git am -3 ../../lrng/kernel_patches/v6.8/v54-00*
# set lrng config annotation for enabling LRNG
cat ../../ubuntu-lrng-build/noble/debian.aws/config/lrng-annotations >> debian.aws/config/annotations

# replace version for uname
sed -i '0,/)\s/s/)\s/+lrng1) /' debian.aws/changelog
# remove arm64 build
sed -i '/^# ARCH: /s/amd64 arm64/amd64/; /^# FLAVOUR: /s/amd64-aws arm64-aws/amd64-aws/' debian.aws/config/annotations

# do the actual debain kerner build
fakeroot debian/rules clean
# TODO: figure out why it fails on the first run and try to fix it 
fakeroot debian/rules defaultconfigs || true
# update annotations - remove arm64 and import them
fakeroot debian/rules defaultconfigs
fakeroot debian/rules binary

# install the kernel packages
cd ..
sudo dpkg -i linux*6.8.0*lrng1*.deb

# grub defaults to enable lrng and AWS serial console
sudo cp  ../ubuntu-lrng-build/noble/etc/default/grub.d/50-cloudimg-settings.cfg /etc/default/grub.d/50-cloudimg-settings.cfg
# set default grub menu
# TODO: make it always work - this is not exactly reliable as the order depends on the kernel version
# TODO: it should maybe grep it based on the kernel deb version
sudo sed -i "s/GRUB_DEFAULT=0/GRUB_DEFAULT=\"1>0\"/" /etc/default/grub
sudo update-grub

# TODO: add clean up to minimize AMI size
```
