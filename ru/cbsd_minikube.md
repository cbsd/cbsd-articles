# CBSD + Minikube

В этой статье описывается вариант запуска Minikube в двух вариантах - с использованием ванильного cloud образа Debian и с демонстрацией возможности создания собственных cloud образов CBSD, что позволит разворачивать minikube за ~10 секунд.

Оба профиля доступны в [базовой поставке/библиотеке профилей](https://github.com/cbsd/cbsd-vmprofiles).

[Minikube](https://minikube.sigs.k8s.io/docs/) - это компактная версия кластера Kubernetes, которую удобно запускать на вашем локальном рабочем компьютере в целях тестирования или разработки. 
Например, вы работаете на FreeBSD сервере и необходимо запускать сервисы, ориентированные на запуск в среде Kubernetes. Для этого, вы запускаете Minikube в гипервизоре `bhyve`,`XEN` или `QEMU` и работаете с сервисом через тонкий клиент `kubectl`.

## На базе Debian 13 cloud

Сервис minikube очень прост в запуске и выполняется одной строкой:
```
minikube start
```

Для создания и запуска ВМ, помимо TUI и CLI интерфейса, CBSD позволяет использовать CBSDfile, являющиеся sh скриптом с заранее определенными функциями, поэтому создадим профиль на базе Debian 13 (конечно, вы можете взять любой другой Linux дистрибутив из коллекции профилей), где в postcreate функции получим и установим <string>.deb</strong> пакет, создадим systemd unit-файл для запуска сервиса и в финале скачаем готовый kubeconfig на наш хост.

доступен на Github: [https://github.com/cbsd/cbsdfile-recipes/tree/master/bhyve/minikube](https://github.com/cbsd/cbsdfile-recipes/tree/master/bhyve/minikube)
<details>
  <summary>CBSDfile</summary>

```
bhyve_minikube()
{
	vm_ram="8g"
	vm_cpus="4"
	imgsize="40g"
	vm_os_type="linux"
	vm_os_profile="cloud-Debian-13-x86_64"

	ip4_addr="DHCP"
	# ~cbsd/etc/bhyve-default-default.conf :
	# ci_gw4="10.0.0.1"

	ci_jname="${jname}"
	ci_fqdn="${fqdn}"
	ci_ip4_addr="${ip4_addr}"
	ci_gw4="${ip4_gw}"
	imgtype="zvol"
	runasap=1
	ssh_wait=1
}


postcreate_minikube()
{
	bexec <<EOF
export DEBIAN_FRONTEND="noninteractive"
sudo apt-get update
#
# Starting v1.39.0, minikube will default to "containerd" container runtime. See #21973 for more info.
# minikube start --container-runtime=containerd
sudo apt-get upgrade -y docker.io
wget https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube*.deb
sudo usermod -aG docker debian && newgrp docker
EOF

	# relogin to apply group members
	bexec <<EOF

/usr/bin/minikube start --driver=docker --embed-certs

[ ! -d /etc/systemd/system ] && sudo mkdir -p /etc/systemd/system

sudo tee /etc/systemd/system/minikube.service <<TEE_EOF
[Unit]
Description=Minikube auto-start
After=docker.service

[Service]
Type=simple
User=debian
ExecStart=/usr/bin/minikube start --driver=docker --embed-certs
Restart=on-failure

[Install]
WantedBy=multi-user.target
TEE_EOF

sudo systemctl daemon-reload
sudo systemctl enable minikube.service
#sudo systemctl start minikube.service

rm -rf .minikube/cache

minikube status
minikube ip

EOF

	bscp verbose=0 ${jname}:.kube/config kubeconfig

	echo "Minikube comes with a convenient graphical interface for managing resources via a browser."
	echo "Use: minikube dashboard"
	echo "To download and install dashboard.  If you're working in a terminal without a graphical interface, "
	echo "use minikube dashboard --url to get the link manually."
	echo
	${ECHO} "Don't forget to add route for 192.168.49.0/24 to ${ip4_addr}: ${N2_COLOR}/sbin/route add 192.168.49.0/24 ${ip4_addr}"
	echo
	${ECHO} "${N2_COLOR}kubectl get pods -A ${H2_COLOR}--kubeconfig=${CIX_PWD}/kubeconfig${N0_COLOR}"

	echo "hint: local PV:"
	echo "cbsd blogin"
	echo "minikube ssh"
	echo "sudo mkdir -p /mnt/data"
	echo "sudo chmod 777 /mnt/data"
	echo "exit"
	${ECHO} "kubectl apply -f pv.yaml ${H2_COLOR}--kubeconfig=${CIX_PWD}/kubeconfig${N0_COLOR}"
}
```

</details>

Обратите внимание, что в CBSDfile мы закомментировали `ci_gw4="10.0.0.1"` - настройка Cloud-init параметра шлюза по-умолчению. 
Рекомендуется указывать его глобально через конфигурационный файл ~cbsd/etc/bhyve-default-default.conf:
```
ci_gw4="<ваш GW>"
```

чтобы не писать в каждом CBSDfile одинаковые параметры.

Создаем и запускаем сервис из CBSDfile:
```
cbsd up
```

### Использование Minikube

Если инструкции предыдущего шага отработали корректно, в вашем каталоге будет файл kubeconfig, который вы можете использовать с `kubectl`.
Дополнительно, вам необходимо добавить маршрут для kubernetes сети, чтобы корректно взаимодействовать с сервисами. CBSDfile выводит сообщения
с инструкциями, но если вы хотите полностью автоматизировать создание сервиса, вместо вывода можете просто добавить инструкции вместо echo:
```
/sbin/route delete 192.168.49.0/24 || true
/sbin/route add 192.168.49.0/24 ${ip4_addr}
```

После чего, начинаем работать с K8S привычным для вас способом:
```
% kubectl get pods -A --kubeconfig=/root/upfile/bhyve/minikube/kubeconfig
NAMESPACE     NAME                               READY   STATUS    RESTARTS      AGE
kube-system   coredns-7d764666f9-hcgh4           1/1     Running   0             26m
kube-system   etcd-minikube                      1/1     Running   0             26m
kube-system   kube-apiserver-minikube            1/1     Running   0             26m
kube-system   kube-controller-manager-minikube   1/1     Running   0             26m
kube-system   kube-proxy-n29ms                   1/1     Running   0             26m
kube-system   kube-scheduler-minikube            1/1     Running   0             26m
kube-system   storage-provisioner                1/1     Running   1 (25m ago)   26m
```

## Minikube GOLD image

Если вы внимательно смотрели сценарий и результат его работы, то наверняка заметили, что при каждом создании сервиса скрипты выкачивают .deb пакет с minikube, который
в свою очередь, скачивает из интернета необходимые файлы и поды объемом ~3 GB.

Это не страшно, когда мы создаем сервис редко, но если мы делаем это часто - логично полученный образ диска превратить в GOLD-образ - создав единожды, использовать многократно, получив тем самым кеш артефактов. Для этого, минимизируем объем создаваемого образа до объема, достаточного лишь для первоначальной инициализации - на момент написания этой статьи, 6GB будет достаточно. Минимизация нужна лишь для компактности образа при дистрибьюции - итоговые системы, которые будут запускаться из этого образа запросить любой объем виртуального диска > 6 GB и при запуске диск будет расширен.
Также, если вы используете ZFS файловую систему, нам выгоднее получить на выходе md/vnode файл вместо ZVOL в директории ~cbsd/jails-data/VM/. Для этого, изменим параметры imgsize=, imgtype=, а также удалим каталог  /var/lib/cloud, чтобы компоненты cloud-init смогли полноценно отработать вновь при запуске образа. Итоговый CBSDfile файл:

доступен на Github: [https://github.com/cbsd/cbsdfile-recipes/tree/master/bhyve/minikube](https://github.com/cbsd/cbsdfile-recipes/tree/master/bhyve/minigold)

<details>
  <summary>CBSDfile</summary>

```
preup_minigold()
{
	${ECHO} "${W1_COLOR}Note: This CBSDfile controls the route for 192.168.49.0/24 network${N2_COLOR}"
}

bhyve_minigold()
{
	vm_ram="8g"
	vm_cpus="4"
	imgsize="6g"
	vm_os_type="linux"
	vm_os_profile="cloud-Debian-13-x86_64"

	ip4_addr="DHCP"
	# ~cbsd/etc/bhyve-default-default.conf :
	# ci_gw4="10.0.0.1"

	ci_jname="${jname}"
	ci_fqdn="${fqdn}"
	ci_ip4_addr="${ip4_addr}"
	ci_gw4="${ip4_gw}"
	imgtype="md"
	runasap=1
	ssh_wait=1
}


postcreate_minigold()
{
	set +o xtrace

	bexec <<EOF
export DEBIAN_FRONTEND="noninteractive"
sudo apt-get update
#
# Starting v1.39.0, minikube will default to "containerd" container runtime. See #21973 for more info.
# minikube start --container-runtime=containerd
sudo apt-get upgrade -y docker.io
wget https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube*.deb
sudo usermod -aG docker debian && newgrp docker
EOF

	# relogin to apply group members
	bexec <<EOF

/usr/bin/minikube start --driver=docker --embed-certs

[ ! -d /etc/systemd/system ] && sudo mkdir -p /etc/systemd/system

sudo tee /etc/systemd/system/minikube.service <<TEE_EOF
[Unit]
Description=Minikube auto-start
After=docker.service

[Service]
Type=simple
User=debian
ExecStart=/usr/bin/minikube start --driver=docker --embed-certs
Restart=on-failure

[Install]
WantedBy=multi-user.target
TEE_EOF

sudo systemctl daemon-reload
sudo systemctl enable minikube.service

rm -rf .minikube/cache
sudo rm -rf /var/lib/cloud
df -h /
minikube status
EOF

	ip4_addr=$( bget jname=${jname} mode=quiet ip4_addr )

	set -o xtrace
	bscp verbose=0 ${jname}:.kube/config kubeconfig
	${ROUTE_CMD} delete 192.168.49.0/24 || true
	${ROUTE_CMD} add 192.168.49.0/24 ${ip4_addr}
	set +o xtrace

	kubectl get pods -A --kubeconfig=${CIX_PWD}/kubeconfig
	_ret=$?

	${ROUTE_CMD} delete 192.168.49.0/24 || true

	bstop jname=${jname}

	if [ ${_ret} -ne 0 ]; then
		${ECHO} "${W1_COLOR}${CIX_APP} tests failed: ${N2_COLOR}kubectl get pods -A --kubeconfig=${CIX_PWD}/kubeconfig${N0_COLOR}"
		exit ${_ret}
	fi

	echo
	${DU_CMD} -sh ~cbsd/jails-data/minigold-data/dsk1.vhd
	echo

	${ECHO} "${N2_COLOR}Copy the disk ${H2_COLOR}~cbsd/jails-data/${jname}-data/dsk1.vhd ${N2_COLOR}into ~cbsd/src/iso, e.g.:${N0_COLOR}"
	${ECHO} "${H2_COLOR}${CP_CMD} -a ~cbsd/jails-data/${jname}-data/dsk1.vhd ~cbsd/src/iso/cbsd-cloud-Minikube-x86-1.0.raw${N0_COLOR}"
	${ECHO} "${W1_COLOR}Info: ${N1_COLOR}Make sure you don't have a file and cbsd-cloud-Minikube-x86-1.0.raw ZOVL (${N2_COLOR}zfs list | grep cbsd-cloud-Minikube-x86-1.0.raw${N1_COLOR}) from a previous installation.${N0_COLOR}"
	echo
}
```
</details>

В этой версии CBSDfile, мы уже безусловно модифицируем маршрут для сети 192.168.49.0/24 на виртуальную машину, поскольку в автоматическом режиме
проверяем доступность сервиса через minikube. Если он отвечает на запрос - мы генерируем образ диска, который можно использовать как GOLD-образ: ~cbsd/jails-data/minigold-data/dsk1.vhd

Скопируем этот файл в директорию ~cbsd/src/ под именем, которое мы будем использовать в профиле, который мы создадим под этот образ в следующем шаге:

```
cp -a ~cbsd/jails-data/minigold-data/dsk1.vhd ~cbsd/src/iso/cbsd-cloud-Minikube-x86-1.0.raw
```

## На базе собственного GOLD образа

В предыдушем шаге был продемонстрирован вариант, как можно легким образом создать свой собственный cloud образ на базе существующих cloud-профилей CBSD. 
Давайте напишем собственный новый CLOUD-профиль для CBSD для Minikube. Создайте файл ~cbsd/etc/vm-linux-cloud-Minikube-x86_64.conf с таким содержимым:

доступен на Github: [https://github.com/cbsd/cbsdfile-recipes/tree/master/bhyve/minik8s](https://github.com/cbsd/cbsdfile-recipes/tree/master/bhyve/minik8s)

<details>
  <summary>CBSDfile</summary>

```
# don't remove this line:
vm_profile="cloud-Minikube-x86_64"
vm_os_type="linux"
# this is one-string additional info strings in dialogue menu
long_description="Minikube: Debian-based (cloud)"

# fetch area:
fetch=1

# Official resources to fetch ISO's
iso_site="https://mirror.convectix.com/cloud/"

# Official CBSD project mirrors ( info: https://github.com/cbsd/mirrors )
cbsd_iso_mirrors="https://mirror.convectix.com/cloud/ https://raw.githubusercontent.com/cbsd/mirrors/refs/heads/main/cbsd-cloud.txt"

iso_img="Minikube-x86-0.1.raw"
iso_img_dist="${iso_img}.xz"

sha256sum="0"
iso_img_dist_size="460119088"

# enp0sX
ci_adjust_inteface_helper=1

iso_img_type="cloud"

iso_extract="nice -n 19 ${IDLE_IONICE} ${XZ_CMD} -d ${iso_img_dist}"

# register_iso as:
register_iso_name="cbsd-cloud-Minikube-x86-1.0.raw"
register_iso_as="cloud-debian-x86-1.0"

default_jailname="minikube"

# disable profile?
xen_active=1
bhyve_active=1
qemu_active=1

# Available in ClonOS?
clonos_active=1

# Available for MyB? image name
myb_image="minikube"

# VNC
vm_vnc_port="0"
vm_efi="uefi"

vm_package="small1"

# VirtualBox Area
virtualbox_ostype="FreeBSD_64"

# is template for vm_obtain
is_template=1
is_cloud=1

imgsize_min="10g"
imgsize="16g"

# enable birtio RNG interface?
virtio_rnd="1"

## cloud-init specific settings ##
ci_template="centos9"
ci_user_pw_root='*'

ci_user_add='debian'

ci_user_gecos='debian user'
ci_user_home='/home/debian'
ci_user_shell='/bin/bash'
ci_user_member_groups='root'
ci_user_pw_crypt='*'
ci_user_pubkey=".ssh/id_rsa.pub"

default_ci_ip4_addr="DHCP"
default_ci_gw4="auto"

ci_nameserver_address="8.8.8.8"
ci_nameserver_search="my.domain"
## cloud-init specific settings end of ##
```
</details>

Теперь этот профиль доступен в выборе в основном списке `cbsd bconstruct-tui` или в качестве vm_os_profile= для `cbsd bcreate`. Но мы продолжим best-practies и напишем
еще один CBSDfile в качестве демонстрации использования профиля. Если вы используете ZFS, при первом запуске он сконвертирует файл из файла в ZVOL и все последующие
запуски будут происходить в течении 5-10 секунд до готовности Kubernetes API.

Для тех, кому minikube недостаточно и хочется использовать специализированный самобытный дистрибутив, ориентированный на Kubernetes - в следующей статье мы рассмотрим работу с Talos Linux

<center><img src="https://convectix.com/img/minik8s.png" width="1024" title="minik8s" alt="minik8s"/></center>
