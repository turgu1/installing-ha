# Installing Home Assistant 

This is the procedure I used to install Home Assistant on a fresh installation of Ubuntu Server 24.04 on a Lenovo ThinkCenter mini computer.

It is expected that the OS has been installed with openSSH and Mosquitto options.

Note that the server **MUST** be connected to the network through an Ethernet port.

0. Connect to the server. Replace `[USERNAME]` with the username created when installing the OS, and `[IP ADDRESS]` with the server IP address. You can find it on the DHCP server (usually the main router of the network):

    ```
    $ ssh [USERNAME]@[IP ADDRESS]
    ```

1. Update the system: `sudo apt update; sudo apt upgrade`

2. Add .ssh credentials for login without password. 

   ```
   $ scp .ssh/id_rsa.pub [USERNAME]@[IP ADDRESS]:.ssh/authorized_keys`
   ```

3. Adjust timezone: `sudo timedatectl set-timezone EST`

4. Lock root login: `sudo passwd -l root`

5. Secure SSH authentication:

    ```
    $ sudo su
    $ cd /etc/ssh
    $ nano sshd_config
      -> PermitRootLogin no
      -> PermitEmptyPassword no
      -> PasswordAuthentication no

      Save file
    $ /etc/init.d/ssh restart
    ```

6. Reboot: `sync; sync; reboot`

7. Login to the Host Assistant server

8. Check if the CPU can run VMs:

    ```
    $ egrep -c '(vmx|svm)' /proc/cpuinfo
    8
    ```

    If the answer is 0, you need to go into the BIOS and change it to allow for VMs creation/run

9. Install kvm:

   ```
   $ sudo apt install -y qemu-kvm \
   libvirt-daemon-system libvirt-clients \
   bridge-utils ovmf virt-manager
   ```
10. Add user to libvirt and kvm (may already been defined):

    ```
    $ sudo adduser [USER] libvirt
    $ sudo adduser [USER] kvm

11. Check that the libvirtd daemon is running:

    ```
    $ sudo systemctl status libvirtd
    ```

12. Create Home Assistant VM folder and download image:

    ```
    $ sudo mkdir -vp /var/lib/libvirt/images/hassos-vm 
    $ sudo cd /var/lib/libvirt/images/hassos-vm
    $ sudo wget https://github.com/home-assistant/operating-system/releases/download/14.2/haos_ova-14.2.qcow2.xz
    ```

13. Setup storage pool:

    ```
    $ sudo virsh pool-create-as --name hassos --type dir --target /var/lib/libvirt/images/hassos-vm
    ```
14. Set up network

    Install net-tools: `sudp apt install net-tools`

    Check the Ethernet device name: `ifconfig` note the entry with the server's IP addres. For me: eno1

    Start the editor:

    ```$ nano /etc/netplan/00-installer-config.yaml```

    Modify the file to get a result similar to the following (replace `eno1` with your own Ethernet device name):

    ```
    # This is the network config written by 'subiquity'
    network:
    ethernets:
        eno1:
        dhcp4: false
        dhcp6: false
    version: 2
    bridges:
        br0:
        dhcp4: true
        dhcp6: false
        interfaces:
                - eno1
        parameters:
            stp: true
    ```

    Change it's protection:

    ```
    chmod 600 /etc/netplan/00-installer-config.yaml
    ```

    This configuration is good for a DHCP based network.

    You may have to remove the file `50-cloud-init.yaml` if present in the folder:

    ```
    sudo rm /etc/netplan/50-cloud-init.yml
    ```

    It's now time to generate the files and apply them to the system:

    ```
    $ sudo netplan generate
    $ sudo netplan apply
    ```

    With the last command, you will loose the connection with the server. Look back at the DHCP server to find a new device appearing with a new address, and reconnect to it:

    ```
    $ ssh [USERNAME]@[NEW IP ADDRESS]
    ```

    (This is the address supplied to the virtual device named br0. This is the bridge for VMs access to the network)

    ```
    $ sudo brctl show
    ```

    Check that you get the list of bridges. br0 must be listed:

    ```
    bridge name	bridge id			STP enabled	interfaces
    br0			8000.00215ec64304	yes		eno1
    ```

15. Install the hassos VM

    Goto the /var/lib/libvirt/images/hassos-vm and decompress the VM image. But before, we use sudo to become an priviledge process (The prompt becomes a '#' instead of '$')

    ```
    $ sudo su
    # cd /var/lib/libvirt/images/hassos-vm
    # unxz haos_ova-14.2.qcow2.xz
    ```

    Now create the VM instance:

    ```
    # virt-install --import --name hassos \
    --memory 4096 --vcpus 4 --cpu host \
    --disk haos_ova-14.2.qcow2,format=qcow2,bus=virtio \
    --network bridge=br0,model=virtio \
    --osinfo detect=on,require=off \
    --graphics none \
    --noautoconsole \
    --boot uefi
    ```

    To see if the VM is running:

    ```
    # virsh list --all
    ```

    The answer will look like this:

    ```
    Id   Name     State
    ------------------------
    1    hassos   running
    ```

    You can retrieve the VM MAC address using the following command:

    ```
    # virsh dumpxml hassos | grep "mac address" | awk -F\' '{ print $2}'
    ```

    If you want the VM to restart automatically at boot time, do the following:

    ```
    # virsh autostart hassos
    ```

