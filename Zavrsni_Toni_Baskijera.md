---
autor: Toni Baskijera
---

# Završni projekt iz kolegija Upravljanje računalnim sustavima

Završni projekt opisuje postavljanje usluge [Gitlab CE](https://about.gitlab.com/) na operacijskom sustavu [Arch Linux](https://archlinux.org/). Sastoji se od ukupno četiri različita dijela. Prvi dio sadrži postupak postavljanja i konfiguracije operacijskog sustava Arch Linux uz pomoć sustava [cloud-init](https://cloudinit.readthedocs.io/en/latest/). Drugi dio opisuje instalaciju i konfiguraciju datotečnog sustava [ZFS](https://wiki.archlinux.org/title/ZFS) te postavljanje automatizirane periodičke sigurnosne kopije. Treći dio vezan je uz korištenje alata [nftables](https://wiki.archlinux.org/title/nftables) za postavljanje vatrozida po potrebi. U posljednjem dijelu opisati će se korištenje alata [Ansible](https://www.ansible.com/overview/how-ansible-works) za automatizirano postavljanje web aplikacije s spomenutom uslugom, odnosno konfiguraciju potrebnih komponenti web aplikacije.

Ekstenzije koje su nam potrebne za Visual Studio Code su sljedeće:

- Ansible (podrška za Ansible jezik)
- markdownlint (potiče pisanje Markdown datoteka u konzistentno, u skladu sa standardima)
- Remote-SSH - (otvaranje i uređivanje datoteka koje se nalaze na udaljenoj mašini na lokalnoj mašini)

## Konfiguracija i virtualizacija operacijskog sustava

Za početak, potrebno je preuzeti [odgovarajuću sliku Arch Linuxa](https://gitlab.archlinux.org/archlinux/arch-boxes/-/jobs/66557/artifacts/file/output/Arch-Linux-x86_64-cloudimg-20220706.66557.qcow2) koja je namijenjena za korištenje u oblaku te sadrži sustav `cloud-init`. Poslužiti će nam za osnovnu konfiguraciju operacijskog sustava. Format datoteke je `qcow2` kojeg koristi alat [Qemu](https://www.qemu.org/), alat za potpunu virtualizaciju.

Zatim preuzetu datoteku kopiramo u korijenski direktorij. Za sljedeće korake važno je koristiti kopiju datoteke kako ne bismo ponovo trebali preuzimati sliku ukoliko je na bilo koji način učinimo neupotrebljivom. Kopiju ćemo preimenovati te joj postaviti naziv `arch-urs`.

U korijenskom direktoriju stvorimo datoteku `user-data` sa sljedećim sadržajem:

```yaml
#cloud-config
users:
  - default
system_info:
   default_user:
     name: tonib
     lock_passwd: False
     plain_text_passwd: '12345'
     gecos: arch Cloud User
     groups: [wheel, adm]
     sudo: ["ALL=(ALL) NOPASSWD:ALL"]
     shell: /bin/bash
```

Nakon toga stvorimo i praznu datoteku `meta-data` koja može ostati i prazna, bitno je da postoji.

Potom možemo iskoristiti sljedeću naredbu za stvaranje datoteke u ISO 9660:

```bash
toni@toni-WRT-WX9:~$ sudo xorriso -as genisoimage -output cloud-init.iso -volid CIDATA -joliet -rock user-data meta-data
xorriso 1.5.2 : RockRidge filesystem manipulator, libburnia project.

Drive current: -outdev 'stdio:cloud-init.iso'
Media current: stdio file, overwriteable
Media status : is blank
Media summary: 0 sessions, 0 data blocks, 0 data, 3241m free
Added to ISO image: file '/user-data'='/home/toni/user-data'
xorriso : UPDATE :       1 files added in 1 seconds
Added to ISO image: file '/meta-data'='/home/toni/meta-data'
xorriso : UPDATE :       2 files added in 1 seconds
ISO image produced: 184 sectors
Written to medium : 184 sectors at LBA 0
Writing to 'stdio:cloud-init.iso' completed successfully.
```

Slici je zatim potrebno dodjeliti više memorije, kako nam zadanih 2GB nije dovoljno:

```bash
toni@toni-WRT-WX9:~$ qemu-img resize arch-urs.qcow2 +10G
Image resized.
```

Tada možemo pokrenuti [Virtual Machine Manager](https://virt-manager.org/) te započnemo s procesom stvaranja nove virtualne mašine. Redom biramo *Import existing disk image ->, Browse local image* gdje izaberemo našu datoteku u `qcow2` formatu.  Na dnu prozora izaberemo ArchLinux kao sustav kojeg instaliramo na novu mašinu. Sljedeći prozor možemo preskočiti, a na zadnjem možemo označiti kvadratić *Customize configuration before install*.

Zatim biramo *Add hardware* te dodamo uređaj za pohranu tipa cd-rom s prilagođenom slikom koju smo malo prije stvorili, `cloud init.iso`. Zatim započnemo instalaciju.

S završetkom instalacije logiramo se u operacijski sustav, te provjerimo koja nam je ip adresa naredbom `ip addr` kako bismo se onda mogli "sshati" u nju naredbom:

```bash
toni@toni-WRT-WX9:~$ ssh tonib@192.168.122.107
tonib@192.168.122.107's password: 
Last login: Wed Jul  6 19:56:15 2022 from 192.168.122.1
[tonib@archlinux ~]$ 
```

Dobra je navika ažurirati sve pakete na operacijskom sustavu te sinkronizirati baze repozitorija, što učinimo uz pomoć Archovog upravitelja paketima `pacman`.

```bash
[tonib@archlinux ~]$ sudo pacman -Syu
:: Synchronizing package databases...
 core                                                              156.6 KiB   108 KiB/s 00:01 [########################################################] 100%
 extra                                                            1720.8 KiB   205 KiB/s 00:08 [########################################################] 100%
 community                                                           6.7 MiB   302 KiB/s 00:23 [########################################################] 100%
:: Starting full system upgrade...
 there is nothing to do
[tonib@archlinux ~]$ 
```

## Instalacija i konfiguracija datotečnog sustava ZFS

Cilj nam je instalacija DKMS verzije paketa ZFS. DKMS, odnosno Dynamic Kernel Module Support omogućava automatsko ponovno buildanje paketa ZFS-a sa svakim ažuriranjem (nadogradnjom) kernela. Na taj način paket ZFS neće biti vezan isključivo uz jednu verziju kernela.

Prije same instalacije navedenog paketa, potrebno je i zadovoljiti neke zavisnosti koje možemo preuzeti uz pomoć pacmana:

```bash
[tonib@archlinux ~]$ sudo pacman -S base-devel linux-headers kmod linux
```

Za instalaciju datotečnog sustava ZFS, prije svega je potrebno preuzeti datoteke [zfs.initcpio.install](https://aur.archlinux.org/cgit/aur.git/plain/zfs.initcpio.install?h=zfs-utils) i [zfs.initcpio.hook](https://aur.archlinux.org/cgit/aur.git/plain/zfs.initcpio.hook?h=zfs-utils):

```bash
[tonib@archlinux ~]$ curl https://aur.archlinux.org/cgit/aur.git/plain/zfs.initcpio.install?h=zfs-utils --output zfs.initcpio.install
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2301  100  2301    0     0   7195      0 --:--:-- --:--:-- --:--:--  7213
[tonib@archlinux ~]$ curl https://aur.archlinux.org/cgit/aur.git/plain/zfs.initcpio.hook?h=zfs-utils --output zfs.initcpio.hook
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  3620  100  3620    0     0  11216      0 --:--:-- --:--:-- --:--:-- 11242


Sljedeći korak je instalacija [zfs-utils](https://aur.archlinux.org/packages/zfs-utils) koji se nalazi na Arch User Repositoryu.

Posjetimo stranicu određenog paketa, u kutu kliknemo na *View PKGBUILD*, a zatim *plain*. Tada kopiramo sadržaj linka na kojem se nalazimo te ga spremimo u datoteku pod nazivom PKGBUILD:

```bash
[tonib@archlinux ~]$ curl https://aur.archlinux.org/cgit/aur.git/plain/PKGBUILD?h=zfs-utils -o PKGBUILD
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2858  100  2858    0     0  11576      0 --:--:-- --:--:-- --:--:-- 11570
```

Naredbom `makepkg` čitamo sadržaj datoteke PKGBUILD i na temelju iste izgrađujemo paket:

```bash
[tonib@archlinux ~]$ makepkg
==> Making package: zfs-utils 2.1.5-1 (Wed 06 Jul 2022 10:14:23 PM UTC)
==> Checking runtime dependencies...
==> Checking buildtime dependencies...
==> Retrieving sources...
  -> Found zfs-2.1.5.tar.gz
  -> Found zfs-2.1.5.tar.gz.asc
  -> Found zfs.initcpio.install
  -> Found zfs.initcpio.hook
==> Validating source files with sha256sums...
    zfs-2.1.5.tar.gz ... Passed
    zfs-2.1.5.tar.gz.asc ... Skipped
    zfs.initcpio.install ... Passed
    zfs.initcpio.hook ... Passed
==> Validating source files with b2sums...
    zfs-2.1.5.tar.gz ... Passed
    zfs-2.1.5.tar.gz.asc ... Skipped
    zfs.initcpio.install ... Passed
    zfs.initcpio.hook ... Passed
==> Verifying source file signatures with gpg...
    zfs-2.1.5.tar.gz ... Passed
==> Extracting sources...
  -> Extracting zfs-2.1.5.tar.gz with bsdtar
==> Starting prepare()...
configure.ac: warning: AM_GNU_GETTEXT is used, but not AM_GNU_GETTEXT_VERSION or AM_GNU_GETTEXT_REQUIRE_VERSION
libtoolize: putting auxiliary files in AC_CONFIG_AUX_DIR, 'config'.
libtoolize: copying file 'config/ltmain.sh'
libtoolize: putting macros in AC_CONFIG_MACRO_DIRS, 'config'.
libtoolize: copying file 'config/libtool.m4'
libtoolize: copying file 'config/ltoptions.m4'
libtoolize: copying file 'config/ltsugar.m4'
...
==> Tidying install...
  -> Removing libtool files...
  -> Purging unwanted files...
  -> Removing static library files...
  -> Stripping unneeded symbols from binaries and libraries...
  -> Compressing man and info pages...
==> Checking for packaging issues...
==> Creating package "zfs-utils"...
  -> Generating .PKGINFO file...
  -> Generating .BUILDINFO file...
  -> Generating .MTREE file...
  -> Compressing package...
==> Leaving fakeroot environment.
==> Finished making: zfs-utils 2.1.5-1 (Wed 06 Jul 2022 10:24:47 PM UTC)

```

Iz izlaza prošle naredbe možemo vidjeti da je izgrađen paket `zfs-utils 2.1.5-1` kojeg sada možemo i instalirati:

```bash
[tonib@archlinux ~]$ makepkg --install
==> WARNING: A package has already been built, installing existing package...
==> Installing package zfs-utils with pacman -U...
loading packages...
resolving dependencies...
looking for conflicting packages...

Packages (1) zfs-utils-2.1.5-1

Total Installed Size:  8.88 MiB

:: Proceed with installation? [Y/n] y
(1/1) checking keys in keyring                                                                 [########################################################] 100%
(1/1) checking package integrity                                                               [########################################################] 100%
(1/1) loading package files                                                                    [########################################################] 100%
(1/1) checking for file conflicts                                                              [########################################################] 100%
(1/1) checking available disk space                                                            [########################################################] 100%
:: Processing package changes...
(1/1) installing zfs-utils                                                                     [########################################################] 100%
Optional dependencies for zfs-utils
    python: for arcstat/arc_summary/dbufstat [installed]
:: Running post-transaction hooks...
(1/4) Reloading system manager configuration...
(2/4) Reloading device manager configuration...
(3/4) Arming ConditionNeedsUpdate...
(4/4) Updating linux initcpios...
==> Building image from preset: /etc/mkinitcpio.d/linux.preset: 'default'
  -> -k /boot/vmlinuz-linux -c /etc/mkinitcpio.conf -g /boot/initramfs-linux.img
==> Starting build: 5.18.9-arch1-1
  -> Running build hook: [base]
  -> Running build hook: [udev]
  -> Running build hook: [autodetect]
  -> Running build hook: [modconf]
  -> Running build hook: [block]
==> WARNING: Possibly missing firmware for module: xhci_pci
  -> Running build hook: [filesystems]
  -> Running build hook: [keyboard]
  -> Running build hook: [fsck]
==> Generating module dependencies
==> Creating zstd-compressed initcpio image: /boot/initramfs-linux.img
==> Image generation successful
==> Building image from preset: /etc/mkinitcpio.d/linux.preset: 'fallback'
  -> -k /boot/vmlinuz-linux -c /etc/mkinitcpio.conf -g /boot/initramfs-linux-fallback.img -S autodetect
==> Starting build: 5.18.9-arch1-1
  -> Running build hook: [base]
  -> Running build hook: [udev]
  -> Running build hook: [modconf]
  -> Running build hook: [block]
==> WARNING: Possibly missing firmware for module: aic94xx
==> WARNING: Possibly missing firmware for module: bfa
==> WARNING: Possibly missing firmware for module: cxgb4
==> WARNING: Possibly missing firmware for module: csiostor
==> WARNING: Possibly missing firmware for module: cxgb3
==> WARNING: Possibly missing firmware for module: isci
==> WARNING: Possibly missing firmware for module: qed
==> WARNING: Possibly missing firmware for module: qla2xxx
==> WARNING: Possibly missing firmware for module: advansys
==> WARNING: Possibly missing firmware for module: qla1280
==> WARNING: Possibly missing firmware for module: wd719x
==> WARNING: Possibly missing firmware for module: xhci_pci
==> WARNING: Possibly missing firmware for module: ums_eneub6250
  -> Running build hook: [filesystems]
  -> Running build hook: [keyboard]
  -> Running build hook: [fsck]
==> Generating module dependencies
==> Creating zstd-compressed initcpio image: /boot/initramfs-linux-fallback.img
==> Image generation successful

```

Prije instalacije paketa `zfs-dkms`, preuzeti ćemo potrebnu [zakrpu](https://aur.archlinux.org/cgit/aur.git/plain/0001-only-build-the-module-in-dkms.conf.patch?h=zfs-dkms) koju spremimo pod postojećim imenom:

```bash
[tonib@archlinux ~]$ curl https://aur.archlinux.org/cgit/aur.git/plain/0001-only-build-the-module-in-dkms.conf.patch?h=zfs-dkms -o 0001-only-build-the-module-in-dkms.conf.patch
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1112  100  1112    0     0   4707      0 --:--:-- --:--:-- --:--:--  4691


```

Sada pokušajmo izgraditi i paket [zfs-dkms](https://aur.archlinux.org/packages/zfs-dkms):

```bash
[tonib@archlinux ~]$ curl https://aur.archlinux.org/cgit/aur.git/plain/PKGBUILD?h=zfs-dkms --output PKGBUILD
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2405  100  2405    0     0  11183      0 --:--:-- --:--:-- --:--:-- 11186
[tonib@archlinux ~]$ makepkg
==> Making package: zfs-dkms 2.1.5-1 (Wed 06 Jul 2022 09:58:33 PM UTC)
==> Checking runtime dependencies...
==> Checking buildtime dependencies...
==> Retrieving sources...
  -> Found zfs-2.1.5.tar.gz
  -> Found zfs-2.1.5.tar.gz.asc
  -> Found 0001-only-build-the-module-in-dkms.conf.patch
==> Validating source files with sha256sums...
    zfs-2.1.5.tar.gz ... Passed
    zfs-2.1.5.tar.gz.asc ... Skipped
    0001-only-build-the-module-in-dkms.conf.patch ... Passed
==> Validating source files with b2sums...
    zfs-2.1.5.tar.gz ... Passed
    zfs-2.1.5.tar.gz.asc ... Skipped
    0001-only-build-the-module-in-dkms.conf.patch ... Passed
==> Verifying source file signatures with gpg...
    zfs-2.1.5.tar.gz ... FAILED (unknown public key 6AD860EED4598027)
==> ERROR: One or more PGP signatures could not be verified!
```

Novu grešku rješavamo tako što ćemo ovjeriti ključeve datoteke izvornog koda:

```bash
[tonib@archlinux ~]$ sudo gpg --receive-keys 6AD860EED4598027
gpg: /home/tonib/.gnupg/trustdb.gpg: trustdb created
gpg: key 6AD860EED4598027: public key "Tony Hutter (GPG key for signing ZFS releases) <hutter2@llnl.gov>" imported
gpg: Total number processed: 1
gpg:               imported: 1

```

Gradnja paketa `zfs-dkms` se sada uspješno izvršava:

```bash
[tonib@archlinux ~]$ makepkg
==> Making package: zfs-dkms 2.1.5-1 (Wed 06 Jul 2022 09:59:57 PM UTC)
==> Checking runtime dependencies...
==> Checking buildtime dependencies...
==> Retrieving sources...
  -> Found zfs-2.1.5.tar.gz
  -> Found zfs-2.1.5.tar.gz.asc
  -> Found 0001-only-build-the-module-in-dkms.conf.patch
==> Validating source files with sha256sums...
    zfs-2.1.5.tar.gz ... Passed
    zfs-2.1.5.tar.gz.asc ... Skipped
    0001-only-build-the-module-in-dkms.conf.patch ... Passed
==> Validating source files with b2sums...
    zfs-2.1.5.tar.gz ... Passed
    zfs-2.1.5.tar.gz.asc ... Skipped
    0001-only-build-the-module-in-dkms.conf.patch ... Passed
==> Verifying source file signatures with gpg...
    zfs-2.1.5.tar.gz ... Passed
==> Extracting sources...
  -> Extracting zfs-2.1.5.tar.gz with bsdtar
==> Starting prepare()...
patching file scripts/dkms.mkconf
configure.ac: warning: AM_GNU_GETTEXT is used, but not AM_GNU_GETTEXT_VERSION or AM_GNU_GETTEXT_REQUIRE_VERSION
libtoolize: putting auxiliary files in AC_CONFIG_AUX_DIR, 'config'.
libtoolize: copying file 'config/ltmain.sh'
libtoolize: putting macros in AC_CONFIG_MACRO_DIRS, 'config'.
libtoolize: copying file 'config/libtool.m4'
libtoolize: copying file 'config/ltoptions.m4'
libtoolize: copying file 'config/ltsugar.m4'
libtoolize: copying file 'config/ltversion.m4'
libtoolize: copying file 'config/lt~obsolete.m4'
configure.ac:48: installing 'config/compile'
configure.ac:42: installing 'config/missing'
==> Starting build()...
==> Entering fakeroot environment...
==> Starting package()...
==> Tidying install...
  -> Removing libtool files...
  -> Purging unwanted files...
  -> Removing static library files...
  -> Stripping unneeded symbols from binaries and libraries...
  -> Compressing man and info pages...
==> Checking for packaging issues...
==> Creating package "zfs-dkms"...
  -> Generating .PKGINFO file...
  -> Generating .BUILDINFO file...
  -> Generating .MTREE file...
  -> Compressing package...
==> Leaving fakeroot environment.
==> Finished making: zfs-dkms 2.1.5-1 (Wed 06 Jul 2022 10:00:17 PM UTC)

```

Paket sada i instaliramo:

```bash
[tonib@archlinux ~]$ makepkg --install
==> WARNING: A package has already been built, installing existing package...
==> Installing package zfs-dkms with pacman -U...
loading packages...
warning: zfs-dkms-2.1.5-1 is up to date -- reinstalling
resolving dependencies...
looking for conflicting packages...

Packages (1) zfs-dkms-2.1.5-1

Total Installed Size:  17.89 MiB
Net Upgrade Size:       0.00 MiB

:: Proceed with installation? [Y/n] y
(1/1) checking keys in keyring                                                                 [########################################################] 100%
(1/1) checking package integrity                                                               [########################################################] 100%
(1/1) loading package files                                                                    [########################################################] 100%
(1/1) checking for file conflicts                                                              [########################################################] 100%
(1/1) checking available disk space                                                            [########################################################] 100%
:: Running pre-transaction hooks...
(1/1) Remove upgraded DKMS modules
==> dkms remove zfs/2.1.5
:: Processing package changes...
(1/1) reinstalling zfs-dkms                                                                    [########################################################] 100%
:: Running post-transaction hooks...
(1/2) Arming ConditionNeedsUpdate...
(2/2) Install DKMS modules
==> dkms install --no-depmod zfs/2.1.5 -k 5.18.9-arch1-1
/usr/bin/dkms: line 1033: sha512: command not found
/usr/bin/dkms: line 1033: sha512: command not found
/usr/bin/dkms: line 1033: sha512: command not found
/usr/bin/dkms: line 1033: sha512: command not found
/usr/bin/dkms: line 1033: sha512: command not found
/usr/bin/dkms: line 1033: sha512: command not found
/usr/bin/dkms: line 1033: sha512: command not found
/usr/bin/dkms: line 1033: sha512: command not found
/usr/bin/dkms: line 1033: sha512: command not found
==> depmod 5.18.9-arch1-1

```

Naredbom `zpool status` provjeravamo je li instalacija uspješna, no prije je potrebno loadati ZFS-ove module:

```bash
[tonib@archlinux ~]$ zpool create
The ZFS modules are not loaded.
Try running '/sbin/modprobe zfs' as root to load them.
[tonib@archlinux ~]$ sudo /sbin/modprobe zfs
[tonib@archlinux ~]$ zpool status
no pools available
```

Uz pomoć Virtual Machine Mangera našoj ćemo virtualnoj mašini dodati još četiri virtualna diska za pohranu od kojih svaki ima veličinu 2GB.

Diskove koje smo dodali potrebno je particionirati na način da sadrže samo jednu particiju tipa `Solaris /usr & Apple ZFS`:

```bash
[tonib@archlinux ~]$ sudo fdisk /dev/vdb

Welcome to fdisk (util-linux 2.38).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0xc6815ca1.

Command (m for help): g
Created a new GPT disklabel (GUID: CA3ACA7C-8BDB-EE48-9D75-F52BCEE4AEFD).

Command (m for help): n
Partition number (1-128, default 1): 
First sector (2048-4194270, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-4194270, default 4192255): 

Created a new partition 1 of type 'Linux filesystem' and of size 2 GiB.

Command (m for help): t
Selected partition 1
Partition type or alias (type L to list all): 157
Changed type of partition 'Linux filesystem' to 'Solaris /usr & Apple ZFS'.

Command (m for help): p
Disk /dev/vdb: 2 GiB, 2147483648 bytes, 4194304 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: CA3ACA7C-8BDB-EE48-9D75-F52BCEE4AEFD

Device     Start     End Sectors Size Type
/dev/vdb1   2048 4192255 4190208   2G Solaris /usr & Apple ZFS

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

```

Nakon što isti postupak ponovimo za diskove vdc, vdd i vde, uvjerimo se da svi imaju particijsku tablicu tipa GPT i na njoj samo jednu particiju tipa Solaris /usr & Apple ZFS:

```bash
[tonib@archlinux ~]$ sudo fdisk -l
Disk /dev/vda: 12 GiB, 12884901888 bytes, 25165824 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: D62C9A64-C50A-4163-887A-EC784559DC75

Device     Start      End  Sectors Size Type
/dev/vda1   2048     4095     2048   1M BIOS boot
/dev/vda2   4096 25165790 25161695  12G Linux filesystem


Disk /dev/vdb: 2 GiB, 2147483648 bytes, 4194304 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: CA3ACA7C-8BDB-EE48-9D75-F52BCEE4AEFD

Device     Start     End Sectors Size Type
/dev/vdb1   2048 4192255 4190208   2G Solaris /usr & Apple ZFS


Disk /dev/vdc: 2 GiB, 2147483648 bytes, 4194304 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 7D648371-7BD3-3540-9A26-884C1F77BF90

Device     Start     End Sectors Size Type
/dev/vdc1   2048 4192255 4190208   2G Solaris /usr & Apple ZFS


Disk /dev/vdd: 2 GiB, 2147483648 bytes, 4194304 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 721D9F4C-7D24-894C-8D57-8713747F78F4

Device     Start     End Sectors Size Type
/dev/vdd1   2048 4192255 4190208   2G Solaris /usr & Apple ZFS


Disk /dev/vde: 2 GiB, 2147483648 bytes, 4194304 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 167B793E-2E7B-E24E-959E-AC60181AD8F4

Device     Start     End Sectors Size Type
/dev/vde1   2048 4192255 4190208   2G Solaris /usr & Apple ZFS

```

Diskove vdb i vdc, odnosno njihove pojedinačne particije dodati ćemo u novi bazen pod nazivom Gitlab_data te iskoristiti argument `mirror`, što omogućuje da se na oba diska zapisuju isti podaci. Isto proces ponoviti ćemo za diskove vdd i vde, samo što ćemo bazen nazvati Gitlab_backup.

```bash
[tonib@archlinux dev]$ sudo zpool create Gitlab_data mirror /dev/vdb1 /dev/vdc1
[tonib@archlinux dev]$ sudo zpool create Gitlab_backup mirror /dev/vdd1 /dev/vde1
```

Nismo naveli argument `-m` pa je mount point bazena /Gitlab_data i /Gitlab_backup, respektivno.

Stanje bazena je u ovome trenutku sljedeće:

```bash
[tonib@archlinux dev]$ zpool status
  pool: Gitlab_backup
 state: ONLINE
config:

 NAME           STATE     READ WRITE CKSUM
 Gitlab_backup  ONLINE       0     0     0
   mirror-0     ONLINE       0     0     0
     vdd1       ONLINE       0     0     0
     vde1       ONLINE       0     0     0

errors: No known data errors

  pool: Gitlab_data
 state: ONLINE
config:

 NAME         STATE     READ WRITE CKSUM
 Gitlab_data  ONLINE       0     0     0
   mirror-0   ONLINE       0     0     0
     vdb1     ONLINE       0     0     0
     vdc1     ONLINE       0     0     0

errors: No known data errors

```

Za kraj je bazene potrebno dodati u `zpool.cache`, čime omogućujemo servisu `zfs-import-cache.service` da automatski importa bazene na svakom bootu:

```bash
[tonib@archlinux ~]$ sudo zpool set cachefile=/etc/zfs/zpool.cache Gitlab_data
[tonib@archlinux ~]$ sudo zpool set cachefile=/etc/zfs/zpool.cache Gitlab_backup
```

Naravno, zbog toga servis `zfs-import-cache.service` mora biti omogućen, kao i `zfs-import.target`:

```bash
[tonib@archlinux ~]$ sudo systemctl enable zfs-import-cache.service
Created symlink /etc/systemd/system/zfs-import.target.wants/zfs-import-cache.service → /usr/lib/systemd/system/zfs-import-cache.service.
[tonib@archlinux ~]$ sudo systemctl enable zfs-import.target
Created symlink /etc/systemd/system/zfs.target.wants/zfs-import.target → /usr/lib/systemd/system/zfs-import.target.
```

Za automatsko montiranje ZFS datotečnog sustava pri svakom bootu potrebno je omogućiti servise `zfs-mount.service` i `zfs.target`:

```bash
[tonib@archlinux ~]$ sudo systemctl enable zfs-mount.service
Created symlink /etc/systemd/system/zfs.target.wants/zfs-mount.service → /usr/lib/systemd/system/zfs-mount.service.
[tonib@archlinux ~]$ sudo systemctl enable zfs.target
Created symlink /etc/systemd/system/multi-user.target.wants/zfs.target → /usr/lib/systemd/system/zfs.target.

```

### Automatizacija periodičkog backupa

Osigurati ćemo automatizirano periodičko (na dnevnoj bazi) stvaranje sigurnosne kopije podataka u bazenu Gitlab_data, koju ćemo zatim pohraniti u bazen Gitlab-backup.

Koristiti ćemo alat [rsync](https://wiki.archlinux.org/title/rsync), pa ga možemo i instalirati:

```bash
[tonib@archlinux ~]$ sudo pacman -S rsync
resolving dependencies...
looking for conflicting packages...

Packages (2) xxhash-0.8.1-2  rsync-3.2.4-1

Total Download Size:   0.39 MiB
Total Installed Size:  0.90 MiB

:: Proceed with installation? [Y/n] y
:: Retrieving packages...
 rsync-3.2.4-1-x86_64  320.7 KiB   193 KiB/s 00:02 [######################] 100%
 xxhash-0.8.1-2-x...    80.2 KiB   290 KiB/s 00:00 [######################] 100%
 Total (2/2)           400.9 KiB   189 KiB/s 00:02 [######################] 100%
(2/2) checking keys in keyring                     [######################] 100%
(2/2) checking package integrity                   [######################] 100%
(2/2) loading package files                        [######################] 100%
(2/2) checking for file conflicts                  [######################] 100%
(2/2) checking available disk space                [######################] 100%
:: Processing package changes...
(1/2) installing xxhash                            [######################] 100%
(2/2) installing rsync                             [######################] 100%
:: Running post-transaction hooks...
(1/2) Reloading system manager configuration...
(2/2) Arming ConditionNeedsUpdate...
```

Osim toga, instalirati ćemo i  [cronie](https://wiki.archlinux.org/title/Cron), planer poslova zasnovan na vremenu u operativnim sustavima sličnim Unixu:

```bash
[tonib@archlinux ~]$ sudo pacman -S cronie
resolving dependencies...
looking for conflicting packages...

Packages (1) cronie-1.6.1-1

Total Download Size:   0.09 MiB
Total Installed Size:  0.21 MiB

:: Proceed with installation? [Y/n] y
:: Retrieving packages...
 cronie-1.6.1-1-x...    88.0 KiB   140 KiB/s 00:01 [######################] 100%
(1/1) checking keys in keyring                     [######################] 100%
(1/1) checking package integrity                   [######################] 100%
(1/1) loading package files                        [######################] 100%
(1/1) checking for file conflicts                  [######################] 100%
(1/1) checking available disk space                [######################] 100%
:: Processing package changes...
(1/1) installing cronie                            [######################] 100%
Optional dependencies for cronie
    smtp-server: send job output via email
    smtp-forwarder: forward job output to email server
:: Running post-transaction hooks...
(1/2) Reloading system manager configuration...
(2/2) Arming ConditionNeedsUpdate...
```

Treba omogućiti izvršavanje servisa `cronie`, odnosno njegovo automatsko pokretanje pri svakom boot-u:

```bash
[tonib@archlinux ~]$ sudo systemctl enable cronie.service
Created symlink /etc/systemd/system/multi-user.target.wants/cronie.service → /usr/lib/systemd/system/cronie.service.
```

Nakon što rebootamo sistem možemo se uvjeriti da se cronie.service uistinu pokreće naredbom systemctl status:

```bash
[tonib@archlinux ~]$ sudo reboot
[tonib@archlinux ~]$ Connection to 192.168.122.88 closed by remote host.
Connection to 192.168.122.88 closed.
toni@toni-WRT-WX9:~$ ssh tonib@192.168.122.88
tonib@192.168.122.88's password: 
Last login: Mon Jul 11 17:16:22 2022 from 192.168.122.1
[tonib@archlinux ~]$ systemctl status cronie.service
● cronie.service - Periodic Command Scheduler
     Loaded: loaded (/usr/lib/systemd/system/cronie.service; enabled; vendor pr>
     Active: active (running) since Mon 2022-07-11 17:17:21 UTC; 9s ago
   Main PID: 280 (crond)
      Tasks: 1 (limit: 1144)
     Memory: 980.0K
        CPU: 1ms
     CGroup: /system.slice/cronie.service
             └─280 /usr/bin/crond -n

Jul 11 17:17:21 archlinux systemd[1]: Started Periodic Command Scheduler.
Jul 11 17:17:21 archlinux crond[280]: (CRON) STARTUP (1.6.1)
Jul 11 17:17:21 archlinux crond[280]: (CRON) INFO (Syslog will be used instead >
Jul 11 17:17:21 archlinux crond[280]: (CRON) INFO (RANDOM_DELAY will be scaled >
Jul 11 17:17:21 archlinux crond[280]: (CRON) INFO (running with inotify support)
lines 1-15/15 (END)
```

Skriptu `backup.sh` koja će generirati sigurnosnu kopiju napraviti ćemo u direkotoriju `cron.daily`, koja će se na taj način pokretati jednom dnevno:

```bash
[tonib@archlinux ~]$ cd /etc/cron.daily
[tonib@archlinux cron.daily]$ sudo touch backup.sh
[tonib@archlinux cron.daily]$ sudo nano backup.sh
```

sa sljedećim sadržajem:

```sh
#!/bin/sh
rsync -a --delete --quiet /Gitlab_data /Gitlab_backup
```

Skriptu je za kraj potrebno učiniti izvršnom:

```bash
[tonib@archlinux cron.daily]$ sudo chmod +x backup.sh
```

## Konfiguracija vatrozida

Instaliramo paket nftables i omogućimo ga:

```bash
[tonib@archlinux ~]$ sudo pacman -S nftables
resolving dependencies...
looking for conflicting packages...

Packages (2) jansson-2.14-2  nftables-1:1.0.4-1

Total Download Size:   0.42 MiB
Total Installed Size:  1.16 MiB

:: Proceed with installation? [Y/n] y
:: Retrieving packages...
 nftables-1:1.0.4...   379.5 KiB   103 KiB/s 00:04 [######################] 100%
 jansson-2.14-2-x...    51.7 KiB  81.3 KiB/s 00:01 [######################] 100%
 Total (2/2)           431.2 KiB  91.1 KiB/s 00:05 [######################] 100%
(2/2) checking keys in keyring                     [######################] 100%
(2/2) checking package integrity                   [######################] 100%
(2/2) loading package files                        [######################] 100%
(2/2) checking for file conflicts                  [######################] 100%
(2/2) checking available disk space                [######################] 100%
:: Processing package changes...
(1/2) installing jansson                           [######################] 100%
(2/2) installing nftables                          [######################] 100%
Optional dependencies for nftables
    python: Python bindings [installed]
:: Running post-transaction hooks...
(1/2) Reloading system manager configuration...
(2/2) Arming ConditionNeedsUpdate...
[tonib@archlinux ~]$ sudo systemctl enable nftables.service
Created symlink /etc/systemd/system/multi-user.target.wants/nftables.service → /usr/lib/systemd/system/nftables.service.

```

Napravimo tablicu:

```bash
[tonib@archlinux ~]$ sudo nft add table inet moja_tablica
[tonib@archlinux ~]$ sudo nft list tables
table inet moja_tablica
```

Dodamo lanac u tablicu:

```bash
[tonib@archlinux ~]$ sudo nft add chain inet moja_tablica moj_lanac '{type filter hook input priority 0;}'
[tonib@archlinux ~]$ sudo nft list table inet moja_tablica
table inet moja_tablica {
 chain moj_lanac {
  type filter hook input priority filter; policy accept;
 }
}
```

Dodamo željena pravila:

```bash
[tonib@archlinux ~]$ sudo nft add rule inet moja_tablica moj_lanac tcp dport {22, 80, 433} accept
```

I promjenimo pravilo lanca:

```bash
[tonib@archlinux ~]$ sudo nft chain inet moja_tablica moj_lanac '{policy drop;}'
```

Krajnje stanje:

```bash
[tonib@archlinux ~]$ sudo nft list table inet moja_tablica
table inet moja_tablica {
  chain moj_lanac {
    type filter hook input priority filter; policy drop;
    tcp dport { 22, 80, 433 } accept
  }
}
```

Trenutne tablice, lance i pravila spremiti ćemo u `/etc/nftables.conf`, odkud servis nftables čita pri svakom bootu (ukoliko je servis omogućen). Za oboje se možemo pobrinuti na sljedeći način:

```bash
[tonib@archlinux ~]$ sudo systemctl enable nftables
Created symlink /etc/systemd/system/multi-user.target.wants/nftables.service → /usr/lib/systemd/system/nftables.service.
[tonib@archlinux etc]$ sudo systemctl enable nftables
[tonib@archlinux etc]$ sudo nft list ruleset > /etc/nftables.conf
-bash: /etc/nftables.conf: Permission denied
[tonib@archlinux etc]$ sudo -i
[root@archlinux ~]\# nft list ruleset > /etc/nftables.conf

```

## Automatizacija postavljanja web aplikacije alatom Ansible

### Instalacija alata Ansible

Instalirati ćemo alat Ansible te provjeriti je li instalacija uspješna:

```bash
[tonib@archlinux ~]$ sudo pacman -S ansible
[tonib@archlinux ~]$ ansible --version
ansible [core 2.13.1]
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/tonib/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3.10/site-packages/ansible
  ansible collection location = /home/tonib/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible
  python version = 3.10.5 (main, Jun  6 2022, 18:49:26) [GCC 12.1.0]
  jinja version = 3.1.2
  libyaml = True
```

Izmjenimo datoteku `etc/ansible/hosts` tako da bude u sljedećem obliku:

```conf
[local]
localhost ansible_connection=local
```

### Postavljanje usluge Gitlab uz pomoć Ansible playbooka

U korijenskom direktoriju stvoriti ćemo datoteku u formatu YAML pod nazivom `moj-playbook.yml`. U navedenoj datoteci "slagati" ćemo jednu po jednu komponentu koje su potrebne za postavljanje usluge Gitlab na Arch Linuxu, prema [preoporučenim koracima na ArchWikiju](https://wiki.archlinux.org/title/GitLab#Firewall).

### Gitlab

Najprije ćemo osvježiti liste master paketa Arch Linuxa:

```yml
---
- name: Setup Gitlab infrastructure
  hosts: all
  become: yes

  tasks:
    - name: Update lists
      ansible.builtin.pacman:
        update_cache: yes
```

Tada se možemo pobrinuti za sam Gitlab:

```yml
ttasks:
    - name: Update lists
      ansible.builtin.pacman:
        update_cache: yes

    #GITLAB
    - name: Ensure gitlab is present
      ansible.builtin.pacman:
        name: gitlab
        state: present

    - name: Write secrets
      shell: |
        hexdump -v -n 64 -e '1/1 "%02x"' /dev/urandom > /etc/webapps/gitlab/secret
        chmod 640 /etc/webapps/gitlab/secret
        hexdump -v -n 64 -e '1/1 "%02x"' /dev/urandom > /etc/webapps/gitlab-shell/secret
        chmod 640 /etc/webapps/gitlab-shell/secret

    - name: Template file secrets.yml
      ansible.builtin.template:
        src: ./secrets.yml.j2
        dest: /etc/webapps/gitlab/secrets.yml

```

Osigurali smo da je Gitlab instaliran. Popunili smo datoteke čiji se sadržaj koristi za generiranje autentifikacijskih tokena i slično te dodjelili određene privilegije nad istima. Formirali smo izgled datoteke `secrets.yml` na navedenoj putanji prema korištenom templateu koji se nalazi u korijenskom direktoriju.

Sadržaj templatea `secrets.yml.j2` je sljedeći:

```yml
production:
  secret_key_base: "{{ secret1 }}"
  db_key_base: "{{ secret1 }}"
  otp_key_base: "{{ secret1 }}"
  openid_connect_signing_key: "{{ secret1 }}"
```

Naravno, u stvarnosti bi za svaku stavku koristili drugačiji secret string, ali zbog čitljivosti i jednostavnosti ćemo u ovom primjeru koristiti iste.

Varijable u vitičastim zagradama pohraniti ćemo u Ansible Vault. Proces je sljedeći:

- stvorimo datoteku `vault_pass.txt` u koju pohranimo svoju master zaporku

- iskoristimo naredbu oblika `$ ansible-vault encrypt_string --vault-id vault_pass.txt 'the var content' --name varname` gdje je `vault_pass.txt` datoteka sa našom master zaporkom, `the var content` sadržaj kojeg želimo enkriptirati, a `varname` ime varijable u koju ćemo pohraniti enkriptirani sadržaj.

- dobiveni enkriptirani sadržaj postavimo u playbook kao sadržaj varijable

U našem slučaju, to izgleda ovako:

```bash
[tonib@archlinux ~]$ touch vault_pass.txt
[tonib@archlinux ~]$ nano vault_pass.txt
[tonib@archlinux ~]$ hexdump -v -n 64 -e '1/1 "%02x"' /dev/urandom
8cbca4b2c747432c07895a1e58994f92769a6e62c34a38199ac53745c10953bd3ce43cca7a971decd91308823cbd6d238d4b94fdc362075c4514f0b8a2b2bc8c
[tonib@archlinux ~]$ ansible-vault encrypt_string --vault-id vault_pass.txt '8cbca4b2c747432c07895a1e58994f92769a6e62c34a38199ac53745c10953bd3ce43cca7a971decd91308823cbd6d238d4b94fdc362075c4514f0b8a2b2bc8c' --name secret1
Encryption successful
secret1: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          37316139373763626432393438393439383962323064393633666363393937613639643133656436
          3165393365326366653964666362303262663836643462340a343562663433333439396332663239
          32353862383935633362383861656232396266373932666534623165626532356539613065643336
          6232326538643864630a623038366137663830343431333439353038316330316333653438343638
          61333661646539323935623836313961653464613331306338646139626139633537653563656231
          36383866613732323939643864636234306538396337303336313764663439663763636538666539
          35346562383535636164333537643433333831393633653033373038373533363231653963303461
          66663435386464626634356232353438346664663836323663616435316639646633313136663562
          65646235663438303066393362313130336430323434323961336532663333313164663365666230
          36666634663564643666386633303435376432613565393464336161316435613364386436633063
          356232333464343564633965386137313736

```

te je sada sadržaj `moj-playbook.yml`:

```yml
---
- name: Setup Gitlab infrastructure
  hosts: all
  become: yes
  vars:
    secret1: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          65613061376439643661383732633236643534363431363233343733623331303565353965346138
          3337386462343735393531623838656466613338313634360a393930333565326166383532333539
          63643433316139323666643763326562353230633665393665646239303531326530393239383933
          3532663939666330650a333230303439386234663236643962636637363261393164376663373463
          33623737313234393535653431336664666638396166396135373936336532656430306564366665
          64353536333935656639356263363565343037333436373365316662313862663833616235666162
          61316632626664383033653432636164316436326363613865613139303538303764666261666665
          62356139333432613262376263366466646135333535633665633162346238363266356162646161
          35663134383039616231396137633734333030353433343464363763653139613135336364323636
          66373134383264353734356562343534363266313732313838323138366361323364303561363665
          323361343431356437313033353362333830

  tasks:
    - name: Update lists
      ansible.builtin.pacman:
        update_cache: yes

    #GITLAB
    - name: Ensure gitlab is present
      ansible.builtin.pacman:
        name: gitlab
        state: present

    - name: Write secrets
      shell: |
        hexdump -v -n 64 -e '1/1 "%02x"' /dev/urandom > /etc/webapps/gitlab/secret
        chmod 640 /etc/webapps/gitlab/secret
        hexdump -v -n 64 -e '1/1 "%02x"' /dev/urandom > /etc/webapps/gitlab-shell/secret
        chmod 640 /etc/webapps/gitlab-shell/secret

    - name: Template file secrets.yml
      ansible.builtin.template:
        src: ./secrets.yml.j2
        dest: /etc/webapps/gitlab/secrets.yml
```

Pokrenuti ćemo `moj-playbook.yml` pritom navodeći datoteku u kojoj je pohranjena naša master zaporka kako bismo pristupili vaultu:

```bash
[tonib@archlinux ~]$ ansible-playbook moj-playbook.yml --vault-password-file=vault_pass.txt

PLAY [Setup Gitlab] ******************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************
[WARNING]: Platform linux on host localhost is using the discovered Python interpreter at /usr/bin/python3.10, but future installation of another Python
interpreter could change the meaning of that path. See https://docs.ansible.com/ansible-core/2.13/reference_appendices/interpreter_discovery.html for more
information.
ok: [localhost]

TASK [Update lists] ******************************************************************************************************************************************
changed: [localhost]

TASK [Ensure gitlab is at the latest version] ****************************************************************************************************************
ok: [localhost]

TASK [Write secrets] *****************************************************************************************************************************************
changed: [localhost]

TASK [Template a file to /etc/webapps/gitlab/secrets.yml] ****************************************************************************************************
ok: [localhost]

PLAY RECAP ***************************************************************************************************************************************************
localhost                  : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```

Možemo se uvjeriti da je datoteka generirana kako treba:

```bash
[tonib@archlinux ~]$ cat /etc/webapps/gitlab/secrets.yml
production:
  secret_key_base: "8cbca4b2c747432c07895a1e58994f92769a6e62c34a38199ac53745c10953bd3ce43cca7a971decd91308823cbd6d238d4b94fdc362075c4514f0b8a2b2bc8c"
  db_key_base: "8cbca4b2c747432c07895a1e58994f92769a6e62c34a38199ac53745c10953bd3ce43cca7a971decd91308823cbd6d238d4b94fdc362075c4514f0b8a2b2bc8c"
  otp_key_base: "8cbca4b2c747432c07895a1e58994f92769a6e62c34a38199ac53745c10953bd3ce43cca7a971decd91308823cbd6d238d4b94fdc362075c4514f0b8a2b2bc8c"
  openid_connect_signing_key: "8cbca4b2c747432c07895a1e58994f92769a6e62c34a38199ac53745c10953bd3ce43cca7a971decd91308823cbd6d238d4b94fdc362075c4514f0b8a2b2bc8c"

```

### PostgreSQL

Sljedeći korak je postavljanje baze što u `moj-playbook.yml` izgleda ovako:

```yaml
---
- name: Setup Gitlab
  hosts: all
  become: yes
  vars:
    ime_baze: gitlabhq_production
    user: gitlab
    user_password: !vault |
      $ANSIBLE_VAULT;1.1;AES256
      30376165643161383366343238376538303830623738313933363865646233633031393538623937
      6638363936636135653332363133636539666238626562330a323538646561633664373062393139
      64353430396239656663363532356238316135306237366266316533626236623763376365623434
      3265616435656435360a306335356461656636636266303766306634613432633632623731633134
      3962
    secret1: !vault |
      $ANSIBLE_VAULT;1.1;AES256
      37316139373763626432393438393439383962323064393633666363393937613639643133656436
      3165393365326366653964666362303262663836643462340a343562663433333439396332663239
      32353862383935633362383861656232396266373932666534623165626532356539613065643336
      6232326538643864630a623038366137663830343431333439353038316330316333653438343638
      61333661646539323935623836313961653464613331306338646139626139633537653563656231
      36383866613732323939643864636234306538396337303336313764663439663763636538666539
      35346562383535636164333537643433333831393633653033373038373533363231653963303461
      66663435386464626634356232353438346664663836323663616435316639646633313136663562
      65646235663438303066393362313130336430323434323961336532663333313164663365666230
      36666634663564643666386633303435376432613565393464336161316435613364386436633063
      356232333464343564633965386137313736

  tasks:
...

#POSTGRES
    - name: Ensure postgres is at the latest version
      ansible.builtin.pacman:
        name: postgresql
        state: latest

    - name: Ensure psycopg2 is at the latest version
      ansible.builtin.pacman:
        name: python-psycopg2
        state: latest

    - name: Create a directory if it does not exist
      ansible.builtin.file:
        path: /var/lib/postgres/data
        state: directory
        mode: "0755"
        owner: postgres
        group: postgres

    - name: Configure postgres
      become_user: postgres
      ansible.builtin.command:
        cmd: initdb -D /var/lib/postgres/data

    - name: Ensure that postgres is started
      ansible.builtin.service:
        name: postgresql
        state: started

    - name: Create a new database
      community.postgresql.postgresql_db:
        name: "{{ime_baze}}"

    - name: Create database user
      community.postgresql.postgresql_user:
        db: "{{ime_baze}}"
        name: "{{user}}"
        password: "{{user_password}}"
        role_attr_flags: SUPERUSER

    - name: Template a file to "/etc/webapps/gitlab/database.yml"
      ansible.builtin.template:
        src: ./database.yml.j2
        dest: "/etc/webapps/gitlab/database.yml"
```

Osigurali smo da je Postgres instaliran, kao i paket psycopg2 koji je potreban. Kreirali smo direktorij u kojem smo zatim incijalizirali bazu. Pokrenuli smo Postgres, pa zatim stvorili bazu s imenom kojeg smo pohranili u varijablu na početku, dakle `gitlabhq_production`. Kreirali smo i korisnika s imenom i lozinkom koje smo također pohranili u varijable i naveli bazu na koju će se spojiti, te naveli koje privilegije će imati. U ovom slučaju je to SUPERUSER što osigurava da Gitlab instalira određene potrebne ekstenzije. Formirali smo izgled datoteke `database.yml` na navedenoj putanji prema korištenom templateu koji se nalazi u korijenskom direktoriju.

Sadržaj templatea `database.yml.j2`:

```yml

#
# PRODUCTION
#
production:
  main:
    adapter: postgresql
    encoding: unicode
    database: gitlabhq_production
    username: gitlab
    password: "{{ user_password }}"
    host: localhost
    # load_balancing:
    #   hosts:
    #     - host1.example.com
    #     - host2.example.com
    #   discover:
    #     nameserver: 1.2.3.4
    #     port: 8600
    #     record: secondary.postgresql.service.consul
    #     interval: 300

#
# Development specific
#
development:
  main:
    adapter: postgresql
    encoding: unicode
    database: gitlabhq_development
    username: postgres
    password: "secure password"
    host: localhost
    variables:
      statement_timeout: 15s

#
# Staging specific
#
staging:
  main:
    adapter: postgresql
    encoding: unicode
    database: gitlabhq_staging
    username: gitlab
    password: "secure password"
    host: localhost

# Warning: The database defined as "test" will be erased and
# re-generated from your development database when you run "rake".
# Do not set this db to the same as development or production.
test: &test
  main:
    adapter: postgresql
    encoding: unicode
    database: gitlabhq_test
    username: postgres
    password:
    host: localhost
    prepared_statements: false
    variables:
      statement_timeout: 15s

```

Nakon što izvršimo playbook, logiranjem u bazu uvjerimo se da sve radi ispravno:

```bash
[tonib@archlinux ~]$ sudo -u gitlab -i
[gitlab@archlinux ~]$ psql -d gitlabhq_production
psql (14.3)
Type "help" for help.

gitlabhq_production=# 

```

### Redis

Potrebno je i postaviti cache bazu Redis za postizanje dostojnih performansi, a to u `moj-playbook.yml` izgleda ovako:

```yaml
...
#REDIS
    - name: Ensure Redis is at the latest version
      ansible.builtin.pacman:
        name: redis
        state: latest

    - name: Enable service redis and ensure it is not masked
      ansible.builtin.systemd:
        name: redis
        enabled: yes
        masked: no

      #redis.conf
    - name: Touch a file, using symbolic modes to set the permissions (equivalent to 0644)
      ansible.builtin.file:
        path: ./redis.conf
        state: touch
        mode: u=rw,g=r,o=r

    - name: Add custom configuration at ./redis.conf
      ansible.builtin.blockinfile:
        path: ./redis.conf
        block: |
          port 6379
          unixsocket /run/redis/redis.sock
          unixsocketperm 770

    - name: Copy custom config to /etc/redis/redis.conf
      ansible.builtin.copy:
        src: ./redis.conf
        dest: /etc/redis/redis.conf

      #resque.yml
    - name: Touch a file, using symbolic modes to set the permissions (equivalent to 0644)
      ansible.builtin.file:
        path: ./resque.yml
        state: touch
        mode: u=rw,g=r,o=r

    - name: Add custom configuration at ./resque.yml
      ansible.builtin.blockinfile:
        path: ./resque.yml
        block: |
          development:
            url: unix:/run/redis/redis.sock
          test:
            url: unix:/run/redis/redis.sock
          production:
            url: unix:/run/redis/redis.sock

    - name: Template a file to /etc/webapps/gitlab/resque.yml
      ansible.builtin.copy:
        src: ./resque.yml
        dest: /etc/webapps/gitlab/resque.yml

    - name: Add the user 'gitlab' to group 'redis'
      ansible.builtin.user:
        name: gitlab
        group: redis

    - name: Restart service redis
      ansible.builtin.service:
        name: redis
        state: restarted
    
    - name: Ensure Nginx is at the latest version
      ansible.builtin.pacman:
        name: nginx
        state: latest

    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

Osigurali smo da je Redis instaliran. Iskoristili smo uslugu systemd kako bi se servis omogućio te osiguralo da nije maskiran. Stvorili smo `redis.conf` konfiguracijsku datoteku s određenim dozvolama u korijenskom direktoriju, ispunili je, a zatim njen sadržaj kopirali u zadanu redis konfiguraciju. Isto smo napravili i s datotekom `resque.yml`. Dodali smo korisnika gitlab u korisničku grupu redis.

Datoteka `./redis.conf` sljedećeg je sadržaja (zapisali smo ga automatski u playbooku):

```conf
# BEGIN ANSIBLE MANAGED BLOCK
port 6379
unixsocket /run/redis/redis.sock
unixsocketperm 770
# END ANSIBLE MANAGED BLOCK
```

Datoteka `./resque.yml` sljedećeg je sadržaja (zapisali smo ga automatski u playbooku):

```yml
[tonib@archlinux ~]$ cat resque.yml
# BEGIN ANSIBLE MANAGED BLOCK
development:
  url: unix:/run/redis/redis.sock
test:
  url: unix:/run/redis/redis.sock
production:
  url: unix:/run/redis/redis.sock
# END ANSIBLE MANAGED BLOCK

```

```bash
[tonib@archlinux ~]$ ansible-playbook moj-playbook.yml --vault-password-file=vault_pass.txt

TASK [Ensure Redis is at the latest version] *****************************************************************************************************************
ok: [localhost]

TASK [Enable service redis and ensure it is not masked] ******************************************************************************************************
ok: [localhost]

TASK [Touch a file, using symbolic modes to set the permissions (equivalent to 0644)] ************************************************************************
changed: [localhost]

TASK [Add custom configuration at ./redis.conf] **************************************************************************************************************
ok: [localhost]

TASK [Copy custom config to /etc/redis/redis.conf] ***********************************************************************************************************
changed: [localhost]

TASK [Touch a file, using symbolic modes to set the permissions (equivalent to 0644)] ************************************************************************
changed: [localhost]

TASK [Add custom configuration at ./resque.yml] **************************************************************************************************************
ok: [localhost]

TASK [Template a file to /etc/webapps/gitlab/resque.yml] *****************************************************************************************************
ok: [localhost]

TASK [Add the user 'gitlab' to group 'redis'] ****************************************************************************************************************
ok: [localhost]

TASK [Restart service redis] *********************************************************************************************************************************
changed: [localhost]

PLAY RECAP ***************************************************************************************************************************************************
localhost                  : ok=25   changed=17   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 

```

### Gitlab - inicijalizacija

Potrebno je pokrenuti inicijalizrati Gitlab, a to u `moj-playbook.yml` izgleda ovako:

```yml
#GITLAB
    - name: Configure permissions for smtp_setting.rb
      ansible.builtin.command:
        chdir: /etc/webapps/gitlab
        cmd: sudo chown gitlab smtp_settings.rb
    
    - name: Ensure python-pip is present
      ansible.builtin.pacman:
        name: python-pip
        state: present
    
    - name: Ensure py module pexpect is present
      ansible.builtin.pip:
        name: pexpect
        state: present
    
    - name: Setup Gitlab
      ansible.builtin.expect:
        echo: yes
        chdir: /usr/share/webapps/gitlab
        command: /bin/bash -c "sudo -u gitlab $(cat environment | xargs) bundle-2.7 exec rake gitlab:setup"
        timeout: 500
        responses:
          'to continue': 'yes'
```

Promjenili smo vlasnika datoteke `smtp_setting.rb`, u suprotnome inicijalizacija ne bi prolazila. Nakon toga smo osigurali da je instaliran pip i python paket pexpect kako bismo mogli koristit ansibleov ugrađeni modul `expect`. Zatim smo uz pomoć navedenog modula sproveli incijalizaciju, te odgovorili automatski odgovorili "yes" na prompt koji se pojavljuje tijekom tog procesa.

### Nginx

Za pristup GitLabu s vanjske mreže, dokumentacija preporučuje korištenje web poslužitelja kao proxyja. Sve upite s web poslužitelja prema GitLabu obrađuje GitLab Workhorse, koji odlučuje kako će se obraditi. Koristiti ćemo Nginx, a postavili smo ga u `moj-playbook.yml` na sljedeći način:

```yml
#NGINX
    - name: Ensure Nginx is present
      ansible.builtin.pacman:
        name: nginx
        state: present
    
    - name: Copy nginx.conf
      ansible.builtin.copy:
        src: ./nginx.conf
        dest: /etc/nginx/nginx.conf
    
    - name: Create directory
      ansible.builtin.file:
        path: /etc/nginx/sites-available
        state: directory
    
    - name: Create directory
      ansible.builtin.file:
        path: /etc/nginx/sites-enabled
        state: directory
    
    - name: Copy gitlab
      ansible.builtin.copy:
        src: ./gitlab
        dest: /etc/nginx/sites-available/gitlab
    
    - name: Create symbolic link 
      ansible.builtin.file:
        src: /etc/nginx/sites-available/gitlab
        dest: /etc/nginx/sites-enabled/gitlab
        state: link
      
    - name: Restart nginx
      ansible.builtin.systemd:
        name: nginx
        enabled: yes
        state: restarted
```

Osigurali smo da je Nginx instaliran. Zadanu konfiguraciju `/etc/nginx/nginx.conf` zamjenili smo prilagođenom koja se nalazi u istoimenoj datoteci u korijenskom direktoriju. Kreirali smo direkotorije `sites-available` i `sites-enabled` kako bismo na jednostavan način mogli omogućiti određeno web stranice kada to poželimo. U `sites-available` postavili smo konfiguracijsku datoteku `gitlab`, a zatim i kreirali link između `/etc/nginx/sites-available/gitlab` i `/etc/nginx/sites-enabled/gitlab` Na kraju smo restartali servis `nginx` kako bi se odrazile promjene.

Datoteka `nginx.conf` u korijenskom direktoriju sljedećeg je sadržaja:

```conf
worker_processes  1;

events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

include sites-enabled/*;
}

```

Datoteka `gitlab` u `etc/nginx/sites-available` sljedećeg je sadržaja:

```conf
upstream gitlab-workhorse {
  server unix:/run/gitlab/gitlab-workhorse.socket fail_timeout=0;
}

server {
  listen 80;                  # IPv4 HTTP
  #listen 443 ssl http2;      # uncomment to enable IPv4 HTTPS + HTTP/2
  #listen [::]:80;            # uncomment to enable IPv6 HTTP
  #listen [::]:443 ssl http2; # uncomment to enable IPv6 HTTPS + HTTP/2
  server_name example.com;

  access_log  /var/log/gitlab/nginx_access.log;
  error_log   /var/log/gitlab/nginx_error.log;

  #ssl_certificate ssl/example.com.crt;
  #ssl_certificate_key ssl/example.com.key;

  location ~ ^/(assets)/ {
    root /usr/share/webapps/gitlab/public;
    gzip_static on; # to serve pre-gzipped version
    expires max;
    add_header Cache-Control public;
  }

  location / {
      # unlimited upload size in nginx (so the setting in GitLab applies)
      client_max_body_size 0;

      # proxy timeout should match the timeout value set in /etc/webapps/gitlab/puma.rb
      proxy_read_timeout 60;
      proxy_connect_timeout 60;
      proxy_redirect off;

      proxy_set_header Host $http_host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;

      #proxy_set_header X-Forwarded-Ssl on;

      proxy_pass http://gitlab-workhorse;
  }

  error_page 404 /404.html;
  error_page 422 /422.html;
  error_page 500 /500.html;
  error_page 502 /502.html;
  error_page 503 /503.html;
  location ~ ^/(404|422|500|502|503)\.html$ {
    root /usr/share/webapps/gitlab/public;
    internal;
  }
}

```

### Ponovo pokretanje potrebnih procesa

Na kraju smo restartali sve potrebne procese:

```yml
#RESTARTS
    - name: Restart postgres
      ansible.builtin.systemd:
        name: postgresql
        enabled: yes
        state: restarted
      
    - name: Restart redis
      ansible.builtin.systemd:
        name: redis
        enabled: yes
        state: restarted

    - name: Restart gitlab.target
      ansible.builtin.systemd:
        name: gitlab.target
        enabled: yes
        state: restarted
    
    - name: Restart gitlab.gitaly
      ansible.builtin.systemd:
        name: gitlab-gitaly
        enabled: yes
        state: restarted
    
    - name: Restart gitlab-puma
      ansible.builtin.systemd:
        name: gitlab-puma
        enabled: yes
        state: restarted
    
    - name: Restart gitlab-workhorse
      ansible.builtin.systemd:
        name: gitlab-workhorse
        enabled: yes
        state: restarted
```

### Testiranje infrastrukture

Datoteka `moj-playbook.yml` je na kraju sljedećeg izgleda:

```yml
---
- name: Setup Gitlab infrastructure
  hosts: all
  become: yes
  vars:
    ime_baze: gitlabhq_production
    user: gitlab
    user_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          31353033343035336531643163323231363763393432346565353263653836396462333962636538
          6563333762346232346538653531626532636435643665370a626666656563666437653834636164
          62336134636235336363643861303633653834383261346264313036313830306537653666313863
          6165306638633039330a643536633464393837393361643938343932396438383861386630313461
          3033
    secret1: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          65613061376439643661383732633236643534363431363233343733623331303565353965346138
          3337386462343735393531623838656466613338313634360a393930333565326166383532333539
          63643433316139323666643763326562353230633665393665646239303531326530393239383933
          3532663939666330650a333230303439386234663236643962636637363261393164376663373463
          33623737313234393535653431336664666638396166396135373936336532656430306564366665
          64353536333935656639356263363565343037333436373365316662313862663833616235666162
          61316632626664383033653432636164316436326363613865613139303538303764666261666665
          62356139333432613262376263366466646135333535633665633162346238363266356162646161
          35663134383039616231396137633734333030353433343464363763653139613135336364323636
          66373134383264353734356562343534363266313732313838323138366361323364303561363665
          323361343431356437313033353362333830



  tasks:
    - name: Update lists
      ansible.builtin.pacman:
        update_cache: yes

    #GITLAB
    - name: Ensure gitlab is present
      ansible.builtin.pacman:
        name: gitlab
        state: present

    - name: Write secrets
      shell: |
        hexdump -v -n 64 -e '1/1 "%02x"' /dev/urandom > /etc/webapps/gitlab/secret
        chmod 640 /etc/webapps/gitlab/secret
        hexdump -v -n 64 -e '1/1 "%02x"' /dev/urandom > /etc/webapps/gitlab-shell/secret
        chmod 640 /etc/webapps/gitlab-shell/secret

    - name: Template file secrets.yml
      ansible.builtin.template:
        src: ./secrets.yml.j2
        dest: /etc/webapps/gitlab/secrets.yml

    #POSTGRES
    - name: Ensure postgres is present
      ansible.builtin.pacman:
        name: postgresql
        state: present

    - name: Ensure psycopg2 is present
      ansible.builtin.pacman:
        name: python-psycopg2
        state: present

    - name: Create a directory if it does not exist
      ansible.builtin.file:
        path: /var/lib/postgres/data
        state: directory
        mode: "0755"
        owner: postgres
        group: postgres
    

    - name: Configure postgres
      become_user: postgres
      ansible.builtin.command:
        cmd: initdb -D /var/lib/postgres/data

    - name: Ensure that postgres is started
      ansible.builtin.service:
        name: postgresql
        enabled: yes
        state: started

    - name: Create a new database
      community.postgresql.postgresql_db:
        name: "{{ime_baze}}"

    - name: Create database user
      community.postgresql.postgresql_user:
        db: "{{ime_baze}}"
        name: "{{user}}"
        password: "{{user_password}}"
        role_attr_flags: SUPERUSER

    - name: Template file database.yml
      ansible.builtin.template:
        src: ./database.yml.j2
        dest: "/etc/webapps/gitlab/database.yml"

    #REDIS
    - name: Ensure Redis is present
      ansible.builtin.pacman:
        name: redis
        state: present

    - name: Enable service redis and ensure it is not masked
      ansible.builtin.systemd:
        name: redis
        enabled: yes
        masked: no

      #redis.conf
    - name: Touch a file, using symbolic modes to set the permissions (equivalent to 0644)
      ansible.builtin.file:
        path: ./redis.conf
        state: touch
        mode: u=rw,g=r,o=r

    - name: Add custom configuration at ./redis.conf
      ansible.builtin.blockinfile:
        path: ./redis.conf
        block: |
          port 6379
          unixsocket /run/redis/redis.sock
          unixsocketperm 770

    - name: Copy redis.conf
      ansible.builtin.copy:
        src: ./redis.conf
        dest: /etc/redis/redis.conf

      #resque.yml
    - name: Touch a file, using symbolic modes to set the permissions (equivalent to 0644)
      ansible.builtin.file:
        path: ./resque.yml
        state: touch
        mode: u=rw,g=r,o=r

    - name: Add custom configuration at ./resque.yml
      ansible.builtin.blockinfile:
        path: ./resque.yml
        block: |
          development:
            url: unix:/run/redis/redis.sock
          test:
            url: unix:/run/redis/redis.sock
          production:
            url: unix:/run/redis/redis.sock

    - name: Copy resque.yml
      ansible.builtin.copy:
        src: ./resque.yml
        dest: /etc/webapps/gitlab/resque.yml

    - name: Add the user 'gitlab' to group 'redis'
      ansible.builtin.user:
        name: gitlab
        group: redis

    - name: Restart service redis
      ansible.builtin.service:
        name: redis
        state: restarted
    
    #GITLAB
    - name: Configure permissions for smtp_setting.rb
      ansible.builtin.command:
        chdir: /etc/webapps/gitlab
        cmd: sudo chown gitlab smtp_settings.rb
    
    - name: Ensure python-pip is present
      ansible.builtin.pacman:
        name: python-pip
        state: present
    
    - name: Ensure py module pexpect is present
      ansible.builtin.pip:
        name: pexpect
        state: present
    
    - name: Setup Gitlab
      ansible.builtin.expect:
        echo: yes
        chdir: /usr/share/webapps/gitlab
        command: /bin/bash -c "sudo -u gitlab $(cat environment | xargs) bundle-2.7 exec rake gitlab:setup"
        timeout: 500
        responses:
          'to continue': 'yes'
    
    #NGINX
    - name: Ensure Nginx is present
      ansible.builtin.pacman:
        name: nginx
        state: present
    
    - name: Copy nginx.conf
      ansible.builtin.copy:
        src: ./nginx.conf
        dest: /etc/nginx/nginx.conf
    
    - name: Create directory
      ansible.builtin.file:
        path: /etc/nginx/sites-available
        state: directory
    
    - name: Create directory
      ansible.builtin.file:
        path: /etc/nginx/sites-enabled
        state: directory
    
    - name: Copy gitlab
      ansible.builtin.copy:
        src: ./gitlab
        dest: /etc/nginx/sites-available/gitlab
    
    - name: Create symbolic link 
      ansible.builtin.file:
        src: /etc/nginx/sites-available/gitlab
        dest: /etc/nginx/sites-enabled/gitlab
        state: link
      
    - name: Restart nginx
      ansible.builtin.systemd:
        name: nginx
        enabled: yes
        state: restarted
    
    #RESTARTS
    - name: Restart postgres
      ansible.builtin.systemd:
        name: postgresql
        enabled: yes
        state: restarted
      
    - name: Restart redis
      ansible.builtin.systemd:
        name: redis
        enabled: yes
        state: restarted

    - name: Restart gitlab.target
      ansible.builtin.systemd:
        name: gitlab.target
        enabled: yes
        state: restarted
    
    - name: Restart gitlab.gitaly
      ansible.builtin.systemd:
        name: gitlab-gitaly
        enabled: yes
        state: restarted
    
    - name: Restart gitlab-puma
      ansible.builtin.systemd:
        name: gitlab-puma
        enabled: yes
        state: restarted
    
    - name: Restart gitlab-workhorse
      ansible.builtin.systemd:
        name: gitlab-workhorse
        enabled: yes
        state: restarted
    
```

Pokrenuti ćemo je i pričekati da se izvrši:

```bash
[tonib@archlinux ~]$ ansible-playbook moj-playbook.yml --vault-password-file=vault_pass.txt

PLAY [Setup Gitlab infrastructure] ***************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************
[WARNING]: Platform linux on host localhost is using the discovered Python interpreter at /usr/bin/python3.10, but future installation of another Python
interpreter could change the meaning of that path. See https://docs.ansible.com/ansible-core/2.13/reference_appendices/interpreter_discovery.html for more
information.
ok: [localhost]

TASK [Update lists] ******************************************************************************************************************************************
changed: [localhost]

TASK [Ensure gitlab is present] ******************************************************************************************************************************
changed: [localhost]

TASK [Write secrets] *****************************************************************************************************************************************
changed: [localhost]

TASK [Template file secrets.yml] *****************************************************************************************************************************
ok: [localhost]

TASK [Ensure postgres is present] ****************************************************************************************************************************
changed: [localhost]

TASK [Ensure psycopg2 is present] ****************************************************************************************************************************
ok: [localhost]

TASK [Create a directory if it does not exist] ***************************************************************************************************************
changed: [localhost]

TASK [Configure postgres] ************************************************************************************************************************************
[WARNING]: Unable to use /var/lib/postgres/.ansible/tmp as temporary directory, failing back to system: [Errno 13] Permission denied:
'/var/lib/postgres/.ansible'
changed: [localhost]

TASK [Ensure that postgres is started] ***********************************************************************************************************************
changed: [localhost]

TASK [Create a new database] *********************************************************************************************************************************
changed: [localhost]

TASK [Create database user] **********************************************************************************************************************************
changed: [localhost]

TASK [Template file database.yml] ****************************************************************************************************************************
changed: [localhost]

TASK [Ensure Redis is present] *******************************************************************************************************************************
ok: [localhost]

TASK [Enable service redis and ensure it is not masked] ******************************************************************************************************
ok: [localhost]

TASK [Touch a file, using symbolic modes to set the permissions (equivalent to 0644)] ************************************************************************
changed: [localhost]

TASK [Add custom configuration at ./redis.conf] **************************************************************************************************************
ok: [localhost]

TASK [Copy redis.conf] ***************************************************************************************************************************************
changed: [localhost]

TASK [Touch a file, using symbolic modes to set the permissions (equivalent to 0644)] ************************************************************************
changed: [localhost]

TASK [Add custom configuration at ./resque.yml] **************************************************************************************************************
ok: [localhost]

TASK [Copy resque.yml] ***************************************************************************************************************************************
changed: [localhost]

TASK [Add the user 'gitlab' to group 'redis'] ****************************************************************************************************************
ok: [localhost]

TASK [Restart service redis] *********************************************************************************************************************************
changed: [localhost]

TASK [Configure permissions for smtp_setting.rb] *************************************************************************************************************
changed: [localhost]

TASK [Ensure python-pip is present] **************************************************************************************************************************
ok: [localhost]

TASK [Ensure py module pexpect is present] *******************************************************************************************************************
ok: [localhost]

TASK [Setup Gitlab] ******************************************************************************************************************************************
changed: [localhost]

TASK [Ensure Nginx is present] *******************************************************************************************************************************
ok: [localhost]

TASK [Copy nginx.conf] ***************************************************************************************************************************************
ok: [localhost]

TASK [Create directory] **************************************************************************************************************************************
ok: [localhost]

TASK [Create directory] **************************************************************************************************************************************
ok: [localhost]

TASK [Copy gitlab] *******************************************************************************************************************************************
ok: [localhost]

TASK [Create symbolic link] **********************************************************************************************************************************
ok: [localhost]

TASK [Restart nginx] *****************************************************************************************************************************************
changed: [localhost]

TASK [Restart postgres] **************************************************************************************************************************************
changed: [localhost]

TASK [Restart redis] *****************************************************************************************************************************************
changed: [localhost]

TASK [Restart gitlab.target] *********************************************************************************************************************************
changed: [localhost]

TASK [Restart gitlab.gitaly] *********************************************************************************************************************************
changed: [localhost]

TASK [Restart gitlab-puma] ***********************************************************************************************************************************
changed: [localhost]

TASK [Restart gitlab-workhorse] ******************************************************************************************************************************
changed: [localhost]

PLAY RECAP ***************************************************************************************************************************************************
localhost                  : ok=40   changed=24   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

```

Korištenjem naredbe `curl` možemo se uvjeriti da se GitLab uistinu poslužuje, i to na adresi `localhost:80`.

```bash
[tonib@archlinux ~]$ curl localhost:80
<html><body>You are being <a href="http://localhost/users/sign_in">redirected</a>.</body></html>[tonib@archlinux ~]$ 
```
