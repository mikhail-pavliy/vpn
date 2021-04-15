# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"
  #config.vbguest.auto_update = false

  config.vm.provider "virtualbox" do |v|
    v.memory = 512
    v.cpus = 1
  end

  config.vm.define "server-ovpn" do |server|
    server.vm.network "private_network", ip: "10.10.10.10"
    server.vm.hostname = "server-ovpn"
  end

  config.vm.define "client-ovpn" do |client|
    client.vm.network "private_network", ip: "10.10.10.20"
    client.vm.hostname = "client-ovpn"
  end

  end
