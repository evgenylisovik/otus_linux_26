# -*- mode: ruby -*-
# vi: set ft=ruby :
$scriptClient = <<-SCRIPTCLIENT
  yum -y install epel-release.noarch
  yum -y update
  yum -y install borgbackup
  yum -y install sshpass              
  ssh-keygen -f /root/.ssh/id_rsa -N ""
  ssh-keyscan -t ecdsa 192.168.56.11 | cat >> /etc/ssh/ssh_known_hosts
  cat /root/.ssh/id_rsa.pub | sshpass -p "borg" ssh borg@192.168.56.11 'cat >> .ssh/authorized_keys'
  sshpass -p "vagrant" ssh vagrant@192.168.56.11 "sudo chown borg:borg /var/backup/"
  sshpass -p "vagrant" ssh vagrant@192.168.56.11 "sudo rm -r /var/backup/lost+found/"
  sshpass -p "vagrant" ssh vagrant@192.168.56.11 "sudo sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config"
  sshpass -p "vagrant" ssh vagrant@192.168.56.11 "sudo sed -i 's/PubkeyAuthentication no/PubkeyAuthentication yes/' /etc/ssh/sshd_config"
  sshpass -p "vagrant" ssh vagrant@192.168.56.11 "sudo systemctl restart sshd"
  BORG_PASSPHRASE='testpassphrase' borg init --encryption=repokey borg@192.168.56.11:/var/backup
  echo -e '[Unit]\nDescription=Borg Backup\n\n[Service]\nType=oneshot\n\nEnvironment="BORG_PASSPHRASE=testpassphrase"\nEnvironment=REPO=borg@192.168.56.11:/var/backup/\nEnvironment=BACKUP_TARGET=/etc\nExecStart=/bin/borg create \\\n   --stats             \\\n   ${REPO}::etc-{now:%%Y-%%m-%%d_%%H:%%M:%%S} ${BACKUP_TARGET}\nExecStart=/bin/borg check ${REPO}\nExecStart=/bin/borg prune \\\n     --keep-minutely 60        \\\n      --keep-hourly 24     \\\n    --keep-daily  90      \\\n    --keep-monthly 12     \\\n   --keep-yearly 1   \\\n${REPO}\nStandardOutput=syslog\nStandardError=syslog\nSyslogIdentifier=borg' >  /etc/systemd/system/borg-backup.service 
  echo -e '[Unit]\nDescription=Borg Backup\n\n[Timer]\nOnUnitActiveSec=5min\nUnit=borg-backup.service\n\n[Install]\nWantedBy=multi-user.target' >  /etc/systemd/system/borg-backup.timer
  systemctl enable rsyslog
  touch /var/log/borg
  touch /etc/rsyslog.d/borg.conf
  echo 'if \$programname == "borg" then /var/log/borg' | sudo tee /etc/rsyslog.d/borg.conf
  systemctl restart rsyslog
  touch /etc/logrotate.d/borg.conf
  echo -e '/var/log/borg {\n  hourly\n  rotate 3\n  size 10K\n  compress\n  delaycompress\n}' > /etc/logrotate.d/borg.conf
  systemctl daemon-reload
  systemctl enable borg-backup.service borg-backup.timer
  systemctl start borg-backup.service
  systemctl start borg-backup.timer  
  SCRIPTCLIENT
$script = <<-SCRIPT
  yum -y install epel-release.noarch
  yum -y update
  yum -y install borgbackup
  adduser borg
  echo "borg" | sudo passwd borg --stdin
  mkdir /var/backup
  echo "y" | mkfs.ext4 /dev/sdb
  sudo -i
  uuid=$(blkid | awk '$1=/sdb/ {print $2}' | cut -c 1-5,7-42)
  echo $uuid /var/backup   ext4     defaults   0 0 >> /etc/fstab
  mkdir /home/borg/.ssh
  touch /home/borg/.ssh/authorized_keys 
  chown -R borg:borg /home/borg/
  chmod 700 .ssh
  chmod 600 .ssh/authorized_keys
  sed -i 's/PubkeyAuthentication yes/PubkeyAuthentication no/' /etc/ssh/sshd_config
  sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
  systemctl restart sshd
  reboot  
  SCRIPT
machines = [
    {
       :name => "backupServer",
       :eth0 => "192.168.56.11",
       :mem => "1024",
       :cpu => "2",
       :script => $script
    },   

   {
        :name => "client",
        :eth0 => "192.168.56.10",
        :mem => "1024",
        :cpu => "1",
        :script => $scriptClient
    }
 
]

Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  config.vm.provider "virtualbox" do |vb|    
 #   vb.customize ["modifyvm", :id, "--memory", "1024"]
  end
  machines.each do |options|
    config.vm.define options[:name] do |config|
      config.vm.hostname = options[:name]
      config.vm.provider "virtualbox" do |v|
        v.customize ["modifyvm", :id, "--name", options[:name]]
        v.customize ["modifyvm", :id, "--memory", options[:mem]]
        v.customize ["modifyvm", :id, "--cpus", options[:cpu]]
        if config.vm.hostname == "backupServer" 
          v.customize ['createhd', '--filename', './sata0.vdi', '--variant', 'Fixed', '--size', 2000]
          v.customize ['storagectl', :id, '--name', 'SATA', '--add', 'sata' ]
          v.customize ['storageattach', :id,  '--storagectl', 'SATA', '--port', 0, '--device', 0, '--type', 'hdd', '--medium', './sata0.vdi']
        end
      end
      config.vm.network :private_network, ip: options[:eth0]
      config.vm.provision "shell", inline: options[:script]
      end      
    end
  end  

