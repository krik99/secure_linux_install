# How to install Fedora with Secure Boot, systemd-boot, UKI, LUKS(full disk encryption)
In my opinion, this is the minimal security configuration for portable devices like laptops to ensure the security of your information in case of loss. You can add TPM in this config.
In total, only the **boot** partition (not used in real, only for Fedora installation) and the **efi** partition are not encrypted.

## Encrypt hard drive
- Install GParted or any other tool to create partitions on the hard drive.
- Create EFI partition fat32 about 512MB. Leave not encrypted.
- Create Boot partition ext4 about 1G. Leave not encrypted.
- Create an unspecified partition using the full available size.
- Create LUKS container. Enter the container password when prompted.
  ```
  cryptsetup luksFormat --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 2000 /dev/nvme0n1p3
  ```
- Open LUKS container
  ```
  cryptsetup luksOpen /dev/nvme0n1p3 container
  ```
- Initialize LVM
  ```
  pvcreate /dev/mapper/sda2_crypt
  ```
- Create volume group
  ```
  vgcreate root-vg /dev/mapper/sda2_crypt
  ```
- Create SWAP volume
  ```
  lvcreate -n swap -L 4G root-vg
  mkswap /dev/root-vg/swap
  ``` 
- Create root volume
  ```
  lvcreate -n root -L 20G root-vg
  mkfs.ext4 /dev/root-vg/root
  ```
- Create home volume
  ```
  lvcreate -n home -L 20G root-vg
  mkfs.ext4 /dev/root-vg/home
  ```

## Install Fedora
For now, with Fedora 40, I can't install the distro using Anaconda without a **/boot** partition, which will be deleted after installing systemd-boot. You can install Fedora using Anaconda or manually.

TODO: add manually installation guide.

## Install Systemd-boot
Fedora uses Grub as the default bootloader. Here's how to move from Grub to systemd-boot.
- Creating the folder for EFI mount point
  ```
  sudo mkdir /efi
  ```
- Edit the fstab file and change mounting point from **/boot/efi** to **/efi**
  ```
  sudo vi /etc/fstab
  - UUID=xxxx-xxxx    /boot/efi    vfat    umask=0077,shortname=winnt 0 2
  + UUID=xxxx-xxxx    /efi    vfat    umask=0077,shortname=winnt 0 2
  ```
- Unmount the efi partition and mount it to the new path
  ```
  sudo systemctl daemon-reload
  sudo umount /boot/efi
  sudo umount /boot
  sudo mount /efi
  ```
- Create a folder in the ESP directory with the machine-id in the name
  ```
  sudo mkdir /efi/$(cat /etc/machine-id)
  ```
- Remove GRUB from DNF's protected packages
  ```
  sudo rm /etc/dnf/protected.d/{grub,shim}*
  ```
- Uninstall GRUB related packages
  ```
  sudo dnf remove -y grubby grub2\* memtest86\* && sudo rm -rf /boot/*
  ```
- Install systemd-boot
  ```
  sudo dnf install -y systemd-boot-unsigned sdubby
  ```
- Edit the file /etc/kernel/install.conf
  ```
  BOOT_ROOT=/efi
  layout=bls
  ```
- Remove **/boot** partition if necessary. It is no longer needed.
- Now we can install the bootloader and the kernel entries
  ```
  cat /proc/cmdline | cut -d ' ' -f 2- | sudo tee /etc/kernel/cmdline
  sudo bootctl install
  sudo kernel-install add $(uname -r) /lib/modules/$(uname -r)/vmlinuz
  sudo dnf reinstall kernel-core
  ```
- Reboot and try to boot with new bootloader

## Config secure boot
- Install required packages
  ```
  cd /tmp
  curl -L "https://github.com/Foxboron/sbctl/releases/download/9.0/sbctl-9.0.tar.gz" | tar zxvf -
  cd "sbctl-0.9"
  make
  sudo make install
  ```
- Enter UEFI setup menu by press either of F2/Del/Esc/F10/F11/F12 depending on your firmware.
- Open the Boot/Secure Boot menu and select **Clear Secure Boot Keys** to enter **Setup Mode**.
- Boot to system and check that **Setup Mode** is Enabled
  ```
  sbctl status
  Installed:   ✘ Sbctl is not installed
  Setup Mode:  ✘ Enabled
  Secure Boot: ✘ Disabled
  ```
- Generate the key by running
  ```
  sudo sbctl create-keys
  ```
- Create custom secure boot keys
  ```
  sbctl create-keys
  ```
- Enroll custom secure boot keys
  ```
  sbctl enroll-keys
  ```
- Confirm that setup mode is disabled now
  ```
  sbctl status
  Installed:   ✔ Sbctl is installed
  Owner GUID:  a9fbbdb7-a05f-48d5-b63a-08c5df45ee70
  Setup Mode:  ✔ Disabled
  Secure Boot: ✘ Disabled
  ```
- Sign your bootloader
  ```
  sudo sbctl sign /efi/EFI/systemd/systemd-bootx64.efi
  ```

## Signing UKI image
Generating a Unified Kernel Image (UKI) can be done either with **Dracut** or with systemd's **ukify** tool. The latter does not generate an **initramfs**, this will have to be done separately with e.g. **Dracut**.

- Edit /usr/lib/kernel/install.conf
  ```
  layout=uki
  initrd_generator=dracut
  uki_generator=dracut
  ```
- UKI contain kernel command line. Add to /etc/dracut.conf
  ```
  kernel_cmdline="enter here output from cat /etc/kernel/cmdline"
  ```
- To automatically sign the generated **UKI** for use with **Secure Boot** add path to key in /etc/dracut.conf
  ```
  uefi_secureboot_cert=/usr/share/secureboot/keys/db/db.pem
  uefi_secureboot_key=/usr/share/secureboot/keys/db/db.key
  ufi=yes
  early_microcode=yes
  ```
- Install tool for signe UKI
  ```
  sudo dnf install sbsigntools
  ```
- Generate and sign UKI
  ```
  sudo dracut -fv
  ```
- Check UKI in **/efi/EFI/Linux**

## Enable secure boot
- Enter UEFI setup menu by press either of F2/Del/Esc/F10/F11/F12 depending on your firmware and enable **Secure Boot**
- That's all. Secure Boot doesn't allow you to boot any unsigned binaries.

