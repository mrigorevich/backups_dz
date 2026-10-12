# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :srv => {
        :box_name => "ubuntu/jammy64",
        :vm_name => "backup-server",
        :ip => "192.168.11.150",
       },
  :client => {
		:box_name => "ubuntu/jammy64",
		:vm_name => "client",
		:ip => "192.168.11.160",
	   }
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, settings|
    config.vm.define boxname do |node|
      node.vm.box = settings[:box_name]
      node.vm.hostname = settings[:vm_name]
	  node.vm.synced_folder ".", "/vagrant", disabled: true
	  node.vm.network "private_network", ip: settings[:ip]
	  if boxname == :srv
		node.vm.disk :disk, name: "bkp_storage", size: "2GB"
	  end
      node.vm.provider "virtualbox" do |vb|
        vb.name = settings[:vm_name]
        vb.memory = "1024"
        vb.cpus = 1
      end
      
      node.vm.provision "shell", inline: <<-SHELL
# включаем парольную аутентификацию
		cat <<EOF > /etc/ssh/sshd_config.d/90-vagrant-password.conf
PasswordAuthentication yes
KbdInteractiveAuthentication yes
EOF
		systemctl restart ssh.service
		
		if [ "$(hostname)" = "backup-server" ]; then
			bkp_disk="/dev/sdc"
			bkp_mount="/var/backup"
			mkdir -p "$bkp_mount"
			mkfs.ext4 -F "$bkp_disk"
			UUID=$(blkid -o value -s UUID "$bkp_disk")
			echo "UUID=$UUID $bkp_mount ext4 defaults,nofail 0 2" >> /etc/fstab
			mount "$bkp_mount"
		fi
		
		timedatectl set-timezone Europe/Moscow
		apt update && apt install -y borgbackup
SHELL
    end
  end
end