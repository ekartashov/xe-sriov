# Iris Xe SR-IOV

<div align=center>
  <picture>
    <img alt="Iris Xe Logo" src="docs/images/iris_xe_logo.png" height="150">
  </picture>
</div>

## About

This repo is a **minimal guide** for enabling **SR-IOV** on Intel **Iris Xe iGPU** so you can pass **PCIe VFs** (Virtual Functions) via **VFIO** to your virtualisation stack.

> ⚠️ _**Warning**_
>
> The actual driver is provided by [`strongtz/i915-sriov-dkms`](https://github.com/strongtz/i915-sriov-dkms) repository, and this project pins a known-good release tag of it and guides how to actually get something working.

---

## Requirements

1. Intel **Iris Xe** iGPU on the host 
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
  
   > 📌 _**Note on Hardware Configuration**_
   >
   > Iris Xe iGPU is not always present, always check with `lspci`

2. **GNU/Linux** _**(Only Debian 13)**_

3. VT-d & IOMMU enabled

4. A supported kernel for the DKMS release tag you use (see upstream “Required kernel versions” in [`i915-sriov-dkms/README.md`](https://github.com/strongtz/i915-sriov-dkms/tree/2482f8fa4b1aabf10c5c9e5c1d4e37a84f2cdf57?tab=readme-ov-file#required-kernel-versions))

---

## DKMS Installation

1. Follow upstream **only**: **“Manual Installation Steps”** in [`i915-sriov-dkms/README.md`](https://github.com/strongtz/i915-sriov-dkms/tree/2482f8fa4b1aabf10c5c9e5c1d4e37a84f2cdf57?tab=readme-ov-file#manual-installation-steps)

   > ⚠️ **Note**
   >
   > - Use the **correct release tag** (this repo is pinned; do not install from random commits).
   > - For Linux guests, upstream expects the DKMS module to be installed **in the guest too**
   >   (see “Linux Guest Installation Steps” in the same README).

2. Set the required kernel parameters (pick **i915** or **xe**)  
   See upstream: [`i915-sriov-dkms/README.md`](https://github.com/strongtz/i915-sriov-dkms/tree/2482f8fa4b1aabf10c5c9e5c1d4e37a84f2cdf57?tab=readme-ov-file#required-kernel-parameters)

3. Create VFs (up to your configured count)  
   See upstream: [`i915-sriov-dkms/README.md`](https://github.com/strongtz/i915-sriov-dkms/tree/2482f8fa4b1aabf10c5c9e5c1d4e37a84f2cdf57?tab=readme-ov-file#creating-virtual-functions-vf)

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

2. Prevent the host from binding **VFs** to `i915/xe` _(persists after reboot)_:
   - Install `driverctl`
     ```bash
     sudo apt install -y driverctl
     ```

   - Set your arbitrary number of devices to be binded to `vfio-pci` skipping the first
     ```bash
     sudo modprobe vfio-pci
     
     # Replace VF_COUNT with the number of VFs you created (e.g. 7)
     VF_COUNT=7
     
     for f in $(seq 1 "$VF_COUNT"); do
       sudo driverctl set-override 0000:00:02.$f vfio-pci || break
     done
     
     driverctl list-overrides
  
     ```

     > ⚠️ **Note**
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

1. Add a VF PCIe device in virt-manager (example: `0000:00:02.1`).

2. Equivalent libvirt XML:

   ```xml
   <hostdev mode="subsystem" type="pci" managed="yes">
     <source>
       <address domain="0" bus="0" slot="2" function="1"/>
     </source>
   </hostdev>
   ```

   > ⚠️ **Note**
   >
   > * `function="1"` maps to `0000:00:02.1` (VF #1). `function="2"` → `0000:00:02.2`, etc.
   > * `managed="yes"` allows libvirt to detach/reattach/bind drivers on VM start/stop.
   >
   >   * If VFs are permanently on `vfio-pci` (recommended above), `managed="yes"` is usually fine.
   >   * If you still see reattach issues, set `managed="no"`:
   >
   >   ```xml
   >   <hostdev mode="subsystem" type="pci" managed="no">
   >     <source>
   >       <address domain="0" bus="0" slot="2" function="1"/>
   >     </source>
   >   </hostdev>
   >   ```

3. Linux guest note (Linux VM only):
   Follow upstream “Linux Guest Installation Steps” in [`i915-sriov-dkms/README.md`](https://github.com/strongtz/i915-sriov-dkms/tree/2482f8fa4b1aabf10c5c9e5c1d4e37a84f2cdf57?tab=readme-ov-file#linux-guest-installation-steps-ubuntu-2504kernel-614)

---

### Docker & Podman

> **Note**
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
