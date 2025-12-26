# Iris Xe SR-IOV

<div align="center">
  <img alt="Iris Xe Max Logo" src="docs/images/iris_xe-max_logo.png" height="150">
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
  <img alt="Iris Xe Logo" src="docs/images/iris_xe_logo.png" height="150">
</div>


## About

This repo is a **minimal guide** for enabling **SR-IOV** on Intel **Iris Xe iGPU** so you can pass **PCIe VFs** (Virtual Functions) via **VFIO** to your virtualisation stack.

> ⚠️ _**Warning & Credits:**_
>
> The actual driver comes from the upstream [`strongtz/i915-sriov-dkms`](https://github.com/strongtz/i915-sriov-dkms) project, and this repo includes it locally under [`./i915-sriov-dkms/`](./i915-sriov-dkms/) as a `git subtree`, thus pining a known-good revision release _(2025.12.10)_, and focuses on how to actually use the created virtual devices correctly.

---

## Requirements

1. **Host** with **Intel Iris Xe** iGPU
   <table>
     <thead>
       <tr>
         <th>Supported Intel CPU Gen</th>
         <th>Gen Number</th>
       </tr>
     </thead>
     <tbody>
       <tr>
         <td>Alder Lake</td>
         <td>12th</td>
       </tr>
       <tr>
         <td>Raptor Lake</td>
         <td>13th</td>
       </tr>
       <tr>
         <td>Raptor Lake Refresh</td>
         <td>14th</td>
       </tr>
     </tbody>
   </table>

   > 📌 _**Note on Hardware Configuration:**_
   >
   > **Iris Xe** iGPU is _**not always present**_, always check with `lspci`.

2. **GNU/Linux** _**(Should be any but, tested only on Debian 13)**_
3. **Enabled VT-d** & **IOMMU**
4. A supported kernel for the DKMS revision you use (see [Required kernel versions](./i915-sriov-dkms/README.md#required-kernel-versions))

---

## DKMS Installation

1. Follow **only**: [**Manual Installation Steps**](./i915-sriov-dkms/README.md#manual-installation-steps)

   > 📌 _**Note on Installation:**_
   >
   > - Use the **correct pinned revision in this repo** (do not install from random commits).
   > - For Linux VM Guests, don't forget that the DKMS module must be installed **in the Guest too** (will come in following sections)

2. Set the required kernel parameters (pick **i915** or **xe**)  
   See upstream: [*Required Kernel Parameters*](./i915-sriov-dkms/README.md#required-kernel-parameters)

3. Reboot to Create VFs (up to your configured count in the kernel parameters)
   
---

## Host: prevent binding to VFs

1. First, check what driver is bound to PF + each VF:
   ```bash
   lspci -Dnn | awk '/VGA compatible controller|3D controller/ {print $1}' | while read -r bdf; do
     echo "== $bdf =="
     lspci -nnk -s "$bdf" | sed -n '1,6p'
     echo -n "driver: "
     basename "$(readlink -f "/sys/bus/pci/devices/$bdf/driver" 2>/dev/null)" || echo "unbound"
     echo
   done
   ```

2. Prevent the host from binding **VFs** to `i915/xe` *(persists after reboot)*:

   * Install `driverctl`

     ```bash
     sudo apt install -y driverctl
     ```

   * Bind an arbitrary number of VFs to `vfio-pci` now and on further reboots (skip the PF at `.0`)

     ```bash
     sudo modprobe vfio-pci

     # Replace VF_COUNT with the number of VFs you created (e.g. 7)
     VF_COUNT=7

     for f in $(seq 1 "$VF_COUNT"); do
       sudo driverctl set-override 0000:00:02.$f vfio-pci || break
     done

     driverctl list-overrides
     ```

     > ❗ _**Important Note to Prevent Crashes:**_
     >
     > Keep:
     >
     > * PF (`0000:00:02.0`) → host graphics driver
     > * VFs (`0000:00:02.1+`) → `vfio-pci` permanently
     >
     > This avoids “driver reattach” during VM start/stop preventing loop of crashes.

---

## Virtual Devices Usage

### QEMU/KVM on libvirtd (& virt-manager)

1. Add a VF PCIe device in virt-manager and/or XML (example: `0000:00:02.1`):

   ```xml
   <hostdev mode="subsystem" type="pci" managed="yes">
     <source>
       <address domain="0" bus="0" slot="2" function="1"/>
     </source>
   </hostdev>
   ```

   > 📌 _**Note on `managed` Option:**_
   >
   > - `function="1"` maps to `0000:00:02.1` (VF #1). `function="2"` → `0000:00:02.2`, etc.
   > - `managed="yes"` allows libvirt to detach/reattach/bind drivers on VM start/stop.
   >
   >   - If VFs are permanently on `vfio-pci` (recommended above), `managed="yes"` is usually fine.
   >   - If you still see reattach issues, set `managed="no"`

2. Guest OS Setup:

   * **Linux VM:** follow [*Linux Guest Installation Steps*](./i915-sriov-dkms/README.md#linux-guest-installation-steps-ubuntu-2504kernel-61)
   * **Windows VM:** follow [*Windows Guest*](./i915-sriov-dkms/README.md#windows-guest-tested-with-proxmox-83--windows-11-24h2--intel-driver-32010164603201016259)

---

### Docker & Podman

> 📌 _**Note on PCIe in Containers:**_
>
> Containers share the host kernel; you typically do not attach an SR-IOV VF as a PCI device to Docker/Podman.
> The usual approach is sharing the host render node.

1. Share render node:

   ```bash
   # docker
   docker run --rm -it \
     --device /dev/dri/renderD128 \
     --group-add video --group-add render \
     <image>

   # podman
   podman run --rm -it \
     --device /dev/dri/renderD128 \
     --group-add keep-groups \
     <image>
   ```

---

### LXC & Incus

1. Attach a VF (example: `0000:00:02.1`) via PCI passthrough:

   ```bash
   # Incus VM (recommended)
   incus config device add <vm-name> igpu-vf pci address=0000:00:02.1
   ```

2. Container attach is possible but more policy-sensitive:

   ```bash
   incus config device add <ct-name> igpu-vf pci address=0000:00:02.1
   ```
