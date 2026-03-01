# CBSD + Talos Linux

В этой статье описывается вариант настройки и использования Talos Linux в виртуальной машине под гипервизором bhyve на платформе FreeBSD.

Talos Linux - это узкоспециализированный дистрибутив Linux с открытым исходным кодом, созданный исключительно для запуска кластеров Kubernetes. 
Он не является ОС общего назначения и не предоставляет оболочки/интерпретатора (bash/sh). Все управление и обслуживание выполняется через 
декларативный API с помощью утилиты `talosctl`.

Профиль для создания ВМ доступен в [базовой поставке/библиотеке профилей](https://github.com/cbsd/cbsd-vmprofiles).

## Подготовка к работе

Мы предполагаем, что сеть, которая отдана под bhyve виртуальных машин обслуживается DHCP сервером (например, с помощью [NFSv4/DHCP/PXE jail-based (vnet) сервис через (dnsmasq)](dnsmasq_service.md)), 
чтобы Talos при старте из ISO образа сразу получал рабочий IP адрес.

На момент написания статьи, коллекция портов FreeBSD не содержит утилиту `talosctl`. Мы можем сделать виртуальную машину на базе CentOS/Ubuntu/Debian для управления Talos кластером,
но для снижения оверхеда (нет смысла запускать целостную операционную систему ради одного тонкого клиента) воспользуемся линуксулятором и возможностями CBSD по созданию Linux контейнеров.

:bangbang: | :information_source: Внимание! Предварительно вы должны активировать линуксулятор через linux_enable="YES" параметр в файле /etc/rc.conf
:---: | :---

Если ранее настройки небыло и хост не перезагружался, загрузите модули linuxulator вручную:
```
kldload linux linux64
```

Создадим Linux-Devuan-based контейнер, либо через TUI интерфейс (`cbsd jconstruct-tui`), либо в CLI и установим talosctl:

:bangbang: | :information_source: Внимание! работа с `talosctl` требует максимально свежого linuxulator. Если в процессе настройки команды не отрабатывают с ожидаемым выводом, найдите любой полноценный Linux, например cloud образ Debian 13
:---: | :---


```
cbsd jcreate jname=talosctl jprofile=devuan_daedalus runasap=1
cbsd jlogin talosctl
apt install -y curl
curl -sL https://talos.dev/install | sh
```

Если все в порядке, вы должны увидеть версию по команде `talosctl version`:
```
Client:
        Tag:         v1.12.4
        SHA:         fc8e600b
        Built:
        Go version:  go1.25.7
        OS/Arch:     linux/amd64
```

Дальнейшие операции с Talos API через talosctl выполняем из этого контейнера.


## Запуск Talos ВМ

проверяем наличие профиля:
```
cbsd get-profiles src=iso | grep -i talos
```
должно быть что-то вроде:
```
Talos-x86_64                     linux       Talos: 1.12.4
```

Создаем и запускаем ВМ. Flavor выбирайте на свое усмотрение, в зависимости от характеристик и свободных ресурсов:

```
cbsd bcreate jname=talos1 imgsize=20g vm_ram=4g vm_cpus=2 vm_os_profile=Talos-x86_64 vm_os_type=linux runasap=1
```
( добавьте `bhyve_vnc_tcp_bind=0.0.0.0`, чтобы соединяться с VNC с удаленного хоста без пробросов/редиректов )

Чтобы узнать IP адрес и состояние системы, обратитесь на графическую консоль (VNC), пример вывода:

<center><img src="https://convectix.com/img/talos1.png" width="1024" title="talos_view" alt="talos_view"/></center>

:information_source: Обратите внимание на поле IP, в нашем примере Talos получила <strong>172.16.0.56</strong> и будем его использовать в далее по тексту.


Сгенерируем по-умолчанию:
```
talosctl gen config my-cluster https://172.16.0.56:6443
```
вывод:
```
generating PKI and tokens
Created /root/controlplane.yaml
Created /root/worker.yaml
Created /root/talosconfig
```

Далее, нам потребуется список дисков, чтобы корректно указать путь установки Talos:

```
talosctl -n 172.16.0.56 get disks --insecure
```
вывод:
```
talosctl -n 172.16.0.56 get disks --insecure
NODE   NAMESPACE   TYPE   ID      VERSION   SIZE     READ ONLY   TRANSPORT   ROTATIONAL   WWID   MODEL           SERIAL
       runtime     Disk   loop0   2         4.1 kB   true
       runtime     Disk   loop1   2         75 MB    true
       runtime     Disk   sr0     2         213 MB   false       sata        true                BHYVE DVD-ROM
       runtime     Disk   vda     2         22 GB    false       virtio      true                                BHYVE-1B20-A59A-A264

```

в нашем случае `vda` - это правильный диск. Отредактируем controlplane.yaml и worker.yaml, указав в секции `install` корректный диск:
```
disk: /dev/vda # The disk used for installations.
```

Кроме этого, в этом примере мы совместим функции control-plane и worker, поэтому в секции cluster раскомментируем и явно выставим параметр allowSchedulingOnControlPlanes:
```
cluster:
  allowSchedulingOnControlPlanes: true
```

:information_source: Если не уверены в своей YAML грамотности, можете воспользоваться validate для проверки синтаксиса:
```
talosctl validate --config controlplane.yaml --mode metal
```

Посылаем команду на инсталляцию Talks:
```
talosctl apply-config --insecure -n 172.16.0.56 --file controlplane.yaml
```

После перезагрузки и доступности API (можно увидеть в графической консоли):

получим talosconfig и создадим кластер:
```
export TALOSCONFIG=./talosconfig
talosctl config endpoint 172.16.0.56
talosctl config node 172.16.0.56
talosctl bootstrap
```

bootstrap запустит `etcd` и компоненты Kubernetes.

Одним из признаков, что bootstrap прошел усешно, будет массовое озеленение компонентов кубернетес в состояние Healty:

<center><img src="https://convectix.com/img/talos2.png" width="1024" title="talos_view2" alt="talos_view2"/></center>

После завершения бутстрапа получаем конфиг для `kubectl` и проверяем работоспособность:

```
talosctl kubeconfig ./kubeconfig
export KUBECONFIG=./kubeconfig
kubectl get nodes
```
Вывод:
```
NAME            STATUS   ROLES           AGE     VERSION
talos-i55-2on   Ready    control-plane   9m46s   v1.35.0
```

## инициализируем PV и деплоим под

Чтобы наша однонодовая инсталляция стала чуть более полноценна, добавим отдельный виртуальный диск
и проинициализируем его в качестве PV. Для этого, остановим гипервизор, добавим диск и перезапустим вновь:

```
cbsd bstop jname=talos1
cbsd bhyve-dsk mode=attach jname=talos1 dsk_controller=virtio-blk dsk_size=20g
cbsd bstart talos1
```

Создадим конфигурацию local-storage.yaml:
```
machine:
  disks:
    - device: /dev/vdb # Корректный диск для данных
      partitions:
        - mountpoint: /var/mnt/local-data
          size: 0 # 0 - использование всего доступного пространства
```

Применяем:
```
talosctl patch mc --nodes 172.16.0.56 --patch @local-storage.yaml
```

:information_source: Talos перезагружает систему при применении, дождитесь готовности кластера обслужить запросы

Проверка: вы должны видеть теперь диск и точку монтирования в выводе этих команд:
```
talosctl -n 172.16.0.56 get volumestatus
talosctl -n 172.16.0.56 get mountstatus
```

Создаем манифест PV, файл pv-local.yaml:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-vdb
spec:
  capacity:
    storage: 20Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /var/mnt/local-data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - talos-ah4-we7
```

Где `talos-ah4-we7` - имя вашей ноды из `kubectl get nodes`

```
kubectl apply -f pv-local.yaml
```

Проверка. Вы должны увидеть PV в выводе `kubectl get pv`:
```
NAME           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS    VOLUMEATTRIBUTESCLASS   REASON   AGE
local-pv-vdb   20Gi       RWO            Retain           Available           local-storage   <unset>                          4m27s
```

Создаем StorageClass, файл sc-local.yaml:
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner # Указывает, что PV создаются вручную
volumeBindingMode: WaitForFirstConsumer   # Критично для локальных дисков
```

Применяем и проверяем: 
```
kubectl apply -f sc-local.yaml
kubectl get sc local-storage
```

Создаем PVC, файл pvc-local.yaml:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-storage # Должно совпадать с SC и PV
  resources:
    requests:
      storage: 20Gi # Должно быть меньше или равно размеру в PV
```

Применяем:
```
kubectl apply -f pvc-local.yaml
```

Проверка - по команде `kubectl get pvc` мы должны увидеть новый ресурс в статусе Pending (ожидает первого пользователя PVC):
```
NAME             STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS    VOLUMEATTRIBUTESCLASS   AGE
local-data-pvc   Pending                                      local-storage   <unset>                 27s
```


Проверяем запись на PVC, создадим тестовый pod, файл test-pod-writer.yaml:
```
apiVersion: v1
kind: Pod
metadata:
  name: storage-writer
spec:
  containers:
  - name: writer
    image: alpine
    # Скрипт пишет дату в файл /mnt/test-log.txt раз в 10секунду
    command: ["/bin/sh", "-c"]
    args:
      - while true; do
          date >> /mnt/test-log.txt;
          echo "data written to /mnt/test-log.txt";
          sleep 10;
        done
    volumeMounts:
    - name: data
      mountPath: /mnt
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: local-data-pvc

```

Применяем:
```
kubectl apply -f test-pod-writer.yaml
```

Смотрим со стороны Talos:
```
talosctl -n 172.16.0.56 list /var/mnt/local-data
talosctl -n 172.16.0.56 cat /var/mnt/local-data/test-log.txt
```
