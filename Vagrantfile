# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "opensuse/Tumbleweed.x86_64"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.

  # if DHCP available on bridged network you can use this and
  # answer the prompt for which interface to use on `vagrant up`
  config.vm.network "public_network" 

  # non-DHCP network and specify the string matching the interface
  # config.vm.network "public_network", ip: "192.168.88.5", bridge: [
  #   "en6: USB 10/100/1000 LAN",
  #   "en0: Wi-Fi"
  # ]

  config.vm.provider "virtualbox" do |vb|
  #  # Display the VirtualBox GUI when booting the machine
  #  vb.gui = true
  
    # Customize the amount of memory on the VM:
    vb.cpus = "2"
    vb.memory = "2048"
  end

  # Enable provisioning with a shell script. Additional provisioners such as
  # Ansible, Chef, Docker, Puppet and Salt are also available. Please see the
  # documentation for more information about their specific syntax and use.

  config.vm.provision "ansible_local" do |ansible|
    ansible.playbook = "provision.yml"
    ansible.limit = "nodes"
    ansible.verbose = false
    ansible.config_file = "ansible.cfg"
    ansible.inventory_path = "hosts"
    ansible.galaxy_command = "sudo ansible-galaxy collection install community.general"
  end

end
