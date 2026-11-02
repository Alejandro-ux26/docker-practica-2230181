Vagrant.configure("2") do |config|

  config.vm.define :clienteUbuntu do |clienteUbuntu|
    clienteUbuntu.vm.box = "ubuntu/jammy64"
    clienteUbuntu.vm.boot_timeout = 600
    clienteUbuntu.vm.network :private_network, ip: "192.168.100.2"
    clienteUbuntu.vm.hostname = "clienteUbuntu"
    clienteUbuntu.vm.provider "virtualbox" do |vb|
      vb.memory = 1024
      vb.cpus = 1
      vb.customize ["modifyvm", :id, "--uartmode1", "disconnected"]
      vb.customize ["modifyvm", :id, "--paravirtprovider", "none"]
    end
  end

  config.vm.define :servidorUbuntu do |servidorUbuntu|
    servidorUbuntu.vm.box = "ubuntu/jammy64"
    servidorUbuntu.vm.boot_timeout = 600
    servidorUbuntu.vm.network :private_network, ip: "192.168.100.3"
    servidorUbuntu.vm.hostname = "servidorUbuntu"
    servidorUbuntu.vm.provider "virtualbox" do |vb|
      vb.memory = 2048
      vb.cpus = 1
      vb.customize ["modifyvm", :id, "--uartmode1", "disconnected"]
      vb.customize ["modifyvm", :id, "--paravirtprovider", "none"]
    end
  end

end
