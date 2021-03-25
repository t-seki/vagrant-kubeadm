# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
    vb.cpus = 2
  end

  config.vm.define "host1" do |node|
    node.vm.hostname = "host1"
    node.vm.network "private_network", ip: "172.16.10.11"
  end
  config.vm.define "host2" do |node|
    node.vm.hostname = "host2"
    node.vm.network "private_network", ip: "172.16.10.12"
  end

  config.vm.provision "shell", :inline => <<-EOT
    yum update -y
  EOT
end
