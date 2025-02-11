# pxe-docs

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

