# 🚀 Secure Dual Boot Guide: Windows 11 & Manjaro (BitLocker + LUKS) with Secure Boot

This guide outlines the process of setting up a secure dual-boot system with **Windows 11** and **Manjaro**, ensuring both operating systems are encrypted (**BitLocker for Windows** and **LUKS for Manjaro**) while maintaining **Secure Boot**. 

By the end of this guide, you'll have a fully secure and encrypted dual-boot system that combines the best of both worlds: Windows 11 for productivity and Manjaro for a customizable, open-source Linux experience. 🎉

---

## ⚠️ Important Notes Before You Begin

- **This process requires a fresh install**, which **will delete everything** on your disk. Please **backup any important data** before proceeding. 💾
- You will need **two USB drives**:
  1. One for the **Windows 11 installation media** (at least 8GB).
  2. One for the **Manjaro installation media** (at least 4GB).
- **Secure Boot** will be enabled at the end of the process, and **GRUB** does not work with this setup. We will use **systemd-boot** instead for better compatibility with Secure Boot.

---

## 1. Prepare Installation Media

1. **Create a bootable Windows 11 USB**  
   - Download the **Windows 11 Media Creation Tool** from Microsoft's official website.
   - Use the tool to create a bootable USB with the Windows 11 installer. 📀

2. **Create a bootable Manjaro USB**  
   - Download **Manjaro ISO** from [Manjaro's official website](https://manjaro.org/download/).
   - Use **Rufus (Windows)** or **dd (Linux/Mac)** to write the ISO to a USB drive. 🖥️

---

## 2. Install Windows 11

1. **Boot from the Windows USB**  
   - Insert your **Windows 11 USB** and reboot your system. Select the USB drive from the BIOS boot menu.

2. **Prepare the installation disk**  
   - When you reach the disk selection screen, press `Shift + F10` to open the command prompt. Then type ````` to launch the disk partition tool.

3. **Create Partitions**  
   In cmd, run the following commands to prepare your disk:

   ```
   diskpart
   list disk
   select disk X  # Replace X with your target disk number
   clean
   convert gpt
   ```

   **Create the EFI System Partition (ESP) and Microsoft Reserved Partition (MSR):**  
   These partitions are required for Windows and Secure Boot to work properly.

   ```
   create partition efi size=500
   format fs=fat32 quick
   assign letter=S
   create partition msr size=128
   ```

4. **Finish Windows installation**  
   - Return to the GUI and proceed with the Windows installation. Follow the on-screen instructions to complete the setup.

---

## 3. Enable BitLocker & Configure TPM with a PIN

To add an extra layer of security, we’ll require a **PIN** for startup rather than relying only on **TPM** (which can be vulnerable). 🔒

### Why TPM Alone is Insecure?
- **Cold boot attacks**: The TPM sends the decryption key unencrypted to the CPU during startup, making it possible for attackers to capture the key.
- **Evil Maid Attacks**: If someone has physical access to your device, they can tamper with the boot process and steal keys.

### Steps to Configure a Startup PIN for BitLocker:

1. **Open Group Policy Editor (gpedit.msc)**  
   - Press `Win + R`, type `gpedit.msc`, and hit Enter.

2. **Navigate to the following path:**
   - `Computer Configuration → Administrative Templates → Windows Components → BitLocker Drive Encryption → Operating System Drives`

3. **Modify the "Require additional authentication at startup" policy**  
   - Double-click **"Require additional authentication at startup."**
   - Set it to **Enabled**.
   - Under "Configure TPM startup," select **"Require startup PIN with TPM."**
   - Click **OK**.

4. **Allow enhanced PINs for BitLocker**  
   - Double-click **"Allow enhanced PINs for startup."**
   - Set it to **Enabled**.

5. **Apply changes and restart your system**  
   - Open **BitLocker settings** (`Control Panel → BitLocker Drive Encryption`).
   - Turn on **BitLocker** and set a secure PIN.

6. **Disable Secure Boot (For Now)**  
   - Before proceeding with Manjaro installation, **disable Secure Boot** in BIOS.

---

## 4. Install Manjaro (LUKS Encryption)

1. **Boot from the Manjaro USB**  
   - Select your USB from the BIOS boot menu.

2. **Start installation and select free space**  
   - When choosing where to install, select the **remaining free space** on the disk.
   - Check the **Encrypt** System option to enable LUKS encryption.

3. **Complete installation**  
   - Finish the setup and reboot into Manjaro.

---

## 5. Switch from GRUB to systemd-boot  
Follow this [guide](https://forum.manjaro.org/t/how-to-convert-to-systemd-boot/128946) on Manjaro Forums.  
   If the link breaks, follow these steps:
1. **Remove GRUB and install systemd-boot**  
   

   ```
   sudo pacman -Rc grub
   sudo mkdir /efi
   efidevice=$(findmnt /boot/efi -no SOURCE)
   sudo umount /boot/efi
   sudo mount ${efidevice} /efi
   sudo bootctl install
   ```

   > **Note:** A security warning may appear. To fix it, edit `/etc/fstab` and add `fmask=0077` to the EFI partition.

2. **Edit loader configuration**  
   - Edit `/efi/loader/loader.conf` and **uncomment** `timeout`.

---

## 6. Install and Configure Kernel Boot Management

1. **Install kernel management tool**  

   ```
   pamac build kernel-install-mkinitcpio
   ```

   - If you get an error (`Failing to read AUR data`), fix it with:

   ```
   pamac update --aur --force-refresh
   ```

   - NOTE: This will update the entire system. But it worked for me.

   Run the build again:

   ```
   pamac build kernel-install-mkinitcpio
   ```

2. **Run the following script to install all kernels:**

   ```
   #!/usr/bin/env bash

   esp=$(bootctl -p)
   machineid=$(cat /etc/machine-id)
   if [[ ${machineid} ]]; then
       mkdir ${esp}/${machineid}
   else
       echo "Failed to get the machine ID"
   fi

   while read -r kernel; do
       kernelversion=$(basename "${kernel%/vmlinuz}")
       echo "Installing kernel ${kernelversion}"
       kernel-install add ${kernelversion} ${kernel}
   done < <(find /usr/lib/modules -maxdepth 2 -type f -name vmlinuz)
   ```

3. **Clean up old boot files**  

   ```
   sudo rm -r /boot/efi /boot/grub /boot/initramfs* /boot/vmlinuz*
   ```

---

## 7. Finalizing LUKS Encryption

1. **Edit `/etc/mkinitcpio.conf`**  
   - Remove `/crypto_keyfile.bin` from `FILES=()`.

2. **Reinstall the kernel**  

   ```
   pacman -Syu linux
   reboot
   ```

3. **Remove the crypto keyfile from LUKS**  

   ```
   sudo cryptsetup luksRemoveKey /dev/sdXY /crypto_keyfile.bin
   ```

   > Replace `/dev/sdXY` with the correct LUKS partition.

4. **Delete the keyfile**  

   ```
   sudo rm /crypto_keyfile.bin
   ```

---

## 8. Enable Secure Boot  
Referencing the following [video](https://www.youtube.com/watch?v=yU-SE7QX6WQ) and [arch wiki]().  
1. **Boot into BIOS and enable Setup Mode**  
   - This allows enrolling new Secure Boot keys.

2. **Install `sbctl` in Manjaro**  

   ```
   pacman -Syu sbctl
   ```

3. **Check Secure Boot status**  

   ```
   sbctl status
   ```
    You should see all red crosses.

4. **Create and enroll Secure Boot keys**  

   ```
   sbctl create-keys
   sbctl enroll-keys -m
   ```

5. **Sign all required files automatically**  
    `sbctl verify` should tell you what needs to be signed. Running the following command will sign everything automatically:
   ```
   sbctl verify | sed 's/✗ /sbctl sign -s /e'
   ```

    Now however run:
    ```
    bootctl install
    ```

    And run **AGAIN**:
    ```
    sbctl verify | sed 's/✗ /sbctl sign -s /e'
    ```

6. **Reboot into BIOS and enable Secure Boot.**

---

## 9. Enjoy Your Secure Dual Boot System! 🎉

You should now have:
- **Windows 11 with BitLocker encryption** 🛡️
- **Manjaro with LUKS encryption** 🔐
- **Secure Boot enabled** 🔒
- **Systemd-boot managing both OS entries** 🚀

Everything is now **secure, encrypted, and dual-bootable** while maintaining security. Enjoy your **new dual-boot system**! 😎

---
