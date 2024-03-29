## floppy

Ошибка в логах про fd0

```sh
rmmod floppy
echo "blacklist floppy" | tee /etc/modprobe.d/blacklist-floppy.conf
dpkg-reconfigure initramfs-tools
```

playbook
```yaml
---
- name: remove floppy
  hosts: all
  gather_facts: true
  connection: ssh

  tasks:

  - name: remove floppy
    community.general.modprobe:
      name: floppy
      state: absent
    notify:
      - update initramfs

  - name: copy content to modprobe
    ansible.builtin.copy:
      content: "blacklist floppy"
      dest: /etc/modprobe.d/blacklist-floppy.conf
    notify:
      - reboot
      - update initramfs

  handlers:

    - name: reboot
      reboot:
        msg: "Reboot initiated by Ansible"
        connect_timeout: 5
        reboot_timeout: 600
        pre_reboot_delay: 0
        post_reboot_delay: 30
        test_command: whoami

    - name: update initramfs
      command: update-initramfs -u

```

## timezone

```sh
timedatectl set-timezone Europe/Moscow
```

playbook
```yaml
---
- name: set timezone
  hosts: all
  gather_facts: true
  connection: ssh

  tasks:

  - name: set timezone
    community.general.timezone:
      name: Europe/Moscow
```

## vmware disk uuid

Ошибка в логах виртуальной машины
```text
sda: add missing path
```

fix `/etc/multipath.conf`
```text
defaults {
    user_friendly_names yes
}
blacklist {
    devnode "^(ram|raw|loop|fd|md|dm-|sr|scd|st)[0-9]*"
    devnode "^sd[a-z]?[0-9]*"
}
```
playbook
```yaml
---
- name: fix multipath
  hosts: all
  gather_facts: true
  connection: ssh

  tasks:

  - name: copy content to multipath
    ansible.builtin.copy:
      content: |
        defaults {
            user_friendly_names yes
        }
        blacklist {
            devnode "^(ram|raw|loop|fd|md|dm-|sr|scd|st)[0-9]*"
            devnode "^sd[a-z]?[0-9]*"
        }
      dest: /etc/multipath.conf
    notify:
      - reboot

  handlers:

    - name: reboot
      reboot:
        msg: "Reboot initiated by Ansible"
        connect_timeout: 5
        reboot_timeout: 600
        pre_reboot_delay: 0
        post_reboot_delay: 30
        test_command: whoami
```

## apt update

```yaml
---
- name: update and reboot
  hosts: all
  gather_facts: true
  connection: ssh

  tasks:

    - name: Only run "update_cache=yes" if the last one is more than 3600 seconds ago
      apt:
        update_cache: yes
        cache_valid_time: 3600
      async: 3600
      poll: 30

    - name: Update all packages to their latest version
      apt:
        name: "*"
        state: latest

    - name: reboot
      reboot:
        msg: "Reboot initiated by Ansible"
        connect_timeout: 5
        reboot_timeout: 600
        pre_reboot_delay: 0
        post_reboot_delay: 30
        test_command: whoami
```

## resize disk

Увеличен диск для виртуалки с 16 до 32. Нужно использовать все свободное место

```text
# fdisk  /dev/sda

Welcome to fdisk (util-linux 2.36).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

GPT PMBR size mismatch (33554431 != 67108863) will be corrected by write.
The backup GPT table is not on the end of the device. This problem will be corrected by write.

Command (m for help): w

The partition table has been altered.
Syncing disks.
```
Либо
```sh
echo 'w' | fdisk /dev/sda
```
Далее playbook с увеличением диска на текущей виртуалке с установленным ansible
```yaml
---
- name: resize lvm
  hosts: localhost
  become: true
  gather_facts: true
  connection: local

  tasks:

  - name: Read device information (always use unit when probing)
    community.general.parted:
      device: /dev/sda
      unit: MiB
    register: sda_info

  - name: Extend an existing partition to fill all available space
    community.general.parted:
      device: /dev/sda
      number: "{{ sda_info.partitions | length }}"
      label: gpt
      unit: MiB
      part_end: "100%"
      state: present
      resize: true

  - name: Resize existing pv
    community.general.lvg:
      vg: ubuntu-vg
      pvs: /dev/sda3
      pvresize: yes

  - name: Extend the logical volume to take all remaining space of the PVs and resize the underlying filesystem
    community.general.lvol:
      vg: ubuntu-vg
      lv: ubuntu-lv
      size: 100%PVS
      resizefs: true
```
