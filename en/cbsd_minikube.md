# CBSD + Minikube

This article describes the option of running Minikube in two ways - using a vanilla Debian cloud image and demonstrating the ability to create your own CBSD cloud images, which will allow you to deploy Minikube in ~10 seconds.

Both profiles are available in the [base distribution/profile library](https://github.com/cbsd/cbsd-vmprofiles).

[Minikube](https://minikube.sigs.k8s.io/docs/) - This is a compact version of a Kubernetes cluster that's convenient to run on your local workstation for testing or development purposes.
For example, suppose you're working on a FreeBSD server and need to run services designed for running in a Kubernetes environment. To do this, you run Minikube in the `bhyve`, `XEN`, or `QEMU` hypervisor and access the service through the kubectl thin client.

## Based on Debian 13 cloud

The minikube service is very easy to run and runs in one line:
```
minikube start
```
To create and launch a VM, in addition to the TUI and CLI interface, CBSD allows you to use CBSDfiles, which are sh scripts with predefined functions. Therefore, we will create a profile based on Debian 13 (of course, you can take any other Linux distribution from the profile collection), where in the postcreate function we will get and install the <string>.deb</strong> package, create a systemd unit file to launch the service, and finally download the finished kubeconfig to our host.

available on Github: [https://github.com/cbsd/cbsdfile-recipes/tree/master/bhyve/minikube](https://github.com/cbsd/cbsdfile-recipes/tree/master/bhyve/minikube)
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

Please note that in the CBSDfile we commented out `ci_gw4="10.0.0.1"` - this is the default gateway parameter Cloud-init configures.
It is recommended to set it globally via the ~cbsd/etc/bhyve-default-default.conf configuration file:
```
ci_gw4="<your GW>"
```

To avoid having to write the same parameters in each CBSDfile.

Create and launch a service from the CBSDfile:
```
cbsd up
```

### Using Minikube

If the instructions in the previous step worked correctly, you'll have a kubeconfig file in your directory that you can use with `kubectl`.
Additionally, you'll need to add a route for the Kubernetes network to properly interact with services. CBSDfile prints messages with instructions, 
but if you want to completely automate service creation, you can simply add instructions instead of echo:
```
/sbin/route delete 192.168.49.0/24 || true
/sbin/route add 192.168.49.0/24 ${ip4_addr}
```

After that, we begin working with K8S in the usual way:
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

If you carefully examined the script and its output, you've probably noticed that each time a service is created, the scripts download a .deb package from minikube, which
in turn downloads the necessary files and pods, approximately 3 GB in size, from the internet.

This isn't a big deal if we create a service infrequently, but if we do this frequently, it makes sense to convert the resulting disk image into a GOLD image—create it once and use it multiple times, thereby creating a cache of artifacts. To achieve this, we'll minimize the size of the created image to just enough for initial initialization—at the time of writing, 6 GB is sufficient. This minimization is necessary only to compact the image for distribution—the resulting systems that will run from this image will request any virtual disk size greater than 6 GB, and the disk will be expanded upon startup. Also, if you're using the ZFS file system, it's more beneficial to output an md/vnode file instead of a ZVOL in the ~cbsd/jails-data/VM/ directory. To achieve this, we'll change the imgsize= and imgtype= parameters and delete the /var/lib/cloud directory so that cloud-init components can fully function again when the image is launched. The resulting CBSDfile file:

available on Github: [https://github.com/cbsd/cbsdfile-recipes/tree/master/bhyve/minikube](https://github.com/cbsd/cbsdfile-recipes/tree/master/bhyve/minigold)

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

In this version of the CBSDfile, we're already unconditionally modifying the route for the 192.168.49.0/24 network to the virtual machine, since we're automatically
checking the service's availability via minikube. If it responds, we'll generate a disk image that can be used as a GOLD image: ~cbsd/jails-data/minigold-data/dsk1.vhd

Copy this file to the ~cbsd/src/ directory under the name we'll use in the profile we'll create for this image in the next step:

```
cp -a ~cbsd/jails-data/minigold-data/dsk1.vhd ~cbsd/src/iso/cbsd-cloud-Minikube-x86-1.0.raw
```

## Based on your own GOLD image

The previous step demonstrated how to easily create your own cloud image based on existing CBSD cloud profiles.
Let's write our own new CLOUD profile for CBSD for Minikube. Create a file ~cbsd/etc/vm-linux-cloud-Minikube-x86_64.conf with the following contents:

available on Github: [https://github.com/cbsd/cbsdfile-recipes/tree/master/bhyve/minik8s](https://github.com/cbsd/cbsdfile-recipes/tree/master/bhyve/minik8s)

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

This profile is now available as a choice in the main `cbsd bconstruct-tui` list or as vm_os_profile= for `cbsd bcreate`. However, we'll continue with best practices and write
another CBSDfile to demonstrate profile usage. If you're using ZFS, it will convert the file to ZVOL on the first run, and all subsequent runs will take 5-10 seconds until the Kubernetes API is ready.

For those who find minikube insufficient and want to use a specialized, native distribution focused on Kubernetes, we'll cover working with [Talos Linux in the next article](cbsd_talos_linux.md).

<center><img src="https://convectix.com/img/minik8s.png" width="1024" title="minik8s" alt="minik8s"/></center>
