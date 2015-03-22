# Useful sites:
- Trackpoint stuff
  - http://askubuntu.com/a/590676
  - http://askubuntu.com/a/590926
- Backlight / volume / power management stuff:
  - http://www.function.fr/advanced-linux-configuration-for-lenovo-thinkpad-x240/
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
            - set up in /etc/crypttab

# Swap
- /swapfile
  - 8GB, I used dd, but could have used fallocate
  - chmod 0600
  - sudo sysctl -w vm.swappiness=1 for SSD
  - added it to /etc/fstab

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
  - buttons but not middle click scrolling work after
    - while running:
      - sudo modprobe -r psmouse
      - sudo modprobe psmouse proto=imps
    - in kernel options:
      - psmouse.proto=imps
  - middle click scroll after some xorg.conf.d changes
    - this seems to break the touchpad somewhat; need to see if I care
  - source: http://askubuntu.com/a/590676
- Power stuff: tlp
  - http://linrunner.de/en/tlp/docs/tlp-linux-advanced-power-management.html
