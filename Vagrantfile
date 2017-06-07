# -*- mode: ruby -*-
# vi: set ft=ruby :


Vagrant.configure(2) do |config|

  if Vagrant.has_plugin?("vagrant-proxyconf")
    config.proxy.http = ENV["http_proxy"]
    config.proxy.https = ENV["https_proxy"]
    config.proxy.no_proxy = ENV["no_proxy"]
  else
  end

  #config.vm.provision :hostmanager
  if Vagrant.has_plugin?("vagrant-hostmanager")
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true
    config.hostmanager.manage_guest = true
    config.hostmanager.ignore_private_ip = false
    config.hostmanager.include_offline = true
  else
  end

  config.vm.define "master" do |master|
    master.vm.box = "ubuntu/xenial64"
    master.vm.hostname = "master"
    master.vm.box_url = "ubuntu/xenial64"

    master.vm.network :private_network, ip: "192.168.50.5"
    if Vagrant.has_plugin?("vagrant-hostmanager")
      master.hostmanager.aliases = %w(ranchermaster.localdomain)
      master.hostmanager.manage_guest = false
    else
    end

    master.vm.provider :virtualbox do |v|
      v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      v.customize ["modifyvm", :id, "--memory", 2048]
      v.customize ["modifyvm", :id, "--name", "master"]
    end
  end

  config.vm.define "node1" do |node1|
    node1.vm.box = "ubuntu/xenial64"
    node1.vm.hostname = "node1"
    node1.vm.box_url = "ubuntu/xenial64"

    node1.vm.network :private_network, ip: "192.168.50.10"
    if Vagrant.has_plugin?("vagrant-hostmanager")
      node1.hostmanager.aliases = %w(ranchernode1.localdomain)
    else
    end

    node1.vm.provider :virtualbox do |v|
      v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      v.customize ["modifyvm", :id, "--memory", 1024]
      v.customize ["modifyvm", :id, "--name", "node1"]
    end
  end

  config.vm.define "node2" do |node2|
    node2.vm.box = "ubuntu/xenial64"
    node2.vm.hostname = "node2"
    node2.vm.box_url = "ubuntu/xenial64"

    node2.vm.network :private_network, ip: "192.168.50.11"
    if Vagrant.has_plugin?("vagrant-hostmanager")
      node2.hostmanager.aliases = %w(ranchernode2.localdomain)
    else
    end

    node2.vm.provider :virtualbox do |v|
      v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      v.customize ["modifyvm", :id, "--memory", 1024]
      v.customize ["modifyvm", :id, "--name", "node2"]
    end
  end

  config.vm.provision "shell" do |s|
      user = ENV["USER"]
      ssh_pub_key = File.readlines("/home/#{user}/.ssh/id_rsa.pub").first.strip
      s.inline = <<-SHELL
          echo #{ssh_pub_key} >> /home/ubuntu/.ssh/authorized_keys
      SHELL
  end

  config.vm.box_check_update = true

  config.vm.provision "shell", inline: <<-SHELL
    sudo apt-get update -y
    #sudo apt-get install python-minimal -y
    #sudo apt-get install python-simplejson -y
    #sudo apt-get install aptitude -y
    #sudo apt-get install python-pip -y
    #sudo apt-get install python-httplib2 -y
  SHELL


  # config.vm.provider "virtualbox" do |vb|
  #      config.vm.network "private_network", :type => 'dhcp', :name => 'vboxnet0', :adapter => 2
  # end

    # Define a Vagrant Push strategy for pushing to Atlas. Other push strategies
  # such as FTP and Heroku are also available. See the documentation at
  # https://docs.vagrantup.com/v2/push/atlas.html for more information.
  # config.push.define "atlas" do |push|
  #   push.app = "YOUR_ATLAS_USERNAME/YOUR_APPLICATION_NAME"
  # end

  # Enable provisioning with a shell script. Additional provisioners such as
  # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
  # documentation for more information about their specific syntax and use.
  # config.vm.provision "shell", inline: <<-SHELL
  #   sudo apt-get update
  #   sudo apt-get install -y apache2
  # SHELL



end
