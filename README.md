## instructions to get QEMU/KVM and virt-manager up and running on archlinux

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

### how to fix the qemu session

Check the output of the command virsh uri. If it returns qemu:///session, but you're using a qemu:///system connection in Virt-Manager, change it to qemu:///system like this:

edit your .bashrc file via sudo nano $HOME/.bashrc and add

    export LIBVIRT_DEFAULT_URI="qemu:///system"

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
