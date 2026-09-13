# Homelab Cluster Configuration with environment variable overrides
# Sizing defaults to heavy nodes suitable for large workloads (e.g. OpenTelemetry demo, GitLab).
# For resource-constrained hosts (e.g. 16 GB RAM), override via environment variables:
#   LAB_WORKERS=2 LAB_WORKER_RAM=2048 LAB_CP_RAM=4096 vagrant up

CP_MEM       = (ENV['LAB_CP_RAM'] || 6144).to_i
CP_CPUS      = (ENV['LAB_CP_CPUS'] || 4).to_i
WORKER_COUNT = (ENV['LAB_WORKERS'] || 4).to_i
WORKER_MEM   = (ENV['LAB_WORKER_RAM'] || 4096).to_i
WORKER_CPUS  = (ENV['LAB_WORKER_CPUS'] || 2).to_i

Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"

  # Control Plane Node
  config.vm.define "debian1" do |node|
    node.vm.hostname = "debian1"
    node.vm.network "private_network", ip: "192.168.56.11"

    node.vm.provider "virtualbox" do |vb|
      vb.name = "debian1"
      vb.memory = CP_MEM
      vb.cpus = CP_CPUS
    end
  end

  # Worker Nodes (1 to WORKER_COUNT, up to 4)
  (1..WORKER_COUNT).each do |i|
    node_name = "debian#{i + 1}"
    node_ip   = "192.168.56.#{11 + i}"

    config.vm.define node_name do |node|
      node.vm.hostname = node_name
      node.vm.network "private_network", ip: node_ip

      node.vm.provider "virtualbox" do |vb|
        vb.name = node_name
        vb.memory = WORKER_MEM
        vb.cpus = WORKER_CPUS
      end
    end
  end
end