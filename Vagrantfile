Vagrant.configure("2") do |config|

  config.vm.provider "virtualbox" do |v|
    v.memory = 256
    v.cpus = 2
    v.linked_clone = true
  end

  config.vm.define "app-pxc", primary: true do |app_config|
     app_config.vm.box = "centos/7"
     app_config.vm.box_version = "1710.01"
     app_config.vm.box_check_update = false
     app_config.vm.hostname = "app"
     app_config.vm.provider :virtualbox do |vb|
     	vb.customize ["modifyvm", :id, "--memory", "1024"]
        vb.customize ["modifyvm", :id, "--cpus", "3"]
     end
     app_config.vm.network "private_network", ip: "192.168.80.200"
     app_config.vm.network "forwarded_port", guest: 80, host: 8083
     app_config.hostmanager.enabled = true
     app_config.hostmanager.manage_guest = true
     app_config.hostmanager.ignore_private_ip = false
  end

  (1..3).each do |i|
    config.vm.define "pxc#{i}" do |node|
      node.vm.box = "centos/7"
      node.vm.box_version = "1710.01"
      node.vm.box_check_update = false
      node.vm.hostname = "pxc#{i}"
      node.vm.network "private_network", ip: "192.168.80.#{i}0"
      node.hostmanager.enabled = true
      node.hostmanager.manage_guest = true
      node.hostmanager.ignore_private_ip = false

      node.vm.provision "ansible_local" do |ansible|
        ansible.playbook = "mysql.yml"
        ansible.verbose = true
        ansible.install = true
        ansible.limit = "all"
        ansible.inventory_path = "inventory"
      end

    end
  end

end
