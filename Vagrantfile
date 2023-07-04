# -*- mode: ruby -*-
# vi: set ft=ruby :

MACHINES = {
  :node1 => '192.168.56.11',
  :node2 => '192.168.56.12',
  :barman => '192.168.56.13'
}

Vagrant.configure(2) do |config|
  config.vm.provider :virtualbox
  config.vm.box_check_update = false
  config.vm.box = 'almalinux/9'
  config.vm.provision "ansible" do |ansible|
    ansible.compatibility_mode = "2.0"
    ansible.playbook = "playbook.yml"
    ansible.limit = "all"
  end
  config.vm.provider "virtualbox" do |vb|
    vb.memory = 512
  end
  MACHINES.each do |boxname, ip_address|
    config.vm.define boxname do |box|
      box.vm.host_name = boxname.to_s
      box.vm.network "private_network", ip: ip_address
    end
  end
end
