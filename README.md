# Secure Boot on Pop!_OS + Windows 11 dual boot

Notes on what I did, why it worked, and why it didn't. Written so I can remember where I am.

**Setup:** Lenovo IdeaPad Gaming 3 15IHU6, Pop!_OS (LUKS encrypted) + Windows 11, systemd-boot as boot menu.
**Goal:** Secure Boot on.

## The problem

Windows boots with Secure Boot because Microsoft signs its bootloader and the firmware trusts Microsoft's keys by default.

Pop!_OS uses systemd-boot and kernelstub (installs kernels and writes boot entries). Nothing signs these files, so with Secure Boot on the firmware refuses to run them and only Windows boots.

With Secure Boot off everything works and I just pick the entry in the systemd-boot menu.

## The fix: sbctl

sbctl creates my own Secure Boot keys, enrolls them in the firmware **next to Microsoft's keys**, and signs the Linux EFI files. That way both OSes are trusted.

### 1. Install sbctl

Not in the Pop repos, so I built it with Go:

```bash
go install github.com/foxboron/sbctl/cmd/sbctl@latest
sudo apt install libpcsclite-dev   # build dependency
sudo cp ~/go/bin/sbctl /usr/local/bin/
```

### 2. Old config error

sbctl kept saying `old configuration detected` because the `secureboot-db` package owns `/usr/share/secureboot`. I moved it out of the way:

```bash
sudo mv /usr/share/secureboot /usr/share/secureboot.bak
```

### 3. Create keys

```bash
sudo sbctl create-keys
sudo sbctl status
```

### 4. Sign the Linux files

Only the Linux ones. Windows files are already signed by Microsoft, and `sbctl verify` showing them as "not signed" is fine.

```bash
sudo sbctl verify
sudo sbctl sign -s /boot/efi/EFI/systemd/systemd-bootx64.efi
sudo sbctl sign -s /boot/efi/EFI/BOOT/BOOTX64.EFI
sudo sbctl sign -s /boot/efi/EFI/Pop_OS-<uuid>/vmlinuz.efi
sudo sbctl sign -s /boot/efi/EFI/Pop_OS-<uuid>/vmlinuz-previous.efi
sudo sbctl sign -s /boot/efi/EFI/Recovery-<id>/vmlinuz.efi
```

### 5. BIOS: Setup Mode

BIOS -> Security -> Secure Boot -> **Reset to Setup Mode** (clears the Platform Key and the old keys).

```bash
sudo sbctl status   # Setup Mode should be Enabled
```

### 6. Enroll keys

`--microsoft` keeps Microsoft's keys so Windows still boots.

```bash
sudo sbctl enroll-keys --microsoft
sudo sbctl list-enrolled-keys
```

### 7. Turn Secure Boot on

BIOS -> Secure Boot -> Enabled. Then:

```bash
sudo sbctl status   # Secure Boot should be Enabled
```

Pop!_OS boots with Secure Boot on.

## NVIDIA problem

`nvidia-smi` fails and `modprobe nvidia` says `Key was rejected by service` (`dmesg`: `Loading of unsigned module is rejected`).

I signed the NVIDIA modules with my sbctl db key using `sign-file`. `modinfo` showed the signer, but it still got rejected.

Why: the kernel did load my db key, but into the **Platform Keyring**, and modules are not checked against that. Modules are checked only against the compiled-in Canonical keys and the secondary trusted keyring. The secondary keyring is fed by the **Machine Keyring (MOK)**, which only gets filled by **shim**. My boot chain is firmware -> systemd-boot -> kernel, with no shim, so MOK is empty.

Useful commands:

```bash
sudo dmesg | grep -i -E "secure ?boot|x\.509|keyring"
modinfo nvidia | grep -E "signer|sig_key"
sudo modprobe nvidia
```

## Next: shim + mokutil (planned)

Add Microsoft-signed shim as a second boot entry (keep the working one), make a separate module-signing key, enroll it with mokutil, sign the NVIDIA modules, and set up DKMS to sign automatically.

```bash
sudo apt install shim-signed mokutil
sudo mokutil --import <module-signing-key>.der
```

## TODO

- [ ] Test Windows boot with Secure Boot on (`msinfo32` -> Secure Boot State: On)
- [ ] Hook to re-sign kernels after updates (`sudo sbctl sign-all`), otherwise Pop won't boot after a kernel update
- [ ] shim + MOK for NVIDIA
- [ ] DKMS auto-signing

## Notes

- If something breaks: BIOS -> Restore Factory Keys, or turn Secure Boot off.
- BitLocker is off on Windows, so no recovery key issues.
- My shell aliases `ls` to something that throws `--icons` errors, use `command ls`.
