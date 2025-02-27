# Installing Home Assistant 

This is the procedure I used to install Home Assistant (HA) on a fresh installation of Ubuntu Server 24.04 on a Lenovo ThinkCenter M900 mini computer.

A monitor and a keyboard are required to be connected to the computer both for the installation of Ubuntu and some steps related to modifying the BIOS in the procedure below. Also, it is neccessary to have the computer connected through an Ethernet cable to your network. The HA server **MUST** be connected to the network through an Ethernet port to run properly.

It is expected that the OS has been installed with the openSSH optional package. Also make sure to unselect the use of the LVM partition scheme in the Ubuntu installation process, as it is not necessary for that kind of server, and simplify the management of the disk's space. A good tutorial on installing Ubuntu server 24.04 is available [here](https://www.linuxtechi.com/how-to-install-ubuntu-server/).

0. Connect to the server using a terminal screen from your PC or MAC workstation/laptop. Replace `[USERNAME]` with the username created when installing the OS, and `[IP ADDRESS]` with the server IP address. You can find it on the DHCP server (usually located on the main router of the network):

    ```
    $ ssh [USERNAME]@[IP ADDRESS]
    ```

1. Update the system: 

    ```
    $ sudo apt update
    $ sudo apt upgrade
    ```

2. **Optional**: If you have a defined rsa key to simplify access to the server from your PC or MAC workstation/laptop, do the following:

     from another terminal session on your workstation/laptop, copy the .ssh credentials for login without password to the HA server: 

   ```
   $ scp .ssh/id_rsa.pub [USERNAME]@[IP ADDRESS]:.ssh/authorized_keys`
   ```
    On that terminal, test if you credentials were properly copied, loging back to the HA server with ssh, as per step 0. That must works without entering your password.

3. Back on the HA server, adjust timezone. But before, we use sudo to become a privileged process (The prompt becomes a '#' instead of '$'). You will be requested to enter your password: 

   ```
   $ sudo su
   [sudo] password for username: ******
   # timedatectl set-timezone EST
   ```

4. Lock root login:

   ```
   # passwd -l root
   ```

5. **Optional**: Secure SSH authentication ONLY IF YOU COPIED YOUR RSA CREDENTIALS AT STEP 2:

    ```
    # cd /etc/ssh
    # nano sshd_config
    ```

    Add the following lines

    ```
    PermitRootLogin no
    PermitEmptyPassword no
    PasswordAuthentication no
    ```

    Save file using Ctrl-X and Y. Then restart the ssh daemon:

    ```
    # /etc/init.d/ssh restart
    ```

6. Reboot: 

    ```
    # sync
    # sync
    # reboot
    ```

7. Login to the HA server as per step 0.

8. Check if the CPU can run VMs:

    ```
    $ sudo su
    # egrep -c '(vmx|svm)' /proc/cpuinfo
    8
    ```

    If the answer is 0, you need to go into the BIOS and change it to allow for VMs creation/run. Here is the procedure to do that:

    To enable virtualization on a Lenovo ThinkCentre M900, you can: 

    1. Request a reboot of the HA server as per step 6
    2. Using the connected keyboard, press Enter many times during the Lenovo startup screen until you get the firmware menu
    3. Press F1 to enter the BIOS screens
    4. Navigate to the "Advanced" screen
    5. Press Enter on "CPU Setup"
    6. Select "Intel(R) Virtualization Technology"
    7. Press Enter, choose Enable, and press Enter
    8. Press F10 (Save and Exit)
    9. Select "Yes" to save the changes and allow the system to reboot
    10. Login back to server as per step 0
    11. Check again if you have VM support 

        ```
        $ sudo su
        # egrep -c '(vmx|svm)' /proc/cpuinfo
        8
        ```

9. Install kvm:

   ```
   # apt install -y qemu-kvm \
   libvirt-daemon-system libvirt-clients \
   bridge-utils ovmf virt-manager
   ```
10. Add user to libvirt and kvm (may already been defined):

    ```
    # adduser [USERNAME] libvirt
    # adduser [USERNAME] kvm
    ```

11. Check that the libvirtd daemon is running:

    ```
    # systemctl status libvirtd
    ```

12. Create Home Assistant VM folder and download image:

    ```
    # mkdir -vp /var/lib/libvirt/images/hassos-vm 
    # cd /var/lib/libvirt/images/hassos-vm
    # wget https://github.com/home-assistant/operating-system/releases/download/14.2/haos_ova-14.2.qcow2.xz
    ```

13. Setup storage pool:

    ```
    # virsh pool-create-as --name hassos --type dir --target /var/lib/libvirt/images/hassos-vm
    ```
14. Set up network

    Install net-tools: 
    
    ``` 
    # apt install net-tools
    ```

    Check the Ethernet device name: 
    
    ```
    # ifconfig
    ```
    
     note the entry with the server's IP address. For me it was: `eno1`

    Start the editor:

    ```
    # nano /etc/netplan/00-installer-config.yaml
    ```

    Modify the file to get a result similar to the following (replace `eno1` with your own Ethernet device name):

    ```
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
    # chmod 600 /etc/netplan/00-installer-config.yaml
    ```

    This configuration is good for a DHCP based network.

    You may have to remove the file `50-cloud-init.yaml` if present in the folder:

    ```
    # rm /etc/netplan/50-cloud-init.yml
    ```

    It's now time to generate the files and apply them to the system:

    ```
    # netplan generate
    # netplan apply
    ```

    With the last command, you will loose the connection with the server. Look back at the DHCP server to find a new device appearing with a new address, and reconnect to it:

    ```
    $ ssh [USERNAME]@[NEW IP ADDRESS]
    ```

    (This is the address supplied to the virtual device named br0. This is the bridge for VMs access to the network)

    ```
    $ sudo su
    # brctl show
    ```

    Check that you get the list of bridges. br0 must be listed (there can be other entries present):

    ```
    bridge name	bridge id			STP enabled	interfaces
    br0			8000.00215ec64304	yes		eno1
    ```

15. Install the hassos VM

    Goto the /var/lib/libvirt/images/hassos-vm and decompress the VM image. 

    ```
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

16. This complete the installation of Home Assistant. Look at your DHCP router and identify the IP address of the Home Assistant server (the entry may have the name `homeassistant`). The MAC address retrieved in step 15 could be helpful to identify the proper entry in the DHCP server. Note the IP address. The startup of the Home Assistant server may takes 10-15 minutes to be completed for the first time, so it may takes some times before the IP address appears in the DHCP router.

    In your browser, you can then connect to Home Assistant using the following (depending on your router DNS support capability):

    ```
    http://[IP ADDRESS]:8123

    or

    http://homeassistant.local:8123
    ```
