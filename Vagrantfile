# -*- mode: ruby -*-
# vi: set ft=ruby :

$expose_port = 8080

# Provisioning shell script
$provisioner = <<SCRIPT
  # Build the apache container
  docker build -t test-apache - < /home/core/share/Dockerfile
  # Switch to root for setting up systemd
  sudo -i
  # Copy the apache service into place
  cp share/apache.service /media/state/units/apache.service
  # Start for the first time (will start automatically on subsequent reboots)
  systemctl restart local-enable.service
SCRIPT

Vagrant.configure("2") do |config|
  config.vm.box = "coreos"
  config.vm.box_url = "http://storage.core-os.net/coreos/amd64-generic/dev-channel/coreos_production_vagrant.box"

  config.vm.provider :virtualbox do |vb, override|
    vb.name = "vagrant-coreos-docker-apache-example"
    # Fix docker not being able to resolve private registry in VirtualBox
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
  end

  # Share this folder so provisioner can access Dockerfile an apache.service
  # This is from coreos/coreos-vagrant, but won't work on Windows hosts
  config.vm.network :private_network, ip: "192.168.2.50"
  config.vm.synced_folder ".", "/home/core/share",
    id: "core", :nfs => true, :mount_options => ['nolock,vers=3,udp']

  # Forward port 8080 from coreos (which is forwarding port 80 of the container)
  config.vm.network :forwarded_port, guest: 8080, host: $expose_port

  # Provision
  config.vm.provision "shell", inline: $provisioner

end
