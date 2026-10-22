Vagrant.configure("2") do |config|
  cpu_count = 2
  memory_mb = 4096
  control_plane_ip = "192.168.57.10"

  config.vm.define "k8s-control-plane" do |node|
    node.vm.box = "bento/ubuntu-24.04"
    node.vm.hostname = "k8s-control-plane"
    node.vm.network "private_network",
      ip: control_plane_ip,
      netmask: "255.255.255.0"

    node.vm.provider "virtualbox" do |virtualbox|
      virtualbox.name = "k8s-control-plane"
      virtualbox.cpus = cpu_count
      virtualbox.memory = memory_mb
    end
  end
end