# pxe air gapped

У цій  інструкції  описано  процес  розгортання TFTP-сервера  для  завантаження  системи  по  мережі  за  допомогою PXE. На  tftp-сервер  треба  встановити  такі  пакети  tftp, dhcp, httpd.
```
yum install –y dhcp-server tftp-server httpd
```
Файл /etc/dhpc/dhcpd.conf повинен мати таку конфігурацію. Задаєте діапазон ip-адрес самі, найголовніше next-server там повинна бути адреса вашого tftp сервера 
```
option architecture-type code 93 = unsigned integer 16;

subnet 192.168.69.0 netmask 255.255.255.0 {
  option routers 192.168.69.1;
  option domain-name-servers 192.168.69.1;
  range 192.168.69.100 192.168.69.200;
  class "pxeclients" {
    match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
    next-server 192.168.69.8;
          if option architecture-type = 00:07 {
            filename "debian-installer/amd64/bootnetx64.efi";
          }
          else {
            filename "pxelinux/pxelinux.0";
          }
  }
  class "httpclients" {
    match if substring (option vendor-class-identifier, 0, 10) = "HTTPClient";
    option vendor-class-identifier "HTTPClient";
    filename "http://192.168.69.8/redhat/EFI/BOOT/BOOTX64.EFI";
  }
}

```

Перейдемо до папки, де буде наш netboot iso 
```
cd /var/lib/tftpboot/ 
```
Скачаємо його 
```
wget http://ftp.debian.org/debian/dists/bookworm/main/installer-amd64/current/images/netboot/netboot.tar.gz
```
```
tar -xzvf netboot.tar.gz
```
```
rm netboot.tar.gz
```
```
ln -s debian-installer/amd64/grubx64.efi .
```
```
ln -s debian-installer/amd64/grub .
```
Перезапустимо dhcp та включимо його за замовчуванням
```
systemctl restart dhcpd
``` 
```
systemctl enable dhcpd
```

## Firewall

Треба додати такі правила 
```
firewall-cmd --add-service=tftp --permanent
```
```
firewall-cmd --add-service=dhcp --permanent
```
```
firewall-cmd --add-service=http --permanent
```
```
firewall-cmd --reload
```

## GRUB config
В цьому файлі /var/lib/tftpboot/debian-installer/amd64/grub/grub.cfg буде ось такий конфіг. Замініть адресу, де буде розташований preseed file 
```
insmod play
play 960 440 1 0 4 440 1

menuentry 'Install' {
    set background_color=black
    linux    /debian-installer/amd64/vmlinuz auto=true priority=critical  preseed/url=http://192.168.69.8/preseed/preseed.cfg vga=788 --- quiet
    initrd   /debian-installer/amd64/initrd.gz
}


menuentry 'manualInstall' {
    set background_color=black
    linux    /debian-installer/amd64/linux vga=788 --- quiet
    initrd   /debian-installer/amd64/initrd.gz
}
```
Перезапустимо tftp

```
systemctl restart tftp
```
```
systemctl enable tftp 
```
## Налаштування preseed file 

Cкопіюйте цей конфіг у файл /var/www/html/preseed/preseed.cfg. Замініть ip-адресу на вашу , там де буде розташований ваш репозиторій.  
```
d-i debian-installer/locale string en_US 
d-i keyboard-configuration/xkb-keymap select us 
# d-i keyboard-configuration/toggle select No toggling 

### Network configuration 

# from being shown, even if values come from dhcp. 
d-i netcfg/get_hostname string debian12 
d-i netcfg/get_domain string tymchenko.local 

# Disable that annoying WEP key dialog. 
d-i netcfg/wireless_wep string 
# The wacky dhcp hostname that some ISPs use as a password of sorts. 
#d-i netcfg/dhcp_hostname string radish 
d-i debian-installer/allow_unauthenticated boolean true 
d-i mirror/country string manual 
d-i mirror/http/hostname string 192.168.68.140 
d-i mirror/http/directory string /debian 
d-i mirror/http/proxy string
d-i passwd/root-password password root 
d-i passwd/root-password-again password root 
# or encrypted using a crypt(3)  hash.
#d-i passwd/root-password-crypted password [crypt(3) hash]


d-i passwd/user-fullname string yura 
d-i passwd/username string yura 

# Normal user's password, either in clear text 
d-i passwd/user-password password yura 
d-i passwd/user-password-again password yura 
#d-i passwd/user-password-crypted password [crypt(3) hash]
  
d-i clock-setup/utc boolean true 

# You may set this to any valid setting for $TZ; see the contents of 
# /usr/share/zoneinfo/ for valid values. 
d-i time/zone string EU/Kiev 

d-i partman-auto/choose_recipe select atomic  
d-i partman-auto-crypto/erase_disks boolean false  
d-i partman-auto/disk string /dev/sda  
d-i partman-auto-lvm/guided_size string max  
d-i partman-auto/method string lvm  
d-i partman/choose_partition select finish 
d-i partman/confirm boolean true  
d-i partman/confirm_nooverwrite boolean true  
#d-i partman-efi/non_efi_system boolean true  
d-i partman-lvm/confirm boolean true  
d-i partman-lvm/confirm_nooverwrite boolean true  
d-i partman-lvm/device_remove_lvm boolean true 
d-i partman-md/confirm boolean true  
d-i partman-md/device_remove_md boolean true  
d-i partman-partitioning/confirm_write_new_label boolean true 

d-i apt-setup/use_mirror boolean false 

# Очищаємо всі стандартні репозиторії 
d-i apt-setup/services-select multiselect 
d-i apt-setup/local0/repository string 
d-i apt-setup/local1/repository string 
d-i apt-setup/local2/repository string 
d-i apt-setup/restricted boolean false 
d-i apt-setup/contrib boolean false 
d-i apt-setup/non-free boolean false 
d-i apt-setup/non-free-firmware boolean false 

 
#Вимикаємо підключення CD/DVD як джерело 
d-i apt-setup/disable-cdrom boolean true 
d-i apt-setup/disable boolean true 

#Додаємо тільки твій репозиторій у sources.list 
d-i preseed/late_command string  
echo "deb http://192.168.68.140/debian bookworm main non-free-firmware" > /target/etc/apt/sources.list 
tasksel tasksel/first multiselect standard, ssh-server 
d-i finish-install/reboot_in_progress note 
```

Щоб пароль був у зашифрованому вигляді вставте цю команду
```
python3 -c "import crypt;print(crypt.crypt(input('clear-text pw: '), crypt.mksalt(crypt.METHOD_SHA512)))"
```
Перезапустимо веб-сервер. Також при подальших змін в цьому файлі - треба перезапускати цей веб-сервер
```
systemctl restart httpd
```
```
systemctl enable httpd
```

## Створення локального debmirror

Треба розгорнути ще одну машину на debian. За допомогою консольної утиліти debmirror ми відзеркалимо  частину official репи під нашу архітектуру. Також перед цим  треба сюди встановити веб сервер.  

```
apt install debmirror
```
```
apt install httpd
```
```
/usr/bin/debmirror --nosource -m --passive --host=deb.debian.org  --root=debian --method=http --progress --dist=bookworm --ignore-release-gpg --section=main,contrib,non-free,non-free-firmware --arch=amd64 /var/www/html/debian/ 
```
