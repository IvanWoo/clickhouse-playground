# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "perk/ubuntu-2204-arm64"
  config.vm.box_version = "20221201"
  config.vm.provider "qemu"

  config.vm.define "clickhouse" do |instance|
    # So we know which instance we're in easily in termninal
    instance.vm.hostname = "clickhouse"
    # Set the instance name
    instance.vm.provider :virtualbox do |vm|
      vm.name = "clickhouse"
    end
  end

  ## Basic setup
  config.vm.provision "shell", inline: <<-SHELL
    echo "sudo su -" >> .bashrc
  SHELL

  ## Install terminfo
  config.vm.provision "shell", inline: <<-SHELL
    apt-get -yqq update && apt-get install -yqq kitty-terminfo
  SHELL

  ## Install bash utils
  config.vm.provision "shell", inline: <<-SHELL
    apt-get -yqq update && apt-get install -yqq binutils pv
  SHELL

  ## Install clickhouse 
  config.vm.provision "shell", inline: <<-SHELL
    cd /usr/local/bin 
    curl https://clickhouse.com/ | sh
  SHELL
end