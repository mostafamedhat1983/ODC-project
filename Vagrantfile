Vagrant.configure("2") do |config|
  config.hostmanager.enabled = true 
  config.hostmanager.manage_host = true
  
### VM1 ###
config.vm.define "m01" do |m01|
  m01.vm.box = "ubuntu/jammy64"
  m01.vm.hostname = "m01"
m01.vm.network "private_network", ip: "192.168.56.11"
m01.vm.provider "virtualbox" do |vb|
   vb.gui = true
   vb.memory = "600"
 end
end
  

  
  
### VM2 ###
  config.vm.define "m02" do |m02|
    m02.vm.box = "ubuntu/jammy64"
    m02.vm.hostname = "m02"
  m02.vm.network "private_network", ip: "192.168.56.11"
  m02.vm.provider "virtualbox" do |vb|
     vb.gui = true
     vb.memory = "600"
   end
end
  
end
