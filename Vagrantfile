# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  $cmnscript = <<-'SCRIPT'
  echo "MESSAGE: Update yum and install prerequisites"
  yum -y update
  yum -y install apr apr-util bash bzip2 curl krb5-server krb5-libs krb5-devel libcurl libevent libxml2 libyaml ntp zlib openldap openssh openssl openssl-libs perl readline rsync R sed tar zip sshpass
  echo "MESSAGE: Disable Selinux"
  sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config
  echo "MESSAGE: Set NTP parameters"
  ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime
  chkconfig ntpd on
  systemctl enable ntpd
  systemctl start ntpd
  echo "MESSAGE: Disable IPtables"
  iptables -F
  systemctl stop iptables
  systemctl disable iptables
  echo "MESSAGE: Disable Firewall"
  systemctl stop firewalld.service
  systemctl disable firewalld.service
  echo "MESSAGE: Set sysctl parameters"
  echo "kernel.shmall = $(expr $(getconf _PHYS_PAGES) / 2)" >> /etc/sysctl.conf
  echo "kernel.shmmax = $(expr $(getconf _PHYS_PAGES) / 2 \* $(getconf PAGE_SIZE))" >> /etc/sysctl.conf
  echo "$(awk 'BEGIN {OFMT = "%.0f";} /MemTotal/ {print "vm.min_free_kbytes =", $2 * .03;}' /proc/meminfo)" >> /etc/sysctl.conf
  cat << EOF >> /etc/sysctl.conf
kernel.shmmni = 4096
vm.overcommit_memory = 2
vm.overcommit_ratio = 95
net.ipv4.ip_local_port_range = 10000 65535
kernel.sem = 500 2048000 200 4096
kernel.sysrq = 1
kernel.core_uses_pid = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.msgmni = 2048
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.conf.all.arp_filter = 1
net.core.netdev_max_backlog = 10000
net.core.rmem_max = 2097152
net.core.wmem_max = 2097152
vm.swappiness = 10
vm.zone_reclaim_mode = 0
vm.dirty_expire_centisecs = 500
vm.dirty_writeback_centisecs = 100
vm.dirty_background_ratio = 3
vm.dirty_ratio = 10
EOF
  echo "MESSAGE: Set limits"
  cat << EOF > /etc/security/limits.d/20-nproc.conf
* soft nofile 524288
* hard nofile 524288
* soft nproc 131072
* hard nproc 131072
EOF
  echo "MESSAGE: Set xfs parameters"
  sed -i 's|.*/dev/mapper/centos_centos7-root.*|/dev/mapper/centos_centos7-root / xfs rw,nodev,noatime,nobarrier,inode64 0 0|' /etc/fstab
  echo "MESSAGE: Set block device parameters"
  chmod +x /etc/rc.d/rc.local && echo "/sbin/blockdev --setra 16384 /dev/sdb" >> /etc/rc.d/rc.local
  echo "MESSAGE: Disk I/O scheduler parameters"
  grubby --update-kernel=ALL --args="elevator=deadline"
  echo "MESSAGE: Disable Transparent Huge Pages (THP)"
  grubby --update-kernel=ALL --args="transparent_hugepage=never"
  echo "MESSAGE: IPC Object Removal"
  sed -i '/RemoveIPC/s/^#//g' /etc/systemd/logind.conf
  echo "MESSAGE: Set SSH Connection Threshold"
  sed -i 's/.*MaxStartups.*/MaxStartups 10:30:200/' /etc/ssh/sshd_config
  sed -i 's/.*MaxSessions.*/MaxSessions 200/' /etc/ssh/sshd_config
  echo "MESSAGE: Create gpadmin group and user"
  groupadd gpadmin
  useradd -r -m -p $(openssl passwd -1 gpadmin) -g gpadmin gpadmin
  echo 'gpadmin ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers.d/gpadmin
  echo "MESSAGE: Generate ssh key"
  runuser -l gpadmin -c 'ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa'
  runuser -l gpadmin -c 'cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys'
  runuser -l gpadmin -c 'chmod 640 ~/.ssh/authorized_keys'
  sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
  systemctl restart sshd
  echo "MESSAGE: Download and installing Greenplum software"
  wget --no-verbose https://github.com/greenplum-db/gpdb/releases/download/6.14.0/open-source-greenplum-db-6.14.0-rhel7-x86_64.rpm
  rpm -i open-source-greenplum-db-*-rhel7-x86_64.rpm && rm -f open-source-greenplum-db-*-rhel7-x86_64.rpm
  echo "MESSAGE: Change the owner and group of the installed files to gpadmin"
  chown -R gpadmin:gpadmin /usr/local/greenplum*
  chgrp -R gpadmin /usr/local/greenplum*
  cat << EOF >> /etc/hosts
192.168.1.30 gpmaster
192.168.1.40 gpstbmaster
192.168.1.31 gpsegment01
192.168.1.32 gpsegment02
EOF
  SCRIPT

  $masscript = <<-'SCRIPT'
  echo "MESSAGE: Prepare to init GP database"
  mkdir -p /data/master /data/primary /data/mirror
  chown -R gpadmin:gpadmin /data
  echo "source /usr/local/greenplum-db/greenplum_path.sh" >> /home/gpadmin/.bashrc
  echo "export MASTER_DATA_DIRECTORY=/data/master/gpseg-1" >> /home/gpadmin/.bashrc
  runuser -l gpadmin -c 'echo -e "gpstbmaster\ngpsegment01\ngpsegment02" > /home/gpadmin/hostfile'
  runuser -l gpadmin -c 'echo -e "gpsegment01\ngpsegment02" > /home/gpadmin/seglist'
  runuser -l gpadmin -c 'ssh-keyscan -f /home/gpadmin/hostfile >> /home/gpadmin/.ssh/known_hosts'
  runuser -l gpadmin -c 'for hostn in $(cat /home/gpadmin/hostfile); do sshpass -p "gpadmin" ssh-copy-id -i /home/gpadmin/.ssh/id_rsa.pub gpadmin@${hostn}; done'
  runuser -l gpadmin -c 'gpssh-exkeys -f /home/gpadmin/hostfile'
  echo "MESSAGE: Prepare gp_init_config file"
  cat << EOF >> /home/gpadmin/gp_init_config
ARRAY_NAME="Greenplum"
MACHINE_LIST_FILE=/home/gpadmin/hostfile
SEG_PREFIX=gpseg
declare -a DATA_DIRECTORY=(/data/primary /data/primary)
declare -a MIRROR_DATA_DIRECTORY=(/data/mirror /data/mirror)
MASTER_DIRECTORY=/data/master
MASTER_PORT=5432
PORT_BASE=6000
MIRROR_PORT_BASE=7000
REPLICATION_PORT_BASE=8000
MIRROR_REPLICATION_PORT_BASE=9000
TRUSTED_SHELL=ssh
CHECK_POINT_SEGMENTS=8
ENCODING=UNICODE
EOF
  echo "MASTER_HOSTNAME=$(hostname)" >> /home/gpadmin/gp_init_config
  chown gpadmin:gpadmin /home/gpadmin/gp_init_config
  chmod 0644 /home/gpadmin/gp_init_config
  SCRIPT

  $stbmasscript = <<-'SCRIPT'
  echo "MESSAGE: Preparing GP standby master host"
  mkdir -p /data/master /data/primary /data/mirror 
  chown -R gpadmin:gpadmin /data
  echo "source /usr/local/greenplum-db/greenplum_path.sh" >> /home/gpadmin/.bashrc
  echo "export MASTER_DATA_DIRECTORY=/data/master/gpseg-1" >> /home/gpadmin/.bashrc
  runuser -l gpadmin -c 'echo -e "gpsegment01\ngpsegment02" > /home/gpadmin/hostfile'
  runuser -l gpadmin -c 'ssh-keyscan -f /home/gpadmin/hostfile >> /home/gpadmin/.ssh/known_hosts'
  runuser -l gpadmin -c 'for hostn in $(cat /home/gpadmin/hostfile); do sshpass -p "gpadmin" ssh-copy-id -i /home/gpadmin/.ssh/id_rsa.pub gpadmin@${hostn}; done'
  runuser -l gpadmin -c 'gpssh-exkeys -f /home/gpadmin/hostfile'
  SCRIPT

  $segscript = <<-'SCRIPT'
  echo "MESSAGE: GP database preparations"
  mkdir -p /data/primary /data/mirror
  chown -R gpadmin:gpadmin /data
  echo "source /usr/local/greenplum-db/greenplum_path.sh" >> /home/gpadmin/.bashrc
  SCRIPT

  config.vm.define "gpsegment01" do |gpseg01|
    # Box settings
    gpseg01.vm.box = "generic/centos7"
    # Network settings
    gpseg01.vm.network "public_network", bridge: "wlp5s0", ip: "192.168.1.31"
    # Set hostname
    gpseg01.vm.hostname = "gpsegment01"
    # Provider settings
    gpseg01.vm.provider "virtualbox" do |vb|
      vb.cpus = 2
      vb.memory = 3096
    end
    # Provisioning script
    gpseg01.vm.provision "shell", inline: $cmnscript
    gpseg01.vm.provision "shell", inline: $segscript
    gpseg01.vm.provision :shell do |shell|
      shell.privileged = true
      shell.inline = 'echo Rebooting to apply changes'
      shell.reboot = true
    end
  end

  config.vm.define "gpsegment02" do |gpseg02|
    # Box settings
    gpseg02.vm.box = "generic/centos7"
    # Network settings
    gpseg02.vm.network "public_network", bridge: "wlp5s0", ip: "192.168.1.32"
    # Set hostname
    gpseg02.vm.hostname = "gpsegment02"
    # Provider settings
    gpseg02.vm.provider "virtualbox" do |vb|
      vb.cpus = 2
      vb.memory = 3096
    end
    # Provisioning script
    gpseg02.vm.provision "shell", inline: $cmnscript
    gpseg02.vm.provision "shell", inline: $segscript
    gpseg02.vm.provision :shell do |shell|
      shell.privileged = true
      shell.inline = 'echo Rebooting to apply changes'
      shell.reboot = true
    end
  end

  config.vm.define "gpstbmaster" do |gpstbmas|
    # Box settings
    gpstbmas.vm.box = "generic/centos7"
    # Network settings
    gpstbmas.vm.network "public_network", bridge: "wlp5s0", ip: "192.168.1.40"
    # Set hostname
    gpstbmas.vm.hostname = "gpstbmaster"
    # Provider settings
    gpstbmas.vm.provider "virtualbox" do |vb|
      vb.cpus = 2
      vb.memory = 2048
    end
    # Provisioning script
    gpstbmas.vm.provision "shell", inline: $cmnscript
    gpstbmas.vm.provision "shell", inline: $stbmasscript
    gpstbmas.vm.provision :shell do |shell|
      shell.privileged = true
      shell.inline = 'echo Rebooting to apply changes'
      shell.reboot = true
    end
  end

  config.vm.define "gpmaster" do |gpmas|
    # Box settings
    gpmas.vm.box = "generic/centos7"
    # Network settings
    gpmas.vm.network "public_network", bridge: "wlp5s0", ip: "192.168.1.30"
    # Set hostname
    gpmas.vm.hostname = "gpmaster"
    # Provider settings
    gpmas.vm.provider "virtualbox" do |vb|
      vb.cpus = 2
      vb.memory = 2048
    end
    # Provisioning script
    gpmas.vm.provision "shell", inline: $cmnscript
    gpmas.vm.provision "shell", inline: $masscript
    gpmas.vm.provision :shell do |shell|
      shell.privileged = true
      shell.inline = 'echo Rebooting to apply changes'
      shell.reboot = true
    end
  end

end