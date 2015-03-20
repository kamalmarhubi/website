# Partitions
- two physical partitions
  - 1 GB FAT32 -> /esp
    - /esp/EFI/ubuntu -> /boot (bind mount)
  - LVM physical volume
    - two logical volumes
      - lvol0
        - crypt -> /
        - key is passphrase
      - lvol1
        - crypthome -> /home
        - two keys
          - passphrase
          - /root/keyfile (root:root 0400!!!)

# Boot
- Using gummiboot as default EFI boot manager
  - Installed at
    - /esp/EFI/Boot/bootx64.efi
    - /esp/EFI/gummiboot/gummibootx64.efi
  - Config at /esp/loader/loader.conf
    - contents: "default ubuntu"
  - entries at /esp/loader/entries/<name>.conf
    - set boot 
- Ubuntu kernels at /esp/EFI/ubuntu/

# Hardware
- Backlight keys do not work
  - attempts to set stuff in some xconf place did not help
  - attempts to set acpi_backlight=vendor boot option did not help
  - install xbacklight and use sxhkd
- trackpoint buttons do not work
  - compiled a kernel, but will not use it
  - about to attempt some STUFF
