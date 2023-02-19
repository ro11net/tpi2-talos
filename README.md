# Overview

## Introduction

This guide will walk you through installing a Talos Kubernetes cluster on Jetson Nanos running on the Turing Pi 2 board. My goals when building Kubernetes clusters for "edge" scenarios is always to conserve as many resources as possible. When running Kubernetes on devices with limited resources, I tend to build them in sort of an HCI type model where every node should be allowed to run workloads instead of wasting resources by letting the controlplane just sit there. There are a few unique features this guide will provide you with:

- A lightweight Kubernetes cluster with a TRUE highly available controlplane
  - 3 controlplane nodes and 1 worker node
  - This cluster will be built with a Virtual IP on the controlplane nodes for Kube API server access
- Distributed local storage across all nodes in the cluster
  - This is possible with Jiva as it is extremely lightweight compared to other distributed storage options such as longhorn, rook-ceph, mayastor, etc.
  - (Optional) With the use of NVMe M.2 as storage
- Dynamic provisioning of Kubernetes Persistent Volume Claims

### Hardware

- Turing Pi 2 board (Version 1 before they shipped with the BMC battery fix)
  - NVMe option addon (This is optional on this guide. You can use SD card storage instead if you'd like)
- 4X 1TB NVMe M.2 drives. Assorted. Basically all the extras I had laying around. (Optional for this guide)
- 4X Jetson Nano Developer Kits (They're cheaper and more accessible than CM4s right now)
  - [https://www.amazon.com/dp/B084DSDDLT?psc=1&ref=ppx_yo2ov_dt_b_product_details](https://www.amazon.com/dp/B084DSDDLT?psc=1&ref=ppx_yo2ov_dt_b_product_details)
- Alumunium Mini-ITX case
  - [https://www.amazon.com/dp/B0953WPZ4W?psc=1&ref=ppx_yo2ov_dt_b_product_details](https://www.amazon.com/dp/B0953WPZ4W?psc=1&ref=ppx_yo2ov_dt_b_product_details)
- Silverstone 350W Flex ATX Power Supply (definitely overkill for this board, but it's small and I'll use it for other projects too)
  - [https://www.amazon.com/dp/B07QGMX5DW?psc=1&ref=ppx_yo2ov_dt_b_product_details](https://www.amazon.com/dp/B07QGMX5DW?psc=1&ref=ppx_yo2ov_dt_b_product_details)
- USB-TTL cable (optional unless you want to view the status of certain things in this guide)
  - [https://www.amazon.com/dp/B08BLKBK1K?ref=ppx_yo2ov_dt_b_product_details&th=1](https://www.amazon.com/dp/B08BLKBK1K?ref=ppx_yo2ov_dt_b_product_details&th=1)

## Additional Resources

If you want more details on any of the steps contained in this repo, you can visit the following sources:

- Installing Talos on the Jetson Nano
  - [https://www.talos.dev/v1.3/talos-guides/install/single-board-computers/jetson_nano/](https://www.talos.dev/v1.3/talos-guides/install/single-board-computers/jetson_nano/)
- Editing the Talos Machine Configuration
  - [https://www.talos.dev/v1.3/talos-guides/configuration/editing-machine-configuration/](https://www.talos.dev/v1.3/talos-guides/configuration/editing-machine-configuration/)
- Replicated Local Storage on Talos with OpenEBS Jiva
  - [https://www.talos.dev/v1.3/kubernetes-guides/configuration/replicated-local-storage-with-openebs-jiva/](https://www.talos.dev/v1.3/kubernetes-guides/configuration/replicated-local-storage-with-openebs-jiva/)
- OpenEBS Jiva Documentation:
  - [https://openebs.io/docs/concepts/jiva](https://openebs.io/docs/concepts/jiva)
- OpenEBS Jiva Helm Charts
  - [https://openebs.github.io/jiva-operator/](https://openebs.github.io/jiva-operator/)

The initial installation steps contained here could also be modified for other SBC boards and you can find thorough guides in the Talos Docs here:

- [https://www.talos.dev/v1.3/talos-guides/install/single-board-computers/](https://www.talos.dev/v1.3/talos-guides/install/single-board-computers/)

Definitely take a look through the Talos Docs as there are a lot of other configurations and capabilities not covered here.

## Pre-Requisites

You need to have the following apps installed on your workstation. This guide will walk you through the rest.

- Docker Desktop
- Make
- Go 1.19 (You will need this to build the crane binary)
- kubectl
- helm
- talosctl >=1.3.4

## Clone this Repository

- This steps within this guide have been tested with Ubuntu 20.04 and I cannot guarantee you wont have issues using a different operating system.

## Build the firmware

To install Talos on the Jetson Nano, we will be using a upstream version of `u-boot` with patches from NVIDIA. This is provided by Sidero Labs, but we will need to use crane to replace the binary in the existing firmware package.

Before we can get the u-boot binary for the firmware, we need to install crane-cli. If you are not familiar with Go, this could seem complicated, but it's a pretty simple process.

First, pull Google's go-containerregistry from here:

- [https://github.com/google/go-containerregistry](https://github.com/google/go-containerregistry)

Next, navigate to the `main.go` file for crane.

```shell
cd go-containerregistry/cmd/crane/
```

You should see a bunch of files in here, and these files will all be compiled into a single binary. You can do this by running the following command:

```shell
go build -o crane main.go
```

Now there is a file here named `crane`. Let's move it to a location in our PATH. I'm using Ubuntu so I will use:

```shell
mv crane /usr/local/bin/
```

The binary also needs to be made executable:

```shell
chmod +x /usr/local/bin/crane
```

Now that crane is installed, we'll be able to modify the firmware, which can be pulled with the following command:

curl -SLO <https://developer.nvidia.com/downloads/remetpack-463r32releasev73t210jetson-210linur3273aarch64tbz2>

Decompress this file you just downloaded:

```shell
tar xf remetpack-463r32releasev73t210jetson-210linur3273aarch64tbz2
```

You will now see a directory named Linux_For_Tegra in your repository

Navigate into this directory and run the following command to replace the u-boot with the one from Sidero Labs:

> **NOTE:** The Docker Daemon will need to be running in order to use crane.

crane --platform=linux/arm64 export ghcr.io/siderolabs/u-boot:v1.3.0-alpha.0-25-g0ac7773 - | tar xf - --strip-components=1 -C bootloader/t210ref/p3450-0000/u-boot.bin

## Flash Firmware to the Compute Module

To flash the firmware on to the Jetson Nano, we want to leave the compute module in the IO board it came in for now. The compute module will also need to be placed into recovery mode. To put the Jetson Nano into Recovery Mode, for the Jetson Nano Developer Kit A02, short Pin 3 and Pin 4 of J40 with a jumper pin and turn on the power. For the Developer Kit B01 and 2GB, short Pin9 and Pin10 of J50 with jumper pins and turn on the power.

Connect a USB-microUSB cable to the microUSB/5V power on the IO board and run the following command to ensure it is connected to your computer

```shell
lsusb | grep -i "nvidia"
```

Now, while still in the Linux_for_Tegra directory, we can run the following command to flash the firmware.

Run the shell script

```shell
sudo ./flash.sh p3448-0000-max-spi external
```

There will be a lot of output displayed and it make take a few minutes to flash.

Repeat this process with all 4 board.

> **NOTE:** The firmware flashing process only needs to be done once per board. If you need to re-image the device, you will do that with the imaging process to the SD card.

## Flash Image to SD card

### Build iscsi extension into image

Since we'll be using the iscsi extension to use OpenEBS Jiva for distributed local storage, we need to build this extension into the image. This step is optional, but if you do not build it into the image, you will need to run an OS upgrade on each Talos node after the installation is complete in order for it to apply download and apply the extension. Steps to upgrade Talos can be found [here](https://www.talos.dev/v1.3/talos-guides/upgrading-talos/). It's not difficult, but it can take up to 20 minutes per node, so I prefer this option.

Clone the Talos source code repository

```shell
git clone https://github.com/siderolabs/talos.git
```

Navigate into the directory you just cloned:

```shell
cd talos/
```

change to the versions branch you would like, I used 1.3, which at the time of writing this builds an image with Talos v1.3.4

```shell
git checkout release-1.3
```

Now we need to make the directory for the image to be placed after it's build

```shell
mkdir _out
```

Run the following command which will use the Makefile in the repository to create the image and place it in the `_out` directory we just created:

```shell
make sbc-jetson_nano IMAGER_SYSTEM_EXTENSIONS="ghcr.io/siderolabs/iscsi-tools:v0.1.1"
```

Navigate into the \_out directory and decompress the image file

```shell
xz --decompress metal-jetson_nano-arm64.img.xz
```

### Flash the SD card

You can now flash the image to the SD card with the following command

> **NOTE:** Balena Etcher may also be used to flash the image to the SD card if you prefer that method.

- Replace `/dev/mmcblk0` with the name of your SD card

```bash
sudo dd if=metal-jetson_nano-arm64.img of=/dev/mmcblk0 conv=fsync bs=4M status=progress
```

## Boot the nodes

You can now insert the Jetson Nano compute modules into the Turing Pi 2 and apply power, then press the KEY1 button to boot the nodes. If you followed the steps so far exactly and used the image and firmware provided in this repo, your devices should boot fine on the Turing Pi 2.

- You will need to either use wireshark or check your router for new devices to find the IP address of each node.

> **NOTE:** I have seen issues with the boards rebooting if it is done through the tpi cli from the BMC console. The best way to reboot until TPI fixes the bugs with the BMC firmware is to pull power completely and power on from scratch.

If you want to ensure the device is booting properly, you may use a USB-TTL cable to monitor a serial connection to the Jetson Nano board.

> **NOTE:** Currently, even from the TPI2's BMC console, you cannot get access to the UART connection when Talos has been installed when it is in the TPI2 board.

> **ANOTHER NOTE:** If you are having issues with your Turing Pi 2 booting the nodes at all, make sure the BMC battery is removed, as it is currently known to cause issues with the BMC booting.

## Generate configs

`talosctl` makes it easy for us to generate the configs for our cluster. First we want to generate secrets so the cluster can talk securely:

```shell
talosctl gen secrets -o secrets.yaml
```

Then generate the configs from the secrets.yaml:

```shell
talosctl gen config --with-secrets secrets.yaml "<cluster-name>" https://<virtual-ip>:6443
```

This will generate 3 files: controlplane.yaml, worker.yaml, and talosconfig

- Rename the controlplane.yaml to controlplane1.yaml or another name of your choosing to keep things organized since we will have multiple controlplanes

## Modify Controlplane 1 Config

Add the nodes ip and hostname (if you plan on using DNS to connect to the nodes) to the `machine.certSANs` value:

```yaml
### snippet ###
machine:
  type: controlplane
  token: your-generated-token
  ca:
    crt: your-generated-crt
    key: your-generated-key

  # Add this: #
  certSANs:
    - <node-1-ip>
    - <node-1-hostname>
### snippet ###
```

Since we plan on using OpenEBS Jiva, there are a few configurations needed to make in order to allow Jiva to have access to the files on our system.

- Add the `machine.kubelet.extraMounts` option to your config:

```yaml
machine:
  ### snippet ###
  kubelet:
    image: ghcr.io/siderolabs/kubelet:v1.26.0
    defaultRuntimeSeccompProfileEnabled: true
    disableManifestsDirectory: true

    # Add this: #
    extraMounts:
      - destination: /var/openebs/local
        options:
          - bind
          - rshared
          - rw
        source: /var/openebs/local
        type: bind
### snippet ###
```

This next part is important to the high availability of our cluster. Since we will be using 3 controlplane nodes, we are going to create a virtual IP on each controlplane nodes which will be used to join new nodes to the cluster as well as access via kubectl or talosctl.

- Configure the hostname and vip

> **NOTE:** The vip address must be different than the rest of the nodes in your cluster. This is also where you may set a static IP for your node if you like. Instead, I have configured DHCP reservations for each Jetson Nano on my router.

```yaml
machine:
  ### snippet ###
  network:
    # Add This: #
    hostname: <node-hostname>
    interfaces:
      - interface: eth0
        dhcp: true
        vip:
          ip: <vip-ip-address>
### snippet ###
```

Next, change the installation location on the SD card. Because we will need the iscsi tools provided by sidero to use Jiva, this is where the extension must be added as well.

> **NOTE:** Do not use the nvme slot here. Talos will not be able to use it as the boot drive. Instead, we'll add the NVMe slot to be mounted where Jiva can use it.

- Change the install disk to your SD card and add the iscsi extension.

```yaml
machine:
  ### snippet ###
  install:
    disk: /dev/mmcblk1 # Change the disk
    image: ghcr.io/siderolabs/installer:v1.3.2
    bootloader: true
    wipe: false
    # Add this #
    extensions:
      - image: ghcr.io/siderolabs/iscsi-tools:v0.1.1
### snippet ###
```

> **NOTEL:** If you need to verify existing disks on your device, you can do so with `talosctl -n <node-ip> disks --insecure`

This is where we will add the nvme drive to be used for our distributed storage and give our applications the best performance possible.

> **NOTE:** This step is optional. If you would like to use the SD card for your distributed storage with Jiva, do not configure anything here.

- Add the following under `machine`. The device will be your NVMe drive and the mountpoint will be `/var/openebs`:

```yaml
machine:
  ### snippet ###
  disks:
    - device: /dev/nvme0n1 # The name of the disk to use
      partitions:
        - mountpoint: /var/openebs # This is the location Jiva will use
```

The last thing we will change is `cluster.apiServer.certSANs`. This should have **ALL** potential alternative names for the API server's certs. We will want to Virtual IP as well as all the controlplane IPs and hostnames added here so we can authenticate to any of them if necessary. We are removing the taints from the controlplane nodes by settings `cluster.allowSchedulingOnControlPlanes` to `true`. This is done because we want all pods to be able to run on all nodes using all the resources we have available.

```yaml
cluster:
  ### snippet ###

  allowSchedulingOnControlPlanes: true # Add this line to remove NoSchedule taints on controlplane nodes
  apiServer:
    image: registry.k8s.io/kube-apiserver:v1.26.0
    certSANs:
      # Add this #
      - <vip-ip>
      - <node-1-ip>
      - <node-2-ip>
      - <node-3-ip>
      - <vip-hostname(if you might be using DNS)>
      - <node-1-hostname>
      - <node-2-hostname>
      - <node-3-hostname>
### snippet ###
```

## Create the config for nodes 2 and 3

copy the file `controlplane1.yaml` to create 2 new files for controlplane 2 and 3

```shell
cp controlplane1.yaml controlplane2.yaml
cp controlplane1.yaml controlplane3.yaml
```

Edit the `machine.certSANs` and `machine.network` settings in the files for nodes 2 and 3.

## Edit the worker file

Edit the following settings in the `worker.yaml` file which was generated for you:

```yaml
### snippet ###
machine:
  type: controlplane
  token: your-generated-token
  ca:
    crt: your-generated-crt
    key: your-generated-key

  # Add this: #
  certSANs:
    - <node-4-ip>
    - <node-4-hostname>
### snippet ###
```

`machine.kubelet`

```yaml
machine:
  ### snippet ###
  kubelet:
    image: ghcr.io/siderolabs/kubelet:v1.26.0
    defaultRuntimeSeccompProfileEnabled: true
    disableManifestsDirectory: true

    # Add this: #
    extraMounts:
      - destination: /var/openebs/local
        options:
          - bind
          - rshared
          - rw
        source: /var/openebs/local
        type: bind
### snippet ###
```

`machine.network`

```yaml
machine:
  ### snippet ###
  network:
    # Add This: #
    hostname: <node-hostname>
    interfaces:
      - interface: eth0
        dhcp: true
        vip:
          ip: <vip-ip-address>
### snippet ###
```

`machine.install`

```yaml
machine:
  ### snippet ###
  install:
    disk: /dev/mmcblk1 # Change the disk
    image: ghcr.io/siderolabs/installer:v1.3.2
    bootloader: true
    wipe: false
    # Add this #
    extensions:
      - image: ghcr.io/siderolabs/iscsi-tools:v0.1.1
### snippet ###
```

- `machine.disks`

```yaml
machine:
  ### snippet ###
  disks:
    - device: /dev/nvme0n1 # The name of the disk to use
      partitions:
        - mountpoint: /var/openebs # This is the location Jiva will use
```

## Apply configs

Now that all of your configs are ready, we can start applying them. Make sure you see the network and power lights on on the Turing Pi.

Run the following command to apply the controlplane config:

```shell
talosctl apply-config --insecure -n <node-1-ip> --file controlplane1.yaml
```

It may take a few seconds but once the config is applied, you'll be able to bootstrap the node with the following command:

```shell
talosctl bootstrap --nodes <node-1-ip>
```

> **NOTE:** The boostrap command only needs to be done once on the first node.

Now that our config has been applied, we will not be able to access it with talosctl because we can no longer use the flag `--insecure` on it. To add the cluster to our talos config, you can run the following command:

```shell
talosctl config merge ./talosconfig
```

Now we can follow the logs on our node with `talosctl -n <virtual-ip> logs etcd -f` or the individual nodes logs with `talosctl -n <node-1-ip> dmesg -f`

We still don't have a kubeconfig though to run kubernetes commands on our node. To generate this with talosctl run:

```shell
talosctl kubeconfig -f -n <virtual-ip>
```

Now you can run `kubectl get nodes` and see if Node 1 is ready.

Next, apply the configs for all the other nodes. Now that node 1 has been bootstrapped, the rest of these can be applied at once if you like.

```shell
talosctl apply-config --insecure -n <node-2-ip> --file controlplane2.yaml
talosctl apply-config --insecure -n <node-3-ip> --file controlplane3.yaml
talosctl apply-config --insecure -n <node-4-ip> --file worker.yaml
```

Once you're cluster is up and running, and all nodes show `Ready` you can continue to the next step

> **NOTE:** `talosctl` also provides us an option to view oour resource using with the following command:

```shell
talosctl -n <virtual-ip or node-ip> dashboard
```

Keyboard Shortcuts:

```shell
Keyboard shortcuts:
 - h, <Left>: switch one node to the left
 - l, <Right>: switch one node to the right
 - j, <Down>: scroll process list down
 - k, <Up>: scroll process list up
 - <C-d>: scroll process list half page down
 - <C-u>: scroll process list half page up
 - <C-f>: scroll process list one page down
 - <C-b>: scroll process list one page up
```

## Verify ISCSI Extension

Before we can install OpenEBS Jiva, we need to ensure the iscsi extensions were installed successfully. You can check this with:

```shell
talosctl -n <node-ip> get extensions
```

The output should look similar to this:

```shell
NODE         NAMESPACE   TYPE              ID               VERSION   NAME          VERSION
192.168.0.101   runtime     ExtensionStatus   tmp.oqYMf2FxtU   1         iscsi-tools   v0.1.1
```

Next, check the services to make sure they are running

```shell
talosctl -n <node-ip> services
```

The output will look similar to this if everything worked as expected:

```shell
NODE         SERVICE      STATE     HEALTH   LAST CHANGE   LAST EVENT
192.168.0.101   apid         Running   OK       16m15s ago    Health check successful
192.168.0.101   containerd   Running   OK       16m21s ago    Health check successful
192.168.0.101   cri          Running   OK       16m4s ago     Health check successful
192.168.0.101   etcd         Running   OK       15m50s ago    Health check successful
192.168.0.101   ext-iscsid   Running   ?        16m4s ago     Started task ext-iscsid (PID 4063) for container ext-iscsid
192.168.0.101   ext-tgtd     Running   ?        16m4s ago     Started task ext-tgtd (PID 4017) for container ext-tgtd
192.168.0.101   kubelet      Running   OK       15m21s ago    Health check successful
192.168.0.101   machined     Running   OK       16m32s ago    Health check successful
192.168.0.101   trustd       Running   OK       16m4s ago     Health check successful
192.168.0.101   udevd        Running   OK       16m6s ago     Health check successful
```

Check this on all 4 nodes before continuing on to the next step

## Install OpenEVS Jiva

Our configuration's default Pod Security Admission will not immediately allow us to run privileged pods on Kubernetes. First, we need to allow privileged pods in the openebs namespace so Jiva can run.

- First create the namespace

```shell
kubectl create namespace openebs
```

- Then apply the label to the namespace

```shell
kubectl label ns openebs pod-security.kubernetes.io/enforce=privileged
```

Now you can install Jiva. This will be installed using helm with the following command:

```shell
helm repo add openebs-jiva https://openebs.github.io/jiva-operator
helm repo update
helm upgrade --install --namespace openebs --version 3.2.0 openebs-jiva openebs-jiva/jiva
```

Check to make sure the pods are coming up with:

```shell
kubectl get pods -n openebs
```

The output will look like this when the installation is complete:

```shell
NAME                                                READY   STATUS    RESTARTS   AGE
openebs-jiva-csi-controller-0                       5/5     Running   0          62s
openebs-jiva-csi-node-7w8ts                         3/3     Running   0          62s
openebs-jiva-csi-node-d67n2                         3/3     Running   0          62s
openebs-jiva-csi-node-gpmk9                         3/3     Running   0          61s
openebs-jiva-csi-node-tbzwp                         3/3     Running   0          62s
openebs-jiva-localpv-provisioner-798557c5bb-2hrj9   1/1     Running   0          62s
openebs-jiva-operator-55cb6679b6-gjjlj              1/1     Running   0          62s
```

If you did not set `cluster.allowSchedulingOnControlPlanes` to `true` in the machineconfig, you will need to either set toleratiions in the Jiva resources or remove the `node-role.kubernetes.io/control-plane:NoSchedule` taint from the controlplane nodes with the following command:

```shell
kubectl taint node rollnet-tpi-1 node-role.kubernetes.io/control-plane:NoSchedule-
kubectl taint node rollnet-tpi-2 node-role.kubernetes.io/control-plane:NoSchedule-
kubectl taint node rollnet-tpi-3 node-role.kubernetes.io/control-plane:NoSchedule-
```

Since Jiva assumes iscsid to be running natively on the host and not as a Talos extension service, we need to modify the CSI node daemon set to enable it to find the PID of the iscsid service. In addition, Jiva also needs to modify the default config map to execute iscsiadm commands inside the PID namespace of the iscsid service.

Create a file named `configmap.yaml` with the following contents (an example is also stored in the manifest directory of this repo):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: openebs-jiva-csi-iscsiadm
  namespace: openebs
  labels:
    app.kubernetes.io/managed-by: pulumi
data:
  iscsiadm: |
    #!/bin/sh
    set -euo pipefail

    # Find the process ID of the iscsid daemon
    iscsid_pid=$(pgrep iscsid)

    # Enter the namespaces of the iscsid process and execute iscsiadm with given arguments
    nsenter --mount=/proc/${iscsid_pid}/ns/mnt --net=/proc/${iscsid_pid}/ns/net -- /usr/local/sbin/iscsiadm "$@"
```

Apply this file to replace the existing configmap:

```shell
kubectl --namespace openebs apply --filename configmap.yaml
```

Next, the Jiva CSI daemonset needs to be run with `hostPID: true` so it can find the PID if the iscsid service we installed with the extension:

```shell
kubectl --namespace openebs patch daemonset openebs-jiva-csi-node --type=json --patch '[{"op": "add", "path": "/spec/template/spec/hostPID", "value": true}]'
```

## Test the storage

If everything installed correctly, we should now be able to use our distributed storage with workloads. To test this you can use the following PVC and Deployment manifests (examples are also stored in the manifest directory of this repo):

Create a file named `deployment-test.yaml` with the following contents:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox
  labels:
    app: busybox
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: busybox
  template:
    metadata:
      labels:
        app: busybox
    spec:
      containers:
        - name: busybox
          image: busybox:1.35
          command: ["sh", "-c", "echo Container 1 is Running ; sleep 3600"]
          imagePullPolicy: IfNotPresent
          resources:
            limits:
              cpu: 500m
              memory: 264Mi
            requests:
              cpu: 250m
              memory: 128Mi
          ports:
            - containerPort: 3306
              name: busybox
          volumeMounts:
            - mountPath: /var/lib/mysql
              name: demo-vol1
          securityContext:
            runAsUser: 1000
            runAsGroup: 3000
            readOnlyRootFilesystem: true
      volumes:
        - name: demo-vol1
          persistentVolumeClaim:
            claimName: example-jiva-csi-pvc
      readinessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 10
        timeoutSeconds: 5
        periodSeconds: 30
        failureThreshold: 3
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 30
        timeoutSeconds: 5
        periodSeconds: 60
        failureThreshold: 5
```

Create a file named `pvc-test.yaml` with the following contents:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-jiva-csi-pvc
spec:
  storageClassName: openebs-jiva-csi-default
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
```

Apply the PVC first:

```shell
kubectl apply -f pvc-test.yaml
```

The PVC should now be bound which says everything is working correctly

```shell
kubectl get pvc
```

You will also notice there are 3 replicas in the openEBS namespace:

```shell
# kubectl get pods -n openebs
NAME                                                              READY   STATUS    RESTARTS   AGE
openebs-jiva-csi-controller-0                                     5/5     Running   0          15m
openebs-jiva-csi-node-6zjdj                                       3/3     Running   0          3m19s
openebs-jiva-csi-node-8l4dg                                       3/3     Running   0          3m14s
openebs-jiva-csi-node-b6hns                                       3/3     Running   0          3m25s
openebs-jiva-csi-node-zjvsk                                       3/3     Running   0          3m30s
openebs-jiva-localpv-provisioner-798557c5bb-2hrj9                 1/1     Running   0          15m
openebs-jiva-operator-55cb6679b6-gjjlj                            1/1     Running   0          15m
pvc-373b9c56-8cf2-48a8-9e47-bb3a825ae931-jiva-ctrl-7fcf7d4sx4zb   2/2     Running   0          3m16s
pvc-373b9c56-8cf2-48a8-9e47-bb3a825ae931-jiva-rep-0               1/1     Running   0          3m16s
pvc-373b9c56-8cf2-48a8-9e47-bb3a825ae931-jiva-rep-1               1/1     Running   0          3m16s
pvc-373b9c56-8cf2-48a8-9e47-bb3a825ae931-jiva-rep-2               1/1     Running   0          3m16s
```

Next apply the deployment manifest you created:

```shell
kubectl apply -f deployment-test.yaml
```

Give it a few seconds to deploy and check to make sure your pods are running:

```shell
kubectl get pods
```

Check the logs to make sure there are no errors using the name of your pod

```shell
kubectl logs busybox-d8fc8d494-nj94r
```

## Conclusion

Hopefully this helped you get Kubernetes running on your Turing Pi 2 board. Kubernetes can be a lot of fun and Talos makes it easy to enjoy. Please submit issues or pull requests to this repo if anything is missing or incorrect in this walkthrough. Thanks!

- Christian Rolland (@ro11net)
