# Module project

I am using Oracle VM VirtualBox to make three virtual computers for a small server environment. One will be assigned to be master, and other two are minions. The operating system for the virtual computers will be Debian12 Bookworm(64).
The idea is to make a script which will install these computers, and Salt-master and Salt-minion daemons automatically. By following this report, you should be able to duplicate every single phase to make your own small virtual environment!

I use Vagrant with my hostOS, Windows. To control my virtual computers for commands, I use Vagrant SSH in Windows PowerShell.
To download and install VirtualBox and Vagrant, visit https://www.virtualbox.org/wiki/Downloads & https://developer.hashicorp.com/vagrant/install.

## Installing the VM:s, Salt-master and Salt-minion 12:45-13:50

First task is to make a new project folder for the operating system, which will contain the Vagrantfile used to install and start the Virtual Machines. This happens in my user's folder (still in HostOS).

    mkdir moduleproject
    cd moduleproject

Secondly download the image for the GuestOS used in the Virtual Machines

    vagrant init debian/bookworm64

Then to make the script for installing the three computers, Salt-master and Salt-minion and assign them as master and minions. More tips https://terokarvinen.com/2023/salt-vagrant/#infra-as-code---your-wishes-as-a-text-file.
For better understanding of the repository downloads necessary for installing Salt, visit https://saltproject.io/blog/salt-project-package-repo-migration-and-guidance/.

    # This opens notepad to modify the vagrantfile
   
    notepad vagrantfile

    # Script (Starting from next line of text)
   
    # -*- mode: ruby -*-
    # vi: set ft=ruby :
    # Copyright 2014-2023 Tero Karvinen http://TeroKarvinen.com

    $minion = <<MINION
    sudo apt-get update
    sudo apt-get -qy install curl
    sudo mkdir -p /etc/apt/keyrings
    sudo curl -fsSL https://packages.broadcom.com/artifactory/api/security/keypair/SaltProjectKey/public | sudo tee /etc/apt/keyrings/salt-archive-keyring-2023.pgp
    echo "deb [signed-by=/etc/apt/keyrings/salt-archive-keyring-2023.pgp arch=amd64] https://packages.broadcom.com/artifactory/saltproject-deb/ stable main" | sudo tee /etc/apt/sources.list.d/salt.list
    sudo apt-get update
    sudo apt-get -qy install salt-minion
    echo "master: 192.168.12.3">/etc/salt/minion
    sudo systemctl restart salt-minion.service
    echo "See also: https://terokarvinen.com/2023/salt-vagrant/"
    MINION

    $master = <<MASTER
    sudo apt-get update
    sudo apt-get -qy install curl
    sudo mkdir -p /etc/apt/keyrings
    sudo curl -fsSL https://packages.broadcom.com/artifactory/api/security/keypair/SaltProjectKey/public | sudo tee /etc/apt/keyrings/salt-archive-keyring-2023.pgp
    echo "deb [signed-by=/etc/apt/keyrings/salt-archive-keyring-2023.pgp arch=amd64] https://packages.broadcom.com/artifactory/saltproject-deb/ stable main" | sudo tee /etc/apt/sources.list.d/salt.list
    sudo apt-get update
    sudo apt-get -qy install salt-master
    sudo systemctl enable salt-master.service
    echo "See also: https://terokarvinen.com/2023/salt-vagrant/"
    MASTER

    Vagrant.configure("2") do |config|
	      config.vm.box = "debian/bookworm64"

	      config.vm.define "tminion1" do |tminion1|
	        	tminion1.vm.provision :shell, inline: $minion
	        	tminion1.vm.network "private_network", ip: "192.168.12.100"
	        	tminion1.vm.hostname = "tminion1"
      	end

        config.vm.define "tminion2" do |tminion2|
	        	tminion2.vm.provision :shell, inline: $minion
          	tminion2.vm.network "private_network", ip: "192.168.12.102"
	        	tminion2.vm.hostname = "tminion2"
    	end

    	config.vm.define "tmaster", primary: true do |tmaster|
        		tmaster.vm.provision :shell, inline: $master
	        	tmaster.vm.network "private_network", ip: "192.168.12.3"
        		tmaster.vm.hostname = "tmaster"
    	end
     end

Next step was to get the machines up and running, test network and that Salt was installed properly.

First the startup. This took a few minutes, since the script installed and updated packages.

    vagrant up

Secondly tested connections and salt installations. With `ping` I tested connections from minions to master. To exit SSH, I used `exit` in the commandline.

    vagrant ssh tminion1
    ping 192.168.12.3
    sudo systemctl status salt-minion.service

    vagrant ssh tminion2
    ping 192.168.12.3
    sudo systemctl status salt-minion.service

    vagrant ssh tmaster
    sudo systemctl status salt-master.service

The result should look something like this

![Add file: Upload](pictures/p2.png)

My master daemon was in inactive status, so I started it with next command. After this it was active.

    sudo systemctl start salt-master.service

After all this was done, it was time to accept keys for the minion-master -Salt connection with master-computer and test the connection

    sudo salt-key --list all
    sudo salt-key -A
    
    sudo salt '*' test.ping

Everything went good, and resulted in True -answers from minions.

![Add file: Upload](pictures/p3.png)

## Installing Apache2 for minion1


### Sources

- Karvinen, T. 2023. Ready made vagrantfile for three computers. https://terokarvinen.com/2023/salt-vagrant/#infra-as-code---your-wishes-as-a-text-file
- Salt repository. https://saltproject.io/blog/salt-project-package-repo-migration-and-guidance/
- Vagrant. https://developer.hashicorp.com/vagrant/install
- VirtualBox. https://www.virtualbox.org/wiki/Downloads
