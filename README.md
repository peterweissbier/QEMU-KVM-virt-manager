## Easy instructions to get QEMU/KVM and virt-manager up and running on Arch

1. Make sure your cpu support `kvm` with below command:

        grep -E "(vmx|svm)" --color=always /proc/cpuinfo

2. Make sure BIOS have enable “Virtualization Technology”.

3. User access to `/dev/kvm` so add your account into kvm(78) group:

        sudo gpasswd -a $(whoami) kvm

4. Loading kernel modules `kvm_intel` or `kvm_amd` depend on your CPU, Add module name in `/etc/modules-load.d/kvm.conf`:

        kvm_intel

  * Load module:

        modprobe kvm_amd

5. Install `qemu`, `virt-manager`, `dnsmasq` and `iptables`:

        sudo pacman -S --needed qemu virt-manager dnsmasq iptables-nft dmidecode

6. Run and enable boot up start `libvirtd` daemon:

        sudo systemctl enable --now libvirtd


7. Use PolicyKit authorization create `/etc/polkit-1/rules.d/50-libvirt.rules` (before `/etc/polkit-1/rules.d/50-org.libvirt.unix.manage.rules`) as below context:

```
/* Allow users in kvm group to manage the libvirt
daemon without authentication */
polkit.addRule(function(action, subject) {
    if (action.id == "org.libvirt.unix.manage" &&
        subject.isInGroup("kvm")) {
            return polkit.Result.YES;
    }
});
```

8. You will need to create the libvirt group and add any users you want to have access to **libvirt** to that group:

        groupadd libvirt
    
        sudo gpasswd -a $(whoami) libvirt


9. Check network interface status:

        sudo virsh net-list --all

  * If it is `inactive` start it using:

        sudo virsh net-start default  

10. Now you can use virt-manager manager your virtual machine.

#### Things to do after installing a Windows VM

  * Check and install drivers on your guest Windows VM like virtio-win

---

#### What to do if `default` network interface is not listed

   * If `virsh net-list` is not listing any network interface just reinitialize it with,
   
         sudo virsh net-define /usr/share/libvirt/networks/default.xml
   
   * Then just `autostart` it like so,
   
         sudo virsh net-autostart default 

---



#### What to do if you cannot access storage file, and get "Permission denied Error in KVM Libvirt"

   * Step 1: Edit `/etc/libvirt/qemu.conf` file:
   
         sudo nano /etc/libvirt/qemu.conf
   
   * Step 2: Find the `user` and `group` directives. By default, both are set to `"root"`,
   
          [...] 
          Some examples of valid values are:
          #
          user = "qemu"   # A user named "qemu"
          user = "+0"     # Super user (uid=0)
          user = "100"    # A user named "100" or a user with uid=100
          #
          #user = "root"
          The group for QEMU processes run by the system instance. It can be
          specified in a similar way to user.
          #group = "root"
          [...]

     Uncomment both lines and replace root with your username and group with libvirt as shown below:

          [...] 
          Some examples of valid values are:
          #
          user = "qemu"   # A user named "qemu"
          user = "+0"     # Super user (uid=0)
          user = "100"    # A user named "100" or a user with uid=100
          #
          user = "sk"
          The group for QEMU processes run by the system instance. It can be
          specified in a similar way to user.
          group = "libvirt"
          [...]

   * Step 3: Restart libvirtd service:

          sudo systemctl restart libvirtd


---
  
## How to extend / increase a Windows Partition on KVM QEMU VM

We have a Windows 7 VM running on Ubuntu KVM. I needed to give the Windows 7 machine more disk space. This turns out to be really easy (when you know how).

1. Shutdown the VM

        virsh shutdown hostname

2. Increase the qcow2 image

Find the qcow2 file of the VM and take a backup (just in case).

    cp hostname.qcow2 hostname.qcow2.backup
    qemu-img resize hostname.qcow2 +100GB
    
3. Start the VM

        virsh start hostname

4. Extend the partition in Window

Windows has a really good partition management utility built into it. Search for `disk management`

---

### fix qemu session

Check the output of the command virsh uri. If it returns qemu:///session, but you're using a qemu:///system connection in Virt-Manager, change it to qemu:///system like this:

edit your .bashrc file via sudo nano $HOME/.bashrc and add

    export LIBVIRT_DEFAULT_URI="qemu:///system"

### add virtio to an existing windows VM

download Windows_VirtIO_Drivers on https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/?C=M;O=D

Wizard Installation

You can use an easy wizard to install all, or a selection, of VirtIO drivers.

Open the Windows Explorer and navigate to the CD-ROM drive.
Simply execute (double-click on) virtio-win-gt-x64
Follow its instructions.
(Optional) use the virtio-win-guest-tools wizard to install the QEMU Guest Agent and the SPICE agent for an improved remote-viewer experience.
Reboot VM
Manual Installation

Open the Windows Explorer and navigate to the CD-ROM drive. There you can see that the ISO consists of several directories, each having sub-directories for supported OS version (for example, 2k19, 2k12R2, w7, w8.1, w10, ...). Balloon guest-agent NetKVM qxl vioscsi ...
Navigate to the desired driver directories and respective Windows Version
Right-click on the file with type "Setup Information"
A context menu opens, select "Install" here.
Repeat that process for all desired drivers
Reboot VM.
Downloading the Wizard in the VM

You can also just download the most recent virtio-win-gt-x64.msi from https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.248-1/ from inside the VM, if you already have network access. Then just execute it and follow the installation process.




### Reference:

Original guide - http://wood1978.dyndns.org/~wood/wordpress/2013/07/22/arch-linux-setup-kvm-with-virt-manager-gui/comment-page-1/
 
KVM @ Arch Wiki - https://wiki.archlinux.org/index.php/KVM

libvirt @ Arch Wiki - https://wiki.archlinux.org/index.php/Libvirt

QEMU @ Arch Wiki - https://wiki.archlinux.org/index.php/QEMU

Network Interface Status - http://ask.xmodulo.com/network-default-is-not-active.html

Network Interface Troubleshooting - https://blog.programster.org/kvm-missing-default-network

Networking libvirt wiki - https://wiki.libvirt.org/page/Networking#NAT_forwarding_.28aka_.22virtual_networks.22.29

Fedora Wiki - Windows Virtio Drivers - https://stg.fedoraproject.org/wiki/Windows_Virtio_Drivers

Extend disk size Windows partition KVM QEMU VM - https://www.randomhacks.co.uk/how-to-extend-increase-a-windows-partition-on-kvm-qemu-vm/

Increasing a KVM machine disk space - https://serverfault.com/questions/324281/how-do-you-increase-a-kvm-guests-disk-space

Fixing `Cannot access storage file, Permission denied Error in KVM Libvirt` - https://ostechnix.com/solved-cannot-access-storage-file-permission-denied-error-in-kvm-libvirt/
