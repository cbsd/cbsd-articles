# CBSD + Talos Linux

This article describes how to configure and use Talos Linux in a virtual machine running the bhyve hypervisor on FreeBSD.

Talos Linux is a highly specialized open-source Linux distribution designed exclusively for running Kubernetes clusters.

It is not a general-purpose OS and does not provide a shell/interpreter (bash/sh). All management and maintenance is performed through a declarative API using the `talosctl` utility.

The profile for creating a VM is available in the [base distribution/profile library](https://github.com/cbsd/cbsd-vmprofiles).

## Getting ready for work

We assume that the network allocated to the bhyve virtual machines is served by a DHCP server (e.g., using an NFSv4/DHCP/PXE jail-based (vnet) service via (dnsmasq) (dnsmasq_service.md)),
so that Talos immediately receives a working IP address when booting from the ISO image.

At the time of writing, the FreeBSD ports collection does not contain the `talosctl` utility. We can create a CentOS/Ubuntu/Debian virtual machine to manage the Talos cluster,
but to reduce overhead (there's no point in running a complete operating system for a single thin client), we'll use Linuxulator and CBSD's capabilities for creating Linux containers.

:bangbang: | :information_source: Important! You must first activate the Linux kernel using the linux_enable="YES" parameter in the /etc/rc.conf file.
:---: | :---

If there was no previous configuration and the host has not been rebooted, load the linuxulator modules manually:
```
kldload linux linux64
```

Let's create a Linux-Devuan-based container, either through the TUI interface (`cbsd jconstruct-tui`) or in the CLI and install talosctl:

:bangbang: | :information_source: Warning! Using talosctl requires the latest Linuxulator. If the commands don't produce the expected output during setup, try using a full-fledged Linux distribution, such as a Debian 13 cloud image.
:---: | :---


```
cbsd jcreate jname=talosctl jprofile=devuan_daedalus runasap=1
cbsd jlogin talosctl
apt install -y curl
curl -sL https://talos.dev/install | sh
```

If everything is OK, you should see the version with the `talosctl version` command:
```
Client:
        Tag:         v1.12.4
        SHA:         fc8e600b
        Built:
        Go version:  go1.25.7
        OS/Arch:     linux/amd64
```

Further operations with the Talos API via talosctl are performed from this container.


## Launching Talos VM

Check the presence of the profile:
```
cbsd get-profiles src=iso | grep -i talos
```
it should be something like:
```
Talos-x86_64                     linux       Talos: 1.12.4
```

Create and launch the VM. Choose a flavor based on your specifications and available resources:

```
cbsd bcreate jname=talos1 imgsize=20g vm_ram=4g vm_cpus=2 vm_os_profile=Talos-x86_64 vm_os_type=linux runasap=1
```
(Add `bhyve_vnc_tcp_bind=0.0.0.0` to connect to VNC from a remote host without forwarding or redirecting.)

To find out the IP address and system status, access the graphical console (VNC). Example output:

<center><img src="https://convectix.com/img/talos1.png" width="1024" title="talos_view" alt="talos_view"/></center>

:information_source: Pay attention to the IP field, in our example Talos received <strong>172.16.0.56</strong> and we will use it further in the text.

Let's generate by default:
```
talosctl gen config my-cluster https://172.16.0.56:6443
```
output:
```
generating PKI and tokens
Created /root/controlplane.yaml
Created /root/worker.yaml
Created /root/talosconfig
```

Next, we need a list of disks to correctly specify the Talos installation path:

```
talosctl -n 172.16.0.56 get disks --insecure
```
output:
```
talosctl -n 172.16.0.56 get disks --insecure
NODE   NAMESPACE   TYPE   ID      VERSION   SIZE     READ ONLY   TRANSPORT   ROTATIONAL   WWID   MODEL           SERIAL
       runtime     Disk   loop0   2         4.1 kB   true
       runtime     Disk   loop1   2         75 MB    true
       runtime     Disk   sr0     2         213 MB   false       sata        true                BHYVE DVD-ROM
       runtime     Disk   vda     2         22 GB    false       virtio      true                                BHYVE-1B20-A59A-A264

```

In our case, `vda` is the correct drive. Let's edit controlplane.yaml and worker.yaml to specify the correct drive in the `install` section:
```
disk: /dev/vda # The disk used for installations.
```

In addition, in this example we will combine the control-plane and worker functions, so in the cluster section we will uncomment and explicitly set the allowSchedulingOnControlPlanes parameter:
```
cluster:
  allowSchedulingOnControlPlanes: true
```

:information_source: If you're unsure of your YAML literacy, you can use validate to check the syntax:
```
talosctl validate --config controlplane.yaml --mode metal
```

Send a command to install Talks:
```
talosctl apply-config --insecure -n 172.16.0.56 --file controlplane.yaml
```

After a reboot and the API is available (visible in the graphical console):

Get talosconfig and create the cluster:
```
export TALOSCONFIG=./talosconfig
talosctl config endpoint 172.16.0.56
talosctl config node 172.16.0.56
talosctl bootstrap
```

bootstrap will start 'etcd' and Kubernetes components.

One sign that bootstrap has been successful will be the mass greening of Kubernetes components to a Healthy state:

<center><img src="https://convectix.com/img/talos2.png" width="1024" title="talos_view2" alt="talos_view2"/></center>

After bootstrapping is complete, we get the config for `kubectl` and check its functionality:

```
talosctl kubeconfig ./kubeconfig
export KUBECONFIG=./kubeconfig
kubectl get nodes
```
output:
```
NAME            STATUS   ROLES           AGE     VERSION
talos-i55-2on   Ready    control-plane   9m46s   v1.35.0
```

## initialize PV and pod deploy

To make our single-node installation a little more complete, let's add a separate virtual disk
and initialize it as a PV. To do this, stop the hypervisor, add the disk, and restart it:

```
cbsd bstop jname=talos1
cbsd bhyve-dsk mode=attach jname=talos1 dsk_controller=virtio-blk dsk_size=20g
cbsd bstart talos1
```

Let's create the local-storage.yaml configuration:
```
machine:
  disks:
    - device: /dev/vdb # Correct disk for data
      partitions:
        - mountpoint: /var/mnt/local-data
          size: 0 # 0 - using all available space
```

Применяем:
```
talosctl patch mc --nodes 172.16.0.56 --patch @local-storage.yaml
```

:information_source: Talos reboots the system when applied, wait until the cluster is ready to serve requests

Verification: You should now see the drive and mount point in the output of these commands:
```
talosctl -n 172.16.0.56 get volumestatus
talosctl -n 172.16.0.56 get mountstatus
```

Create a PV manifest, file pv-local.yaml:
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

Where `talos-ah4-we7` is the name of your node from `kubectl get nodes`

```
kubectl apply -f pv-local.yaml
```

Verify. You should see the PV in the output of `kubectl get pv`:
```
NAME           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS    VOLUMEATTRIBUTESCLASS   REASON   AGE
local-pv-vdb   20Gi       RWO            Retain           Available           local-storage   <unset>                          4m27s
```

Create a StorageClass, file sc-local.yaml:
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner # Indicates that PVs are created manually
volumeBindingMode: WaitForFirstConsumer   # Critical for local disks
```

Apply and test: 
```
kubectl apply -f sc-local.yaml
kubectl get sc local-storage
```

Create a PVC, file pvc-local.yaml:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-storage # Must match SC and PV
  resources:
    requests:
      storage: 20Gi # Must be less than or equal to the size in PV
```

Apply:
```
kubectl apply -f pvc-local.yaml
```

Check - using the command `kubectl get pvc` we should see a new resource in the Pending status (waiting for the first PVC user):
```
NAME             STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS    VOLUMEATTRIBUTESCLASS   AGE
local-data-pvc   Pending                                      local-storage   <unset>                 27s
```


We'll test writing to the PVC by creating a test pod and a file called test-pod-writer.yaml:
```
apiVersion: v1
kind: Pod
metadata:
  name: storage-writer
spec:
  containers:
  - name: writer
    image: alpine
    # The script writes the date to the file /mnt/test-log.txt every 10 seconds.
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

Apply:
```
kubectl apply -f test-pod-writer.yaml
```

Looking from Talos's side:
```
talosctl -n 172.16.0.56 list /var/mnt/local-data
talosctl -n 172.16.0.56 cat /var/mnt/local-data/test-log.txt
```
