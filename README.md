本项目是参考(《Installing a Windows Virtual Machine in a Linux Docker Container》)[https://medium.com/axon-technologies/installing-a-windows-virtual-machine-in-a-linux-docker-container-c78e4c3f9ba1] 在Linux宿主机上运行windows容器的方案。

<br/>
==========================================================
<br/>
<br/>
<br/>
<br/>
<br/>
# Installing a Windows Virtual Machine in a Linux Docker Container

## A Step-by-Step Guide to Containerizing a Windows Virtual Machine — with RDP Access — on a Linux Docker Platform with KVM Hypervisor



# Background

Recently, I was tasked with developing a PoC of a lab environment where students can access their pre-installed and pre-configured machines — Linux and Windows — to do necessary training exercises. I wanted to make the access to all machines uniform over HTTP/HTTPS (browser-based). While the web-based access to machines can easily be implemented using a platform like Guacamole, the greater challenge was how to deploy the machines in a way that utilizes system resources — particularly, CPU, RAM, and HDD — efficiently and with speed. It became evident by that point that a technology like Docker containerization is the best way to go. However, that presented other challenges.

Each of Windows and Linux machines have their own containerization requirements — as will be discussed in the next section. Natively, one cannot run Linux and Windows containers simultaneously on the same Docker daemon. After some research, the solution that presented itself as the most viable was to install each Windows machine as a virtual machine inside a single Linux container. From the perspective of Docker daemon, all containers are Linux-based. However, some of those containers run a hypervisor, and on top of which there is a Windows VM. Even though a container with a VM in it takes more disk space than other containers, the efficiency in saving disk space when we have many containerized VMs is still high compared to running un-containerized VMs.

Ultimately, I wanted to access a containerized Windows machine using RDP, and enjoy the full remote desktop access to that machine. Unfortunately, there are not satisfactory detailed tutorials and complete walkthroughs that explain the entire procedure easily and clearly. And I had to face many small challenges along the way. During my research, I have also seen many people — on various technical forums — struggling with such an implementation and expressing their frustration! I hope that this document serves as a complete guide to solving that problem.

# Why Containerizing a VM: Our Use Case

You might be asking yourself why would someone want to install a VM inside a Container? It looks strange at first since the Container layer seems unnecessary and one can simply install the VM directly on the base OS. However, there are different reasons why this can be a solution and a necessary requirement.

Our particular use case involves spinning up multiple identical Windows VMs to be used by different users. Had we wanted a single VM only, then, there would not be any need to containerize it. But since we want to create many identical VMs, we will save tremendous resources (HDD, RAM, and CPU) by containerizing those VMs.

If we compare the scenario where we run a single VM directly on our base OS to scenario of containerizing that VM, we will find out that both will consume similar disk space and other resources. In fact, every time we want to run the containerized VM, we will do two steps: run the container and then power on the VM. The following diagram shows those two scenarios; a direct VM occupies 30GB on the HDD, while the Docker Image occupies 35GB. Not much benefit — in terms of saving system resources — is achieved here.

![](https://miro.medium.com/v2/resize:fit:700/1*1Ce92YDQd0vEkU6pZIjBZQ.png)

However, what happens if we want to run **6** copies of our intended VMs? We will have to create **6** copies of that VM where each occupies the exact same disk space as the original one. Thus, if the original VM is 30GB, having **6** copies will occupy **180GB** on the HDD.

![](https://miro.medium.com/v2/resize:fit:700/1*y22BdVrBfUe2iqgmeirg4g.png)

This changes dramatically when we containerize each of those identical VMs. This is the added-value of Docker containerization technology. And it owes its value to how Docker differentiates between two main concepts: images and containers. Images are read-only and form the base of containers. Containers created from the same Image share the same read-only core (i.e., the image), while each container adds its own read-write layer that interacts with the read-only image. For more discussion on the difference between Images and Containers, please check this document: <[click here](https://docs.docker.com/storage/storagedriver/#container-and-layers)>

If we take the **35GB** Docker Image, in our example, and creates **6** Containers from it, each Container will create its own read-write layer through which will access the read-only image. At the time of creation, that R/W layer has **0** size; however, as the user starts interacting with the Container — doing things like powering on the VM — that layer’s size starts increasing. And if we assume that the all dynamic changes in a single layer have accumulated a size of **10GB**, that means all **6** containers have added a total of **60BG** on top of the original **35GB** of the image.

![](https://miro.medium.com/v2/resize:fit:700/1*DOjssfkgbMRy836QJX8p7A.png)

# Challenges

## Challenge 1 Windows containers on Windows platform, and Linux containers on Linux platform

One of the biggest obstacles you face with Docker, and containerization in general, is that you cannot have Linux and Windows containers running simultaneously on the same platform (i.e. same Docker daemon). The reason for this is that Docker is an OS-Level Virtualization; meaning, its main function is to *contain* and *isolate* applications as they run on an Operating System. The Docker daemon provides each container with any necessary kernel-level properties so that the containerized application can run. Because of that, containers running Linux services/applications need to run on a Linux platform, and containers running Windows services/application need to run on a Windows platform.

The Windows Docker Desktop has the feature of providing **Linux Subsystem**; and in this case, running Linux container can ultimately run on Windows. However, we should note that if this feature is enabled, only Linux containers can run while Windows containers cannot. One has to switch off this feature to run Windows containers; and in this scenario, Linux containers cannot run. It is still not possible run both Linux and Windows containers simultaneously on the same platform.

If one needs to have Linux and Windows containers running simultaneously and communicating with other, a possible solution is to have each group run on their respective platform, then, configure the network routing, NAT, and port-forwarding rules.

## Challenge 2 Windows Docker containers cannot be accessed through RDP or VNC, i.e., no graphical desktop

Even if we decided to have two separate platforms — Windows platform for Windows containers, and Linux platform for Linux containers — with appropriate network configuration, we will face the challenge that Windows containers cannot have desktop environment. This is a fact for all Windows containers. They are designed and built to run services and applications; and they can be accessed using PowerShell/CMD command line interface.

Unlike Linux system where the Desktop environment is an installable service, Windows Desktop comes bundled directly with the OS as shipped by Microsoft. And when it comes to Windows-based containers, Microsoft has published certain images (known as base images) which form the base of any Windows container. Those base images do not come up with a Desktop service; and one does not have the luxury to install it later as an add-on.

For more information about Windows containers/images, <[Click Here](https://docs.microsoft.com/en-us/virtualization/windowscontainers/about/)>

# Architecture Overview

Our ultimate goal here is to have a fully running Windows OS, accessible through RDP, and containerized and managed by Docker daemon. And to achieve that, we will have the following:

![](https://miro.medium.com/v2/resize:fit:700/1*qnvlTotZ5tv3jP4rP_ZeDQ.png)

- **The Base Operating System**: it will be the main platform hosting everything else. In our particular example, it will be an *Ubuntu 18.04 Linux machine*.
- **The Docker Daemon**: this is the containerization platform installed on the Base OS. Through Docker, we will create our final “image” out of which we will spawn many containers.
- **A Docker Image with an Operating System**: This OS will be part of every created container, and its main function is to run a hypervisor on which the Windows VM will be running. In our case here, we will use *Ubuntu:18.04* Docker Image (available on Docker Hub).
- **A Hypervisor on the Docker Image**: Inside the Ubuntu Docker Image, we will also have a Hypervisor which will allow us to install the Windows VM later. In our particular case, we will use *KVM-QEMU* hypervisor.
- **The Windows Virtual Machine**: this is the machine we are going to access at the end through RDP. In our example, we will use a pre-packaged *Windows 10* Vagrant Box available at (https://app.vagrantup.com/peru/boxes/windows-10-enterprise-x64-eval)

# Installing Docker on the Main Platform

The first thing we need to do is to install Docker into our main Operating System. For the sake of this tutorial, our main system is Ubuntu 20.04 (Linux Kernel 5.4.0–40-generic) with 70GB HDD, 4GB RAM, and 2 CPU Cores.

Follow the following the steps to install Docker:

**[1] Update the apt package index and install packages to allow apt to use a repository over HTTPS:**

sudo apt-get updatesudo apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common

**[2] Add Docker’s official GPG key:**

curl -fSSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add   
sudo apt-key fingerprint 0EBFCD88

**[3] Set up the stable Docker’s repository:**

sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

**[4] Update the apt package index:**

sudo apt update

> N**ote**: This is an important step after adding the new repository in Step 3.

**[5] Install the latest version of Docker Engine:**

sudo apt install docker-ce -y

> **Note**: You don’t need to install docker-ce-cli or containerd.io packages since they are installed directly with docker-ce package.

**[6] Start Docker now and configure it to automatically start after a reboot:**

sudo systemctl start dockersudo systemctl enable docker

# Building the Right Docker Image

Now that Docker is installed, we can start building the image that will be the base for our Container on which we will have the VM. The first section below explains how to build this image manually without using a Dockerfile. Then, in the second section, I will explain how to automate building the image using a Dockerfile.

However, before building the image, we need to check if our system supports virtualization. Since our Container will run a hypervisor, it will not work unless the main platform supports virtualization. Otherwise, we will face an error later on when trying to install the VM. We can run the following command:

sudo egrep -c '(vmx|svm)' /proc/cpuinfo

If the output is a number greater than 0, then, you are good to proceed further. Otherwise, you need to make sure virtualization (VT-x) is enabled in the BIOS settings. If your main platform is itself a virtual machine, make sure VT-x is enabled through the virtualization software.

- Enable VT-x in VMWare Workstation
- Enable VT-x in Virtualbox

## Building the Image without a Dockerfile

**[1] Pull the core Docker image ubuntu:18.04:**

sudo docker pull ubuntu:18.04

> ***Note***: to verify that the image has been added successfully, issue the following command:
> 
> sudo docker image ls

**[2] Run a Container (with the name ubuntukvm) from the Image ubuntu:18:04 with some privileged parameters:**

sudo docker run --privileged -it --name ubuntukvm --device=/dev/kvm --device=/dev/net/tun -v /sys/fs/cgroup:/sys/fs/cgroup:rw --cap-add=NET_ADMIN --cap-add=SYS_ADMIN ubuntu:18.04 /bin/bash

> Since we will install a hypervisor (QEMU-KVM) in this container, we need to run it with certain parameters as follows:  
> **— device=/dev/kvm** will map the device /dev/kvm in the main OS inside the Container.  
> **— device=/dev/net/tun** will map the device /dev/net/tun in the main OS inside the Container.  
> **-v /sys/fs/cgroup:/sys/fs/cgroup:rw** will map the directory /sys/fs/cgroup in the main OS inside the Container, and the Container will have read-write permissions on that directory.  
> **— cap-add=NET_ADMIN** will add network admin capabilities to the Container.  
> **— cap-add=SYS_ADMIN** will add system admin capabilities to the Container.

Once the command runs successfully, you should be inside the Container with a shell prompt:

root@<container_id>:/#

**[3] Inside the Container, update the apt package index:**

root@<container_id>:/# apt-get update -y

**[4] Inside the Container, install the hypervisor package QEMU-KVM and Libvirt:**

root@<container_id>:/# apt-get install -y qemu-kvm libvirt-daemon-system libvirt-dev

> You don’t have to install libvirt-clients and bridge-utils as they would already be installed along with libvirt-daemon-sys.
> 
> The libvirt-dev is an important package especially to run Vagrant Boxes on.

**[5] Change group ownership of /dev/kvm:**

root@<container_id>:/# chown root:kvm /dev/kvm

> **Note**: the device /dev/kvm must be owned by the group kvm, and any user who needs to run VMs need to be part of the kvm group.

**[6] Start the Libvirt services:**

root@<container_id>:/# service libvirtd startroot@<container_id>:/# service virtlogd start

**[7] Install the Linux Image package which contains any necessary Kernel modules:**

root@<container_id>:/# apt-get install -y linux-image-$(uname -r)

> **Note**: this is an important step. There are certain modules (e.g., ip_tables and ip6_tables) that are needed at a later stage; and if they are missing, an error message would be generated.

**[8] Install the curl package (it is used to download Vagrant application):**

root@<container_id>:/# apt-get install curl -y

**[9] Install the net-tools package (it provides ipconfig utility):**

root@<container_id>:/# apt-get install net-tools -y

**[10] Download and run the latest version Vagrant application:**

root@<container_id>:/# apt-get install jq -yroot@<container_id>:/# vagrant_latest_version=$(curl -s https://checkpoint-api.hashicorp.com/v1/check/vagrant  | jq -r -M '.current_version')root@<container_id>:/# echo $vagrant_latest_versionroot@<container_id>:/# curl -O https://releases.hashicorp.com/vagrant/$(echo $vagrant_latest_version)/vagrant_$(echo $vagrant_latest_version)_x86_64.debroot@<container_id>:/# dpkg -i vagrant_$(echo $vagrant_latest_version)_x86_64.deb

> **Note 1**: The above commands perform the following actions:  
> 
> - Install the JSON Query parser tool, jq, which will be used in the next command.  
> - Get the Vagrant latest version value and store it in the environment variable vagrant_latest_version.  
> - Download the latest version of Vagrant package.  
> - Install the downloaded Vagrant package.
> 
> **Note 2**: It is very important and critical that you download and install Vagrant in this method. Do **NOT** get it from the Ubuntu repository (or any other Linux repositories, like Red Hat’s) using the command apt-get install vagrant. The reason for this is that **WinRM** library is not shipped with Vagrant packages provided by Linux distribution, and is shipped natively with the official package. WinRM library is needed to run Windows Vagrant boxes.

**[11] Install the Vagrant Libvirt plugin:**

root@<container_id>:/# vagrant plugin install vagrant-libvirt

**[12] Download and install Windows10 Vagrant box:**

root@<container_id>:/# mkdir /win10root@<container_id>:/# cd /win10root@<container_id>:/win10# vagrant init peru/windows-10-enterprise-x64-evalroot@<container_id>:/win10# VAGRANT_DEFAULT_PROVIDER=libvirt vagrant up

> the vagrant init command will download a **Vagrantfile** which contains the instructions of building the Vagrant box.
> 
> the vagrant up command will build the box. Please note that this command takes some time. The particular Vagrant box we are downloading here (peru/windows-10-enterprise-x64-eval) has a size of 5.62 GB.
> 
> once the above command finishes execution, type the following command which will attempt to access the box over RDP. Even though it will fail (since there is no RDP client installed in the Container), we will get the IP address of the Vagrant box:

root@< container_id >:/win10# vagrant rdp==> default: Detecting RDP info…  
 default: Address: 192.168.121.68:3389  
 default: Username: vagrant  
==> default: Vagrant will now launch your RDP client with the connection parameters  
==> default: above. If the connection fails, verify that the information above is  
==> default: correct. Additionally, make sure the RDP server is configured and  
==> default: running in the guest machine (it is disabled by default on Windows).  
==> default: Also, verify that the firewall is open to allow RDP connections.  
An appropriate RDP client was not found. Vagrant requires either  
`xfreerdp` or `rdesktop` in order to connect via RDP to the Vagrant  
environment. Please ensure one of these applications is installed and  
available on the path and try again.

**[13] Set up iptables port forwarding rules:**

This is an important step if you want to access RDP port on Vagrant box from out the Container. By default, the Vagrant application configures firewall rules to allow access only from within the Container to the Vagrant box and vice versa. Machines outside the Container do not have any access to Vagrant box. We would like to set up the rules in such a way to allow our main OS (Ubuntu) to access the Vagrant box on RDP. The following diagram logically illustrates this:

![](https://miro.medium.com/v2/resize:fit:700/1*GmUzh709nt0b3O7n2RHRHw.png)

Add the following rules to NAT/Port Forward connections from the main OS to the container on port 3389 to be forwarded to the Vagrant Box on port 3389:

root@<container_id>:/# iptables -A FORWARD -i eth0 -o virbr1 -p tcp --syn --dport 3389 -m conntrack --ctstate NEW -j ACCEPTroot@<container_id>:/# iptables -A FORWARD -i eth0 -o virbr1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPTroot@<container_id>:/# iptables -A FORWARD -i virbr1 -o eth0 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPTroot@<container_id>:/# iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 3389 -j DNAT --to-destination 192.168.121.68root@<container_id>:/# iptables -t nat -A POSTROUTING -o virbr1 -p tcp --dport 3389 -d 192.168.121.68 -j SNAT --to-source 192.168.121.1

After that, we should remove the rules that reject all traffic to/from virb1 interface; these rules take precedent over our newly inserted rules:

root@<container_id>:/# iptables -D FORWARD -o virbr1 -j REJECT --reject-with icmp-port-unreachableroot@<container_id>:/# iptables -D FORWARD -i virbr1 -j REJECT --reject-with icmp-port-unreachableroot@<container_id>:/# iptables -D FORWARD -o virbr0 -j REJECT --reject-with icmp-port-unreachableroot@<container_id>:/# iptables -D FORWARD -i virbr0 -j REJECT --reject-with icmp-port-unreachable

> if you mess up the iptables, or if the communication is problematic later, you may want to clear up all tables and then add the rules (mentioned above) on a clean slate. To clear the iptables, issue the following:

root@<container_id>:/# sudo iptables-save > $HOME/firewall.txt  
root@<container_id>:/# sudo iptables -X  
root@<container_id>:/# sudo iptables -t nat -F  
root@<container_id>:/# sudo iptables -t nat -X  
root@<container_id>:/# sudo iptables -t mangle -F  
root@<container_id>:/# sudo iptables -t mangle -X  
root@<container_id>:/# sudo iptables -P INPUT ACCEPT  
root@<container_id>:/# sudo iptables -P FORWARD ACCEPT  
root@<container_id>:/# sudo iptables -P OUTPUT ACCEPT

**[14] Commit all Changes to Create a New Image:**

Up to this point, we have a fully running Container with the desired Windows VM. However, we cannot transfer or store that Container. In addition, we cannot create multiple copies of this Container without going through all the steps we have done so far. For this reason, we need to commit the changes to a new Docker Image. The Image can be transferred or stored. Multiple Containers can be created — instantiated — almost immediately.

To commit the changes to a new Image, we need first to exit the Container:

root@<container_id>:/win10# exit$ sudo docker ps -a

Take note of the Container ID; and then, issue the following command:

$ sudo docker commit <container_id> ubuntukvm

> Note 1: You can substitute the name “ubuntukvm” with any name you like. This will be the name of the new Image.

## Building the Image Using a Dockerfile

Instead of building the Image in a manual way — as shown in the previous section, we can automate the whole process using a Dockerfile.

**[1] Prepare the Dockerfile:**

In a new directory, create a Dockerfile (with the name *Dockerfile*), and write the following commands in it. Mostly, they are the same commands we have executed individually in the previous section:

**[2] Prepare a Startup Shell Script (startup.sh):**

This file will be copied to the Image and will run automatically every time you instantiate a Container from that Image. The script will assign certain permissions and startup the necessary services. In addition, it will create the iptables rules that will port-forward RDP traffic.

**[3] Build the Container from the Docker file:**

sudo chmod +x startup.shsudo docker build -t ubuntukvm:latest -f Dockerfile .

**[4] Instantiate a Container and Run it:**

sudo docker run --privileged -it --name kvmcontainer1 --device=/dev/kvm --device=/dev/net/tun -v /sys/fs/cgroup:/sys/fs/cgroup:rw --cap-add=NET_ADMIN --cap-add=SYS_ADMIN ubuntukvm bash

# Testing the RDP Access

By now, we should be able to access the RDP service on the Windows Vagrant box by connecting to the IP address of the Docker container. To test that port 3389/tcp (RDP) is reachable from the main OS, we will use a simple Nmap command.

First, if you are inside the Docker container, press Ctrl+p+q to put the Container in the background while running; this should return you to the main OS terminal prompt:

root@<container_id>:/win10# <Ctrl+p+q>$ sudo nmap -Pn -p 3389 172.17.0.2

Next, we need to install an RDP client for Linux. A popular one is RDesktop:

sudo apt-get install rdesktop

Finally, we can access the Windows VM:

sudo rdesktop 172.17.0.2

The Windows Vagrant box that we have installed has two built-in accounts:

1. Username: **vagrant** Password: **vagrant**
2. Username: **Administrator** Password: **vagrant**

![](https://miro.medium.com/v2/resize:fit:700/1*bu2UhBY3twBJigfxWfurdw.png)

# Conclusion

I hope this post has been a comprehensive guide to containerize a virtual machine. There are different advantages of running a VM in a Container; one of them is running multiple Containers simultaneously. You can automatically build the desired image using a Dockerfile, or you can build it manually by running each command individually. We have covered both ways in this post.
