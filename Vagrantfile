# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
    :web => {
        :box_name => "almalinux/9",
        :cpus => 1,
        :memory => 512,
        :ip_addr => '192.168.50.10',
        :playbook => "ansible/syslog-client.yml",
    },
    :log => {
        :box_name => "almalinux/9",
        :cpus => 1,
        :memory => 512,
        :ip_addr => '192.168.50.15',
        :playbook => "ansible/log-server.yml",
    },
    :elk => {
        :box_name => "ubuntu/focal64",
        :cpus => 2,
        :memory => 4096,
        :ip_addr => '192.168.50.20',
        :playbook => "ansible/elk-server.yml",
    },
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
      config.vm.synced_folder ".", "/vagrant", disabled: true
      config.vm.define boxname do |box|
          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s
          box.vm.network "private_network", ip: boxconfig[:ip_addr]
          box.vm.provider :virtualbox do |vb|
            vb.name = boxname.to_s
            vb.memory = boxconfig[:memory]
            vb.cpus = boxconfig[:cpus]
          end
          box.vm.provision "ansible" do |ansible|
            ansible.playbook = boxconfig[:playbook]
          end
      end
  end
end