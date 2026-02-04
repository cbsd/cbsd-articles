# NFSv4/DHCP/PXE jail-based (vnet) сервис через (dnsmasq)

В этой статье описывается вариант запуска и использования контейнера, адаптированного для работы с CBSD с использованием [dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html), который обеспечит (все вместе или опционально):

- DHCP сервис для виртуальных машин и контейнеров, использующих виртуальный стек (например vnet/VIMAGE);
- PXE сервис для запуска бездисковых ОС (как виртуальных машин, так и bare metal);
- HTTP/NFSv4 сервис для раздачи контента бездисковым ОС;

В процессе настройки рассмотрим практические варианты unattended/preseed инсталляций и генерации FreeBSD-based систем, а также автоматическую установку ряда Linux дистрибутивов, реализовав какое-то подобие iVentoy. К примеру,
описанным методом генерируются в полностью автоматическом режиме cloud-образы гостевых систем проекта CBSD;


<center><img src="https://convectix.com/img/dnsmasq_cbsd1.png" width="1024" title="dnsmasq_pxe" alt="dnsmasq_pxe"/></center>


## Запуск и настройка контейнера

Для корректной настройки необходимо заранее выбрать IP адрес (в этом примере используется <strong>172.16.0.101</strong>) маршрутизируемой сети (в этом примере: <strong>172.16.0.0/24</strong>), который вы назначите DHCP/PXE сервису. Также, определите, за какой диапазон адресов DHCP сервер будет отвечать.

:bangbang: | :information_source: Внимание! В статье фигурируют пути файлов и директорий /usr/jails/*. Если путь вашего рабочего каталога CBSD отличается, замените /usr/jails на свой
:---: | :---

:bangbang: | :information_source: Для простоты настройки и во избежание переусложнения, мы не будем рассматривать вариант, когда диапазон статических адресов и DHCP адресов пересекается или совпадает. Например, у нас есть подсать 172.16.0.0/24. Пусть диапазон адресов 172.16.0.2-99 обслуживается DHCP, а 172.16.0.101-243 - будет использован для статических адресов. Хотя вариант указать 172.16.0.1-254 и для статических и для DHCP возможен, он требует задействования механизма CBSD хуков, чтобы гарантировать исключение статических адресов из раздачи DHCP (и в обратную сторону - экспорт списка выданных DHCP сервисом адресов в качестве списка исключений для статических IP).
:---: | :---


a) Запускаем `cbsd initenv-tui` и выставляем настройку `nodeippool` в значение: 172.16.0.102-243 (172.16.0.101 мы отдали под dnsmasq). Это заставит CBSD выдавать автоматически статические адреса для ВМ и контейнеров только в заданном диапазоне, не конфликтуемые с DHCP.

b) Вы можете получить уже готовый образ из репозитория, либо использовать [CBSDFile](https://github.com/cbsd/cbsdfile-recipes/blob/master/jail/dnsmasq/CBSDfile) из коллекции рецептов (потребуется установить `git`) для генерации контейнера с нуля. 

<details>
  <summary>Вариант 1: готовый образ dnsmasq</summary>

1) Install 'cbsd' package:
```
pkg install -y cbsd
```
</details>

<details>
  <summary>Вариант 2: генерация контейнера dnsmasq</summary>

```
git clone https://github.com/cbsd/cbsdfile-recipes.git /tmp/cbsdfile-recipes
cd /tmp/cbsdfile-recipes/jail/dnsmasq
```
Для Запускаем контейнер с IP адресом <strong>172.16.0.101</strong>:
```
cbsd up ip4_addr=172.16.0.101
```
</details>

c) Запускаем реконфигуратор сервиса:
```
cbsd jconfig jname=dnsmasq net
```
или в TUI варианте:
```
cbsd jconfig jname=dnsmasq net-tui
```

Пример ответов:
```
Trusted NFSv3/v4 network: 0
Please set DHCP-range: 172.16.0.2,172.16.0.99,12h
Please set GW4: 172.16.0.1
NS1: 172.16.0.1
NS2: 0
```

Для неинтерактивного реконфигурирования используйте env или непосредственно в CBSDFile (см. комментированный пример в CBSDfile):
```
	export H_NFSNET="0" \
		H_IP4RANGE="172.16.0.2,172.16.0.100,12h" \
		H_GW4="172.16.0.1" \
		H_NS1="172.16.0.1" \
		H_NS2="0"
```

- <strong>H_NFSNET</strong> или <strong>NFSv3/v4 network</strong>: IP сеть в формате CIDR (net/prefix), с которой дозволено монтировать NFS ресурсы (PXE/diskless) (корректная IP/сеть, либо '0' если не включать NFS сервис ). Ниже рассмотрим вариант использования этого сервиса;
- <strong>H_IP4RANGE</strong> или <strong>DHCP-range</strong>: диапазон IP адресов, которые обслуживает (будет выдавать) DHCP сервер в формате <start>,<end>,<ttl> , например: 10.0.0.5,10.0.0.10,12h - будут выдаваться ардеса с 10.0.0.5 - 10.0.0.10 сроком на 12 часов;
- <strong>H_GW4</strong> или <strong>GW4</strong>: gateway адрес, который будет отдавать DHCP сервер в качестве маршрута по-умолчанию ( корректный IP адрес шлюза );
- <strong>H_NS1</strong> или <strong>NS1</strong>: nameserver1, который будет отдавать DHCP сервер в качестве авто-настройки (корректный IP адрес DNS сервера, либо '0' если не слать );
- <strong>H_NS2</strong> или <strong>NS2</strong>: nameserver2, который будет отдавать DHCP сервер в качестве авто-настройки (корректный IP адрес DNS сервера, либо '0' если не слать );

После запуска и инициализации контейнера, наш рабочий IPXE файл - ~cbsd/jails/dnsmasq/usr/local/srv/pxe/main.ipxe (если идти от файловой системы хостера) или /usr/local/srv/pxe/main.ipxe относительно корня FS непосредственно внутри контейнера.

### Дамп записей leases файла

Доступно через HTTP запрос на адрес контейнера, например: <strong>http://172.16.0.101</strong>

<center><img src="https://convectix.com/img/dnsmasq_cbsd2.png" width="1024" title="dnsmasq_pxe" alt="dnsmasq_pxe"/></center>

Или через CLI:
```
cbsd jexec jname=dnsmasq dnsmasq_records
```

<center><img src="https://convectix.com/img/dnsmasq_cbsd3.png" width="1024" title="dnsmasq_pxe" alt="dnsmasq_pxe"/></center>

## PXE/NFS(TFTP) сервис для Diskless систем и Unattended инсталляций.

Рассмотрим несколько вариантов добавления PXE/diskless ОС.

:bangbang: | :information_source: В примерах по генерарации ISO/PXE структуры используется CBSD скрипт `jail2iso`. Если вы пользуетесь альтернативными механизмами генерации mfsbsd - пропустите строчки с 'jail2iso'(генерируя своим способом), однако остальной алгоритм/записи/настройки/советы скорее всего, будут полезны и подойдут всем.
:---: | :---

--

### FreeBSD PXE: unattended/preseed/auto install (с использованием официального ISO образа)

<details>
  <summary>Step-by-step HOW-TO</summary>

На момент написания статьи (<strong>2026</strong>), [bsdinstall(8)](https://man.freebsd.org/cgi/man.cgi?query=bsdinstall) поддерживает преднастроенные параметры (см. секцию <strong>SCRIPTING</strong> ) через файл <strong>/etc/installerconfig</strong>, однако автору неизвестно, каким образом через kenv/loader можно передать источник preseed конфигурации (например, URL внешнего HTTP сервиса) и при этом не прибегать к модификации официального ISO образа. Примечание: в рамках [GSoC был проект](https://wiki.freebsd.org/SummerOfCode2014/FreeBSD_PXE_preseed) по улучшению поддержки preseed-инсталляции, но наработки не попали в апстрим.

В связи с этим, мы перегенерируем ISO образ, чтобы разместить preseed файл в структуре данных самого образа и заодно конвертируем официальный образ в MFSBSD режим (т.е, он будет полностью загружен в оперативную память - в этом случае, загруженное Live-окружение не будет зависеть от потенциальных сбоев сети или закрытия браузера, через который вы презентуете ISO образ в KVM/IPMI/Redfish).

В примере мы работаем с <strong>FreeBSD 15.0-RELEASE</strong>.

1) Получаем официальный образ через утилиту `fetch` или используя профиль CBSD:

```
fetch -o /tmp/FreeBSD-15.0-RELEASE-amd64-disc1.iso.xz https://download.freebsd.org/releases/ISO-IMAGES/15.0/FreeBSD-15.0-RELEASE-amd64-disc1.iso.xz
```
или (+ сканирование и поиск быстрых зеркал):
```
cbsd fetch_iso name=FreeBSD-15.0-x86_64 fastscan=1 keepname=1 dstdir=/tmp
```
и распаковываем:
```
xz -d /tmp/FreeBSD-15.0-RELEASE-amd64-disc1.iso.xz
```

Результат - наличие ISO образа /tmp/FreeBSD-15.0-RELEASE-amd64-disc1.iso, проверка:
```
file -s /tmp/FreeBSD-15.0-RELEASE-amd64-disc1.iso
```
> /tmp/FreeBSD-15.0-RELEASE-amd64-disc1.iso: ISO 9660 CD-ROM filesystem data (DOS/MBR boot sector) '15_0_RELEASE_AMD64_CD' (bootable)

2) Смонтируем содержимое ISO образа в каталог /mnt (mdconfig + mount):
```
mount_cd9660 /dev/`mdconfig -a -t vnode -f /tmp/FreeBSD-15.0-RELEASE-amd64-disc1.iso` /mnt
```

если образ корректный, мы можем вывести листинг директорий образа:
```
ls -1 /mnt/
```
```
.rr_moved
COPYRIGHT
bin
boot
dev
etc
lib
libexec
media
mnt
net
proc
rescue
root
sbin
tmp
usr
var
```

3) CBSD позволяет сгенерировать ISO/memstick/MFSBSD образ и/или PXE структуру из `jail` - воспользуемся этим механизмом. Для этого, создадим пустой `jail` и скопируем в него содержимое ISO образа из каталога /mnt. По желанию, вы можете сразу назначить статические IP адреса и активировать SSH доступ (см. `cbsd jail2iso --help`), но в этом примере мы будем использовать настройку DHCP (не зря же мы запускали dnsmasq контейнер!).

a)
```
cbsd jcreate jname=myjail baserw=1 ver=empty pkg_bootstrap=0 floatresolv=0 applytpl=0 etcupdate_init=0
```

b)
копируем /mnt/* в директорию контейнера:
```
cp -a /mnt/* ~cbsd/jails-data/myjail-data/
umount /mnt
```

удаляем записи в оригинальном fstab, поскольку они пробуют использовать CD9660/ISO:
```
truncate -s0 ~cbsd/jails-data/myjail-data/etc/fstab
```

Удалим ненужную нам символическую ссылку /etc/resolv.conf и добавим свой NS для корректного разрешения имен (по умолчанию, ISO имеет такую ссылку):
```
lrw-r--r--   1 root wheel       31 Nov 28 08:18 resolv.conf -> /tmp/bsdinstall_etc/resolv.conf
```

```
rm -f ~cbsd/jails-data/myjail-data/etc/resolv.conf
echo "nameserver 8.8.8.8" > ~cbsd/jails-data/myjail-data/etc/resolv.conf
```

Добавим немного SCRIPTING/preseed/unattended для инсталлятора, ~cbsd/jails-data/myjail-data/etc/installerconfig:
```
#PARTITIONS=DEFAULT
#export DISK=$(sysctl -n kern.disks | awk '{print $1}')
export DISK="vtbd0"
export PARTITIONS="${DISK} GPT { 512K freebsd-boot, auto freebsd-ufs / }"

export DISTRIBUTIONS="kernel.txz base.txz"
export BSDINSTALL_SKIP_FIRMWARE=true
export BSDINSTALL_SKIP_HARDENING=true
export BSDINSTALL_SKIP_KEYMAP=true
export BSDINSTALL_SKIP_MANUAL=true
export BSDINSTALL_SKIP_SERVICES=true
export BSDINSTALL_SKIP_TIME=true
export BSDINSTALL_SKIP_USERS=true
export BSDINSTALL_SKIP_FINALCONFIG=true

# doesn't work
#export ROOTPASS_ENC='$6$hIzCCQNYT7wL0bpY$78hF0uQQRg45wzybpwx/lXCaX5mcKv2FW40cBG4xTQ28hAPv4ptcpz9WthWLGGgIi7Gi.Gd2qBjgxFbUMCxa10'
#export ROOTPASS_PLAIN="test"

export BSDINSTALL_DISTSITE="http://ftp.freebsd.org/pub/FreeBSD/releases/amd64/amd64/15.0-RELEASE"
export NONINTERACTIVE=yes


#!/bin/sh
sysrc ifconfig_DEFAULT=DHCP sshd_enable=YES sshd_flags="-oUseDNS=no -oPermitRootLogin=yes"

```


c) Генерируем MFSBSD. Параметр `mfs_struct_only=1` означает, что нам ненужно генерировать образ - только MFSBSD структуру файлов для PXE сервиса.

убедитесь, что модуль roothack установлен (требуется для MFSBSD):
```
cbsd module mode=install roothack
make -C /usr/local/cbsd/modules/roothack.d
```

```
cbsd jail2iso jname=myjail dstdir=/usr/jails/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-15.0/ media=mfs freesize=2m ver=15.0 efi=1 mfsbsd_leave_kernel_dir=1 mfs_struct_only=1 kernel_dir=/usr/jails/jails-data/myjail-data/boot/kernel mfsbsd_ip_addr=REALDHCP mfsbsd_interface=auto
```

4) Добавляем пункт меню и релевантные записи в IPXE файл ( ~cbsd/jails/dnsmasq/usr/local/srv/pxe/main.ipxe ):

```
item --key 1 FreeBSD-15.0         FreeBSD 15.0


:FreeBSD-15.0
set img FreeBSD-15.0
chain /${img}/boot/loader.efi dhcp.root-path=tftp:/${img} || goto failed
boot || goto failed
```
Дополнительно (но не обязательно), имеет смысл сжать ядро и модули - loader сначала пробует использовать *.gz при загрузке файлов. Это значительно ускорит загрузку:
```
gzip -f --best /usr/jails/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-15.0/boot/kernel/kernel
gzip -f --best /usr/jails/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-15.0/boot/kernel/*.ko
```

</details>

### FreeBSD PXE: генерация microvm для гипервизора bhyve

<details>
  <summary>Step-by-step HOW-TO</summary>

:information_source: Примечание: этот пример служит частным вариантом https://github.com/cbsd/microbhyve, но с использованием PXE (отличаются лишь несколько строк конфига ядра и индексных файлов в целях минимализма).

Предыдущий пример использовал официальный ISO образ (который, все же, потребовалось модифицировать). В этом примере мы создадим виртуальную машину полностью из CBSD jail, минимизировав ее настолько, насколько это возможно, если не прибегать к различным хакам. Данный пример можно считать базой для создания любых минималистических прошивок на базе FreeBSD.

1)
Создадим пустой контейнер:
```
cbsd jcreate jname=micro1 baserw=1 ver=empty pkg_bootstrap=0 floatresolv=0 applytpl=0 etcupdate_init=0
```

2) Наполним структуру контейнера по индексному файлу, который содержит минимальный список файлов (поставляется с CBSD в качестве примера минимализма):


:bangbang: | :information_source: Внимание! Мы наполняем файлы по индексу, беря их из вашей хост системы (basedir=/) файлы (например, файлы паролей будут взяты из вашего хоста). Вы можете взять в качестве источника альтернативные файлы, например ~cbsd/basejail/bases_XXX_YYY_VV/ каталоги 
:---: | :---
```
cbsd copy-binlib basedir=/ chaselibs=1 dstdir=/usr/jails/jails-data/micro1-data filelist=/usr/local/cbsd/share/FreeBSD-microbhyve-pxe.txt.xz
```
3) Кастомизация. Поскольку мы создаем действительно минимальный образ, в нем нет конфигуратора сети и dhclient, поэтому сконфигурируем сеть через /etc/rc.local - поменяйте IP адрес и шлюз на ваши рабочие (вместо 172.16.0.99 и 172.16.0.1)!
```
cbsd sysrc jname=micro1 \
        sshd_flags="-oUseDNS=no -oPermitRootLogin=yes" \
        root_rw_mount="YES" \
        sshd_enable=YES \
        rc_startmsgs="YES" 

cat > /usr/jails/jails-data/micro1-data/etc/rc.local <<EOF
/sbin/ifconfig vtnet0 inet 172.16.0.99/24 up
/sbin/route add default 172.16.0.1
EOF

truncate -s0 /usr/jails/jails-data/micro1-data/etc/fstab
cp -a /etc/ssh /usr/jails/jails-data/micro1-data/etc/
cp -a /etc/gss /usr/jails/jails-data/micro1-data/etc/
cp -a /etc/pam.d /usr/jails/jails-data/micro1-data/etc/
mkdir -p /usr/jails/jails-data/micro1-data/var/empty /usr/jails/jails-data/micro1-data/var/log /usr/jails/jails-data/micro1-data/var/run /usr/jails/jails-data/micro1-data/root /usr/jails/jails-data/micro1-data/dev
chmod 0700 /usr/jails/jails-data/micro1-data/var/empty
pw -R /usr/jails/jails-data/micro1-data usermod root -s /bin/sh

# strip debug info via strip(1)
find /usr/jails/jails-data/micro1-data/ -type f -perm +111 -exec strip {} \;
```
4) Собираем ядро BHYVE-PXE, минимальный рабочий конфиг ядра (см. ~cbsd/etc/defaults/FreeBSD-kernel-BHYVE-PXE-amd64-15.0 ) для bhyve+PXE (поставляется с CBSD в качестве примера минимализма):
```
cbsd srcup
cbsd kernel name=BHYVE-PXE
```
5) убедитесь, что модуль roothack установлен (требуется для MFSBSD):
```
cbsd module mode=install roothack
make -C /usr/local/cbsd/modules/roothack.d
```
генерируем PXE иерархию:
```
cbsd jail2iso jname=micro1 dstdir=/usr/jails/jails/dnsmasq/usr/local/srv/pxe/MicroBSD-15.0/ media=mfs freesize=2m ver=15.0 efi=1 mfsbsd_leave_kernel_dir=1 mfs_struct_only=1 name=BHYVE-PXE
```
hint: если вы хотите сгенерировать загружаемый образ (в каталог /tmp ) виртуальной машины:
```
cbsd jail2iso jname=micro1 dstdir=/tmp/ media=mfs freesize=2m ver=15.0 efi=1 mfsbsd_leave_kernel_dir=1 mfs_struct_only=1 name=BHYVE-PXE
```

отредактируем /usr/jails/jails/dnsmasq/usr/local/srv/pxe/MicroBSD-15.0/boot/loader.conf в следующий вид:
```
if_vtnet_load="YES"

kernels_autodetect="NO"
kern.msgbuf_show_timestamp=1
hw.usb.no_boot_wait=1
hw.usb.no_shutdown_wait="1"
hw.usb.no_pf="1"
hw.memtest.tests=0
boot_serial=NO
boot_multicons=NO
console=efi

autoboot_delay="-1"

net.inet.ip.fw.default_to_accept="1"
vfs.mountroot.timeout=3
# MFS
mfs_load="YES"
mfs_name="/mfsroot"
mfs_type="mfs_root"
vfs.root.mountfrom="ufs:/dev/md0"
vfs.root_mount_always_wait=1
```

6) Добавляем пункт меню и релевантные записи в IPXE файл ( ~cbsd/jails/dnsmasq/usr/local/srv/pxe/main.ipxe ):

```
item --key 2 MicroBSD-15.0         MicroBSD 15.0


:MicroBSD-15.0
set img MicroBSD-15.0
chain /${img}/boot/loader.efi dhcp.root-path=tftp:/${img} || goto failed
boot || goto failed
```

Дополнительно (но не обязательно), имеет смысл сжать ядро и модули - loader сначала пробует использовать *.gz при загрузке файлов. Это значительно ускорит загрузку:
```
gzip -f --best /usr/jails/jails/dnsmasq/usr/local/srv/pxe/Micro-15.0/boot/kernel/kernel
gzip -f --best /usr/jails/jails/dnsmasq/usr/local/srv/pxe/Micro-15.0/boot/kernel/*.ko
```

</details>

### FreeBSD PXE: бездисковая рабочая станция с использованием NFSv4

<details>
  <summary>Step-by-step HOW-TO</summary>

На момент написания данной статьи, официальный FreeBSD loader имеет крайне ограниченную поддержку сетевого стека (см. /usr/src/stand/libsa/* ) и NFS в частности, ( см: /usr/src/stand/libsa/nfs.c, /usr/src/stand/libsa/nfsv2.h )

В связи с тем, что современные NFS сервера не рекомендуют и избавляются от UDP транспорта, а NFSD развернутый в jail vnet и вовсе [не поддерживает UDP](https://people.freebsd.org/~rmacklem/nfsd-vnet-prison-setup.txt):
> At this time, it is possible to enable NFSv3 and/or NFSv4 over TCP, but not NFSv3 over UDP

Мы вновь откажемся от использования ванильного образа FreeBSD и попытками использовать NFS с параметрами в loader.conf (возможно, когда-нибудь это заработает) вида:
```
#vfs.root_mount_always_wait=1
vfs.root.mountfrom="172.16.0.101:/usr/local/srv/pxe/FreeBSD-NFS-15.0-rw"
vfs.root.mountfrom.options=rw

# IP-адрес клиента и маска
boot.netif.ip="172.16.0.X"
boot.netif.netmask="255.255.255.0"
boot.netif.gateway="172.16.0.1"
#boot.netif.hwaddr="XX:XX:XX:XX:XX:XX" # MAC-адрес вашего интерфейса
#boot.netif.name="vtnet0"

# Данные NFS-сервера
boot.nfsroot.server="172.16.0.101"
boot.nfsroot.path="/usr/local/srv/pxe/FreeBSD-NFS-15.0-rw"
```

И вновь будем использовать MFSBSD прослойку, которое загрузит систему с полноценным сетевым стеком и поддержкой NFSv4, после чего выполним pivot/reroot на полноценное окружение (которое может быть зашифрованным).

1) Подготовим каталог для полной системы (предполагаем, что в /usr/src имеем исходный код и была выполнена процедура сборки мира через `make -C /usr/src buildworld`. Вы можете предпочесть метод установки базы через base-in-pkg посредством `pkg`, либо `bsdinstall jail`, либо просто скопировать существующую базу от CBSD из каталога ~cbsd/basejail/base_XXX_YYY_VER ):
```
mkdir -p /usr/jails/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-NFS-15.0-rw
make -C /usr/src installworld distribution DESTDIR="/usr/jails/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-NFS-15.0-rw"
```

Настроим внутри NFS окружения /etc/rc.conf, чтобы загрузка системы происходила корректно:
```
touch ~cbsd/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-NFS-15.0-rw/etc/rc.conf
sysrc -qf ~cbsd/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-NFS-15.0-rw/etc/rc.conf nfs_client_enable="YES" nfs_access_cache="60" root_rw_mount="NO" varmfs="NO" tmpmfs="NO" ifconfig_DEFAULT="DHCP"
```

Настроим внутри NFS окружения /etc/fstab, чтобы система монтировала NFS и tmpfs, создайте файл  ~cbsd/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-NFS-15.0-rw/etc/fstab с содержимым:


```
172.16.0.101:/FreeBSD-NFS-15.0-rw	/	nfs	rw,noinet6,nfsv4,nolockd,tcp,rw,rsize=1048576,wsize=1048576,noatime	0 0
tmpfs	/tmp		tmpfs	rw	0 0
tmpfs	/var/run	tmpfs	rw	0 0

```

2) Убедимся, что /etc/exports внутри dnsmasq контейнера содержил релевантные записи для экспорта нужных каталогов: ~cbsd/jails/dnsmasq/etc/exports
```
/usr/local/srv/pxe -alldirs -network 172.16.0.0/24 -maproot=root -sec=sys
V4: /usr/local/srv/pxe -network 172.16.0.0/24
```

:bangbang: | :information_source: Если dnsmasq расположен на ZFS, возможно вам потребуется выставить флаг sharenfs на данном датасете: `zfs set sharenfs=on <dataset>`

Убедимся, что сервис перезапускается:
```
cbsd jexec jname=dnsmasq /etc/rc.d/mountd restart
```

3) В остальном, наш MFS образ будет повторять вариант с BHYVE-PXE, за исключением того что в конфигурации ядра нам потребуется поддержка NFS и в /etc/rc.d/ мы разместим скрипт, монтирующий NFS и выполняющий pivot_root/nfs функцию:

Собираем ядро BHYVE-PXENFS, минимальный рабочий конфиг ядра (см. ~cbsd/etc/defaults/FreeBSD-kernel--BHYVE-PXENFS-amd64-15.0 ) (поставляется с CBSD в качестве примера минимализма):
```
cbsd srcup
cbsd kernel name=BHYVE-PXENFS
```

4) Создадим пустой контейнер:
```
cbsd jcreate jname=micro1 baserw=1 ver=empty pkg_bootstrap=0 floatresolv=0 applytpl=0 etcupdate_init=0
```

5) Наполним структуру контейнера по индексному файлу, который содержит минимальный список файлов (поставляется с CBSD в качестве примера минимализма):


:bangbang: | :information_source: Внимание! Мы наполняем файлы по индексу, беря их из вашей хост системы (basedir=/) файлы (например, файлы паролей будут взяты из вашего хоста). Вы можете взять в качестве источника альтернативные файлы, например ~cbsd/basejail/bases_XXX_YYY_VV/ каталоги 
:---: | :---
```
cbsd copy-binlib basedir=/ chaselibs=1 dstdir=/usr/jails/jails-data/micro1-data filelist=/usr/local/cbsd/share/FreeBSD-microbhyve-pxenfs.txt.xz
```
6) Кастомизация. Поскольку мы создаем действительно минимальный образ, в нем нет конфигуратора сети и dhclient, поэтому сконфигурируем сеть через /etc/rc.local - поменяйте IP адрес и шлюз на ваши рабочие (вместо 172.16.0.99 и 172.16.0.1)!
```
cbsd sysrc jname=micro1 \
	root_rw_mount="NO"

truncate -s0 /usr/jails/jails-data/micro1-data/etc/fstab
cp -a /etc/ssh /usr/jails/jails-data/micro1-data/etc/
cp -a /etc/gss /usr/jails/jails-data/micro1-data/etc/
cp -a /etc/pam.d /usr/jails/jails-data/micro1-data/etc/
mkdir -p /usr/jails/jails-data/micro1-data/var/empty /usr/jails/jails-data/micro1-data/var/log /usr/jails/jails-data/micro1-data/var/run /usr/jails/jails-data/micro1-data/root /usr/jails/jails-data/micro1-data/dev
chmod 0700 /usr/jails/jails-data/micro1-data/var/empty
pw -R /usr/jails/jails-data/micro1-data usermod root -s /bin/sh
find /usr/jails/jails-data/micro1-data/ -type f -perm +111 -exec strip {} \;
```

Создайте в ~cbsd/jails-data/micro1-data/etc/rc.local файл с содержимым:

```
/sbin/ifconfig vtnet0 inet 172.16.0.99/24 up
/sbin/route add default 172.16.0.1

getdesk_nfs=$( /bin/kenv getdesk_nfs )
getdesk_nfs_opts=$( /bin/kenv getdesk_nfs_opts )

if [ -z "${getdesk_nfs}" ]; then
	echo "error: no such getdesk_nfs kenv, checkout PXE server settings"
	exec /bin/sh
fi
if [ -z "${getdesk_nfs_opts}" ]; then
	echo "error: no such getdesk_nfs_opts kenv, checkout PXE server settings"
	exec /bin/sh
fi

# mount test:
[ ! -d /mnt ] && mkdir /mnt

for i in 1 2 3 4 5 6; do
	echo "mount [${i}/6]: ${getdesk_nfs} ..."
	set -o xtrace
	/bin/timeout 3 /sbin/mount -o ${getdesk_nfs_opts} ${getdesk_nfs} /mnt
	_ret=$?
	set +o xtrace
	if [ ${_ret} -eq 0 ]; then
		echo "   * mount ok"
		break
	fi
	sleep 1
done

if [ ${_ret} -ne 0 ]; then
	echo "mount failed: /sbin/mount -o ${getdesk_nfs_opts} ${getdesk_nfs} /mnt"
	exec /bin/sh
fi

/sbin/umount -f /mnt

echo "reroot -> nfs:${getdesk_nfs} ..."
/bin/kenv vfs.root.mountfrom="nfs:${getdesk_nfs}"
/bin/kenv vfs.root.mountfrom.options="${getdesk_nfs_opts}"
/sbin/reboot -r
```

7) убедитесь, что модуль roothack установлен (требуется для MFSBSD):
```
cbsd module mode=install roothack
make -C /usr/local/cbsd/modules/roothack.d
```
8) генерируем PXE иерархию:
```
cbsd jail2iso jname=micro1 dstdir=/usr/jails/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-NFS-15.0/ media=mfs freesize=2m ver=15.0 efi=1 mfsbsd_leave_kernel_dir=1 mfs_struct_only=1 name=BHYVE-PXENFS
```
hint: если вы хотите сгенерировать загружаемый образ (в каталог /tmp ) виртуальной машины:
```
cbsd jail2iso jname=micro1 dstdir=/tmp/ media=mfs freesize=2m ver=15.0 efi=1 mfsbsd_leave_kernel_dir=1 mfs_struct_only=1 name=BHYVE-PXENFS
```

добавьте в ~cbsd/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-NFS-15.0/boot/loader.conf две строчки для передачи информации по монтированию NFS:
```
# смотрите содержимое /etc/rc.local
getdesk_nfs="172.16.0.101:/FreeBSD-NFS-15.0-rw"
getdesk_nfs_opts="nfsv4,tcp,rw,rsize=1048576,wsize=1048576"
```

9) Добавляем пункт меню и релевантные записи в IPXE файл ( ~cbsd/jails/dnsmasq/usr/local/srv/pxe/main.ipxe ):

```
item --key 2 FreeBSD-NFS-15.0         FreeBSD-NFS-15.0


:FreeBSD-NFS-15.0
set img FreeBSD-NFS-15.0
chain /${img}/boot/loader.efi dhcp.root-path=tftp:/${img} || goto failed
boot || goto failed
```

Дополнительно (но не обязательно), имеет смысл сжать ядро и модули - loader сначала пробует использовать *.gz при загрузке файлов. Это значительно ускорит загрузку:
```
gzip -f --best /usr/jails/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-NFS-15.0/boot/kernel/kernel
gzip -f --best /usr/jails/jails/dnsmasq/usr/local/srv/pxe/FreeBSD-NFS-15.0/boot/kernel/*.ko
```

</details>

### Debian PXE: unattended/preseed/auto install (с использованием официального ISO образа)

<details>
  <summary>Step-by-step HOW-TO</summary>

Пример автоматической инсталляции Debian с использованием preseed скрипта и PXE сервиса в контейнере dnsmasq.

Как и в случае с FreeBSD ISO, мы здесь не сможем обойтись лишь официальным ISO образом, поскольку в него не входят совместимые с PXE файлы ( initrd/vmlinuz ) - их скачаем отдельно и положим в структуру PXE. Начнем с официального диска. 

В примере мы работаем с <strong>Debian 13.3.0 ( trixie )</strong>.

1) Получаем официальный образ через утилиту `fetch` или используя профиль CBSD:

```
fetch -o /tmp/debian-13.3.0-amd64-DVD-1.iso https://cdimage.debian.org/debian-cd/current/amd64/iso-dvd/debian-13.3.0-amd64-DVD-1.iso
```
или (+ сканирование и поиск быстрых зеркал):
```
cbsd fetch_iso name=Debian-13-x86_64 fastscan=1 keepname=1 dstdir=/tmp
```
Результат - наличие ISO образа /tmp/debian-13.3.0-amd64-DVD-1.iso, проверка:
```
file -s /tmp/debian-13.3.0-amd64-DVD-1.iso
```

> /tmp/debian-13.3.0-amd64-DVD-1.iso: ISO 9660 CD-ROM filesystem data (DOS/MBR boot sector) 'Debian 13.3.0 amd64 1' (bootable)

2) Смонтируем содержимое ISO образа в каталог /mnt (mdconfig + mount):
```
mount_cd9660 /dev/`mdconfig -a -t vnode -f /tmp/debian-13.3.0-amd64-DVD-1.iso` /mnt
```

если образ корректный, мы можем вывести листинг директорий образа:
```
ls -1 /mnt/
```
```
.disk
EFI
README.html
README.mirrors.html
README.mirrors.txt
README.source
README.txt
boot
css
debian
dists
doc
firmware
install
install.amd
isolinux
md5sum.txt
pics
pool
```

2) Наполняем структуру PXE

a)
копируем /mnt/* в директорию контейнера:
```
mkdir -p ~cbsd/jails/dnsmasq/usr/local/srv/pxe/Debian-13.3.0
cp -a /mnt/* ~cbsd/jails/dnsmasq/usr/local/srv/pxe/Debian-13.3.0
umount /mnt
```

b) заменяем vmlinux/initrd с поддержкой PXE/NFS:
```
cd /tmp
fetch -o /tmp/netboot.tar.gz http://ftp.debian.org/debian/dists/trixie/main/installer-amd64/current/images/netboot/netboot.tar.gz
tar xfz netboot.tar.gz
mv /tmp/debian-installer ~cbsd/jails/dnsmasq/usr/local/srv/pxe/Debian-13.3.0/
```

c) Создадим d-i (Debian-installer) preseed файл, на который ссылается наша загрузка:
```
mkdir -p /usr/jails/jails/dnsmasq/usr/local/srv/debian13
```

<details>
  <summary>Содержимое /usr/jails/jails/dnsmasq/usr/local/srv/debian13/auto.cfg:</summary>

```
d-i mirror/http/hostname string 172.16.0.101
d-i mirror/http/directory string /pxe/Debian-13.3.0
d-i debian-installer/allow_unauthenticated boolean true

d-i console-setup/ask_detect boolean false
d-i console-setup/layoutcode string us
d-i keyboard-configuration/xkb-keymap select us
d-i clock-setup/utc boolean true

d-i time/zone string Europe/Moscow
d-i clock-setup/ntp boolean true

d-i localechooser/supported-locales multiselect en_US.UTF-8, ru_RU.UTF-8

d-i netcfg/choose_interface select auto
d-i netcfg/get_hostname string space-controller
d-i netcfg/get_domain string local
d-i debian-installer/locale string en_US.UTF-8
d-i time/zone string RU/Moscow

d-i passwd/make-user boolean false
d-i passwd/root-login boolean true
d-i passwd/root-password password cbsd
d-i passwd/root-password-again password cbsd
d-i user-setup/allow-password-weak boolean true

d-i apt-setup/cdrom/set-first boolean false
d-i apt-setup/cdrom/set-next boolean false
d-i apt-setup/cdrom/set-failed boolean false

d-i partman/early_command string debconf-set partman-auto/disk "$(list-devices disk | grep -v nvme[0-9]c[0-9] | head -n1)"
d-i partman-auto/method string lvm
d-i partman-auto/choose_recipe select atomic
d-i partman-lvm/device_remove_lvm boolean true
d-i partman-md/device_remove_md boolean true
d-i partman-lvm/confirm boolean true
d-i partman-lvm/confirm_nooverwrite boolean true
d-i partman-partitioning/confirm_write_new_label boolean true
d-i partman/choose_partition select finish
d-i partman/confirm boolean true
d-i partman/mount_style select uuid
d-i partman/confirm_nooverwrite boolean true

# Keep that one set to true so we end up with a UEFI enabled
# system. If set to false, /var/lib/partman/uefi_ignore will be touched
d-i partman-efi/non_efi_system boolean true

# enforce usage to GPT
d-i partman-basicfilesystems/no_swap boolean false
d-i partman-basicfilesystems/choose_label string gpt
d-i partman-basicfilesystems/default_label string gpt
d-i partman-partitioning/choose_label string gpt
d-i partman-partitioning/default_label string gpt
d-i partman/choose_label string gpt
d-i partman/default_label string gpt

d-i partman/mount_style select uuid
d-i partman-md/confirm boolean true

d-i apt-setup/use_mirror boolean false
d-i apt-setup/services-select multiselect
apt-cdrom-setup apt-setup/cdrom/set-first boolean false
popularity-contest popularity-contest/participate boolean false
d-i grub-installer/only_debian boolean true
d-i grub-installer/with_other_os boolean true
tasksel tasksel/first multiselect standard

apt-cacher-ng apt-cacher-ng/tunnelenable boolean false

# pkg
d-i pkgsel/include string openssh-server

d-i finish-install/reboot_in_progress note

###
#d-i partman-auto-lvm/guided_size string 50%
d-i partman-auto-lvm/guided_size string max
d-i partman-auto-lvm/new_vg_name string sys_vg01

#d-i partman-auto/choose_recipe select boot-root
#d-i partman-auto/choose_recipe select atomic
d-i partman-auto/expert_recipe string                           \
        boot-root ::                                            \
                256 8 1024 free                                 \
                $iflabel{ gpt }                                 \
                $reusemethod{ }                                 \
                method{ efi }                                   \
                format{ }                                       \
                .                                               \
                512 16 512 ext4                                 \
                $primary{ } $bootable{ }                        \
                method{ format } format{ }                      \
                use_filesystem{ } filesystem{ ext4 }            \
                mountpoint{ /boot }                             \
                .                                               \
                20000 64 -1 ext4                                \
                $lvmok{ }                                       \
                in_vg { sys_vg01 }                              \
                lv_name{ root }                                 \
                method{ format } format{ }                      \
                use_filesystem{ } filesystem{ ext4 }            \
                mountpoint{ / }                                 \
                .

```

</details>
</details>

### Rocky/CentOS PXE: unattended/preseed/auto install (с использованием официального ISO образа)

<details>
  <summary>Step-by-step HOW-TO</summary>

Пример автоматической инсталляции Rocky с использованием preseed скрипта и PXE сервиса в контейнере dnsmasq.

//Как и в случае с FreeBSD ISO, мы здесь не сможем обойтись лишь официальным ISO образом, поскольку в него не входят совместимые с PXE файлы ( initrd/vmlinuz ) - их скачаем отдельно и положим в структуру PXE. Начнем с официального диска. 

В примере мы работаем с <strong>Rocky 10.1</strong>.

1) Получаем официальный образ через утилиту `fetch` или используя профиль CBSD:

```
fetch -o /tmp/Rocky-10.1-x86_64-dvd1.iso https://download.rockylinux.org/pub/rocky/10/isos/x86_64/Rocky-10.1-x86_64-dvd1.iso
```
или (+ сканирование и поиск быстрых зеркал):
```
cbsd fetch_iso name=Rocky-10-x86_64 fastscan=1 keepname=1 dstdir=/tmp
```
Результат - наличие ISO образа /tmp/Rocky-10.1-x86_64-dvd1.iso, проверка:
```
file -s /tmp/Rocky-10.1-x86_64-dvd1.iso
```

> /tmp/Rocky-10.1-x86_64-dvd1.iso: ISO 9660 CD-ROM filesystem data (DOS/MBR boot sector) 'Rocky-10-1-x86_64-dvd' (bootable)

2) Смонтируем содержимое ISO образа в каталог /mnt (mdconfig + mount):
```
mount_cd9660 /dev/`mdconfig -a -t vnode -f /tmp/Rocky-10.1-x86_64-dvd1.iso` /mnt
```

если образ корректный, мы можем вывести листинг директорий образа:
```
ls -1 /mnt/
```
```
.discinfo
.treeinfo
AppStream
BaseOS
COMMUNITY-CHARTER
EFI
EULA
LICENSE
RPM-GPG-KEY-Rocky-10
RPM-GPG-KEY-Rocky-10-Testing
boot
extra_files.json
images
media.repo
```

2) Наполняем структуру PXE

a)
копируем /mnt/* в директорию контейнера:
```
mkdir -p ~cbsd/jails/dnsmasq/usr/local/srv/pxe/Rocky-10.1
cp -a /mnt/* ~cbsd/jails/dnsmasq/usr/local/srv/pxe/Rocky-10.1
umount /mnt
```

b) Создадим kickstart preseed файл, на который ссылается наша загрузка:
```
mkdir -p /usr/jails/jails/dnsmasq/usr/local/srv/rocky10
```

<details>
  <summary>Содержимое /usr/jails/jails/dnsmasq/usr/local/srv/rocky10/auto.ks:</summary>

```
d-i mirror/http/hostname string 172.16.0.101
d-i mirror/http/directory string /pxe/Debian-13.3.0
d-i debian-installer/allow_unauthenticated boolean true

d-i console-setup/ask_detect boolean false
d-i console-setup/layoutcode string us
d-i keyboard-configuration/xkb-keymap select us
d-i clock-setup/utc boolean true

d-i time/zone string Europe/Moscow
d-i clock-setup/ntp boolean true

d-i localechooser/supported-locales multiselect en_US.UTF-8, ru_RU.UTF-8

d-i netcfg/choose_interface select auto
d-i netcfg/get_hostname string space-controller
d-i netcfg/get_domain string local
d-i debian-installer/locale string en_US.UTF-8
d-i time/zone string RU/Moscow

d-i passwd/make-user boolean false
d-i passwd/root-login boolean true
d-i passwd/root-password password cbsd
d-i passwd/root-password-again password cbsd
d-i user-setup/allow-password-weak boolean true

d-i apt-setup/cdrom/set-first boolean false
d-i apt-setup/cdrom/set-next boolean false
d-i apt-setup/cdrom/set-failed boolean false

d-i partman/early_command string debconf-set partman-auto/disk "$(list-devices disk | grep -v nvme[0-9]c[0-9] | head -n1)"
d-i partman-auto/method string lvm
d-i partman-auto/choose_recipe select atomic
d-i partman-lvm/device_remove_lvm boolean true
d-i partman-md/device_remove_md boolean true
d-i partman-lvm/confirm boolean true
d-i partman-lvm/confirm_nooverwrite boolean true
d-i partman-partitioning/confirm_write_new_label boolean true
d-i partman/choose_partition select finish
d-i partman/confirm boolean true
d-i partman/mount_style select uuid
d-i partman/confirm_nooverwrite boolean true

# Keep that one set to true so we end up with a UEFI enabled
# system. If set to false, /var/lib/partman/uefi_ignore will be touched
d-i partman-efi/non_efi_system boolean true

# enforce usage to GPT
d-i partman-basicfilesystems/no_swap boolean false
d-i partman-basicfilesystems/choose_label string gpt
d-i partman-basicfilesystems/default_label string gpt
d-i partman-partitioning/choose_label string gpt
d-i partman-partitioning/default_label string gpt
d-i partman/choose_label string gpt
d-i partman/default_label string gpt

d-i partman/mount_style select uuid
d-i partman-md/confirm boolean true

d-i apt-setup/use_mirror boolean false
d-i apt-setup/services-select multiselect
apt-cdrom-setup apt-setup/cdrom/set-first boolean false
popularity-contest popularity-contest/participate boolean false
d-i grub-installer/only_debian boolean true
d-i grub-installer/with_other_os boolean true
tasksel tasksel/first multiselect standard

apt-cacher-ng apt-cacher-ng/tunnelenable boolean false

# pkg
d-i pkgsel/include string openssh-server

d-i finish-install/reboot_in_progress note

###
#d-i partman-auto-lvm/guided_size string 50%
d-i partman-auto-lvm/guided_size string max
d-i partman-auto-lvm/new_vg_name string sys_vg01

#d-i partman-auto/choose_recipe select boot-root
#d-i partman-auto/choose_recipe select atomic
d-i partman-auto/expert_recipe string                           \
        boot-root ::                                            \
                256 8 1024 free                                 \
                $iflabel{ gpt }                                 \
                $reusemethod{ }                                 \
                method{ efi }                                   \
                format{ }                                       \
                .                                               \
                512 16 512 ext4                                 \
                $primary{ } $bootable{ }                        \
                method{ format } format{ }                      \
                use_filesystem{ } filesystem{ ext4 }            \
                mountpoint{ /boot }                             \
                .                                               \
                20000 64 -1 ext4                                \
                $lvmok{ }                                       \
                in_vg { sys_vg01 }                              \
                lv_name{ root }                                 \
                method{ format } format{ }                      \
                use_filesystem{ } filesystem{ ext4 }            \
                mountpoint{ / }                                 \
                .

```

</details>
</details>


## Полезные ссылки
<details>
  <summary>Некоторые полезные ресурсы, относящиеся к теме</summary>

( если знаете еще - не стесняйтесь добавить )

https://docs.freebsd.org/en/books/handbook/advanced-networking/#network-diskless

https://www.iventoy.com/en/index.html

https://github.com/annetutil/annet

https://theforeman.org

https://developer.hashicorp.com/packer

https://fai-project.org/

https://www.yoctoproject.org/

</details>
