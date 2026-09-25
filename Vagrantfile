# NOTE: This Vagrantfile is currently configured to run heavy nodes (suitable for deploying heavy workloads like the OpenTelemetry demo).
# Adjust RAM and CPU values for each VM below if you have limited host resources.
Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"

  config.vm.define "debian1" do |node|
    node.vm.hostname = "debian1"
    node.vm.network "private_network", ip: "192.168.56.11"

    node.vm.provider "virtualbox" do |vb|
      vb.name = "debian1"
      vb.memory = 4608
      vb.cpus = 4
    end
  end

  config.vm.define "debian2" do |node|
    node.vm.hostname = "debian2"
    node.vm.network "private_network", ip: "192.168.56.12"

    node.vm.provider "virtualbox" do |vb|
      vb.name = "debian2"
      vb.memory = 2048
      vb.cpus = 2
    end
  end

  config.vm.define "debian3" do |node|
    node.vm.hostname = "debian3"
    node.vm.network "private_network", ip: "192.168.56.13"

    node.vm.provider "virtualbox" do |vb|
      vb.name = "debian3"
      vb.memory = 2048
      vb.cpus = 2
    end
  end

  config.vm.define "debian4" do |node|
    node.vm.hostname = "debian4"
    node.vm.network "private_network", ip: "192.168.56.14"

    node.vm.provider "virtualbox" do |vb|
      vb.name = "debian4"
      vb.memory = 8192
      vb.cpus = 2
    end
  end

  config.vm.define "debian5" do |node|
    node.vm.hostname = "debian5"
    node.vm.network "private_network", ip: "192.168.56.15"

    node.vm.provider "virtualbox" do |vb|
      vb.name = "debian5"
      vb.memory = 2048
      vb.cpus = 2
    end
  end
end