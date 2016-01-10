# -*- mode: ruby -*-
# vi: set ft=ruby :

## BASE OS
##############################################

# non-updated CentOS 7.1 OS
# Uncomment BOX_NAME below after running:
# vagrant box add --name CentOS-7-x64 https://github.com/CommanderK5/packer-centos-template/releases/download/0.7.1/vagrant-centos-7.1.box
BOX_NAME = "CentOS-7-x64"

# updated/upgraded OS (faster, no-internet)
#BOX_NAME = "dcos-centos"
# zk docker and nginx docker loaded
#BOX_NAME = "dcos-boot"

## CLUSTER CONFIG
##############################################
IP_DETECT_SCRIPT="ip-detect"
#DCOS_CLUSTER_CONFIG="1_master-config.json"
#DCOS_CLUSTER_CONFIG="3_master-config.json"
#DCOS_CLUSTER_CONFIG="5_master-config.json"
DCOS_CLUSTER_CONFIG="3_master-config.yaml"
#DCOS_CLUSTER_CONFIG="5_master-config.yaml"
DCOS_SLAVE_CONFIG="3_mesos-slave-common"
#DCOS_SLAVE_CONFIG="5_mesos-slave-common"
DCOS_LB_CONFIG="3_master-haproxy.cfg"
#DCOS_LB_CONFIG="5_master-haproxy.cfg"

DCOS_GENERATE_CONFIG_PATH= ENV['DCOS_GENERATE_CONFIG_PATH'] || "file:///vagrant/dcos_generate_config.sh"

## Commands for configuring systems for DCOS req, master and worker install
##############################################

DCOS_OS_PROVISION = <<-SHELL
    #setenforce 0
    #sed -i s/SELINUX=enforcing/SELINUX=permissive/g /etc/selinux/config
    #echo ">>> Set SELinux to permissive"

    useradd -m -s /bin/bash -g wheel -N core
    sudo -i -u core sh -c 'mkdir -p ~/.ssh && chmod 700 ~/.ssh && rm -rf ~/.ssh/* && cd ~/.ssh && cp /vagrant/etc/authorized_keys . && chmod 600 authorized_keys'
    echo ">>> Setup default Mesosphere user: core"

    #sysctl -w net.ipv6.conf.all.disable_ipv6=1
    #sysctl -w net.ipv6.conf.default.disable_ipv6=1
    #echo ">>> Disabled IPV6"

    #yum makecache fast
    #yum install --assumeyes --tolerant --quiet deltarpm epel-release
    #yum upgrade --assumeyes --tolerant --quiet
    #echo ">>> Upgraded Base OS"

    #yum install --assumeyes --tolerant --quiet dkms qemu-guest-agent
    #echo ">>> Installed dkms & qemu-guest-agent"
    #/etc/init.d/vboxadd setup
    #echo ">>> Rebuilt Virtualbox Additions"

    #yum install --assumeyes --tolerant --quiet tar xz unzip curl git bind-utils nfs-client
    #echo ">>> Installed tar, xz, unzip, curl, git, bind-utils & nfs-client"
    yum install --assumeyes --tolerant --quiet vim strace perf htop bash-completion
    echo ">>> Installed vim, strace, perf, htop & bash-completion"
    yum install --assumeyes --tolerant --quiet python-pip
    echo ">>> Installed python-pip"

    #curl -sSL https://get.docker.com | sh
    #echo ">>> Installed Docker"

    #usermod -aG docker vagrant
    #echo ">>> Added vagrant to the docker group"

    cp /vagrant/etc/overlay.conf /etc/modules-load.d/
    cp /vagrant/etc/docker.service /etc/systemd/system/
    systemctl daemon-reload
    systemctl restart systemd-modules-load
    systemctl restart docker
    echo ">>> Setup docker to use the OverlayFS storage engine"

    #systemctl enable docker
    #echo ">>> Enabled Docker to start on reboot"

    #service docker start
    #echo ">>> Starting docker and running (docker ps)"
    #docker ps

    ln -sf /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem /etc/ssl/certs/ca-certificates.crt
    ln -sf /etc/pki/ca-trust/extracted/openssl/ca-bundle.trust.crt /etc/ssl/certs/ca-bundle.trust.crt
    echo ">>> Added /etc/ssl/certs/ca-certificates.crt symlink to /etc/ssl/certs/ca-certificates.crt to make cURL happy"

    yum clean all
    echo ">>> Cleaned up yum"
SHELL

DCOS_HOSTSFILE_PROVISION = <<-SHELL
    cp /vagrant/etc/hosts.file /etc/hosts
    echo ">>> Setup /etc/hosts"
SHELL

DCOS_BOOT_PROVISION = <<-SHELL
    mkdir -p /var/tmp/dcos && rm -rf /var/tmp/dcos/*
    docker run -d -p 3181:2181 -p 2888:2888 -p 3888:3888 jplock/zookeeper
    echo ">>> Creating docker service (jplock/zookeeper) for exhibitor bootstrap and quorum."

    docker run -d -p 8888:80 -v /var/tmp/dcos:/usr/share/nginx/html nginx
    echo ">>> Creating docker service (nginx) for ease of distributing bootstrap artifacts to cluster."

    docker ps

    yum makecache fast
    yum install --assumeyes --tolerant --quiet haproxy
    yum clean all
    cp /vagrant/etc/#{DCOS_LB_CONFIG} /etc/haproxy/haproxy.cfg
    systemctl enable haproxy
    systemctl start haproxy
    echo ">>> Setup DCOS HAProxy Load Balancer"

    mkdir -p ~/dcos/genconf && rm -rf ~/dcos/genconf/* && cd ~/dcos
    cp /vagrant/etc/#{IP_DETECT_SCRIPT} ~/dcos/genconf/ip-detect
    cp /vagrant/etc/#{DCOS_CLUSTER_CONFIG} ~/dcos/genconf/config.yaml
    echo ">>> Copied (ip-detect, config.yaml) for building bootstrap image for system."

    curl -O -sS #{DCOS_GENERATE_CONFIG_PATH}
    echo ">>> Downloading (dcos_generate_config.sh) for building bootstrap image for system."

    bash ~/dcos/dcos_generate_config.sh
    echo ">>> Built bootstrap artifacts under (#{ENV['PWD']}/genconf/serve)."

    cp -rp ~/dcos/genconf/serve/* /var/tmp/dcos/
    echo ">>> Copied bootstrap artifacts to nginx directory (/var/tmp/dcos)."

    cp /vagrant/build/jq-linux64 /usr/bin/jq
    echo ">>> Installed JQ"
    pip install --quiet --upgrade pip
    pip install --quiet --upgrade virtualenv
    echo ">>> Installed and updated pip and virtualenv"
    sudo -i -u vagrant sh -c 'mkdir -p ~/dcos && rm -rf ~/dcos/* && cd dcos && virtualenv --quiet . && source bin/activate && pip install --quiet --upgrade pip dcoscli httpie'
    echo ">>> Installed and updated dcoscli and HTTPie"
    sudo -i -u vagrant sh -c 'mkdir -p ~/.dcos && cp /vagrant/etc/dcos.toml ~/.dcos && cp /vagrant/etc/bash_profile ~/.bash_profile'
    echo ">>> Setup dcoscli config"
    sudo -i -u vagrant sh -c 'cd dcos && source bin/env-setup && dcos package update --validate && dcos package search --json | jq '.[].packages[].name' | xargs -L 1 dcos package install --cli --yes'
    echo ">>> Setup DCOS CLI configuration, updated package manifest and installed available subcommands for DCOS Services"

    mkdir -p /var/tmp/dcos/java && rm -rf /var/tmp/dcos/java/*
    cp -rp /vagrant/build/gs-spring-boot-0.1.0.jar /var/tmp/dcos/java/
    cp -rp /vagrant/build/jre-*-linux-x64.* /var/tmp/dcos/java/
    echo ">>> Copied java artifacts to nginx dir (/var/tmp/dcos/java)."
SHELL

DCOS_INSTALL_PROVISION = <<-SHELL
    groupadd nogroup
    mkdir -p ~/dcos && rm -rf ~/dcos && cd ~/dcos
    curl -O -sS http://boot.dcos:8888/dcos_install.sh && \
    echo ">>> Mesos install script downloaded"
SHELL

DCOS_MASTER_PROVISION = <<-SHELL
    bash dcos_install.sh master
    echo ">>> Mesos Master installed"
SHELL

DCOS_WORKER_ZK_PROVISION = <<-SHELL
    mkdir -p /var/lib/dcos && rm -rf /var/lib/dcos/*
    cp /vagrant/etc/#{DCOS_SLAVE_CONFIG} /var/lib/dcos/mesos-slave-common
    echo ">>> Overriding Zookeper URL to ensure proper HA when masters fail"
SHELL

DCOS_WORKER_PROVISION = <<-SHELL
    bash dcos_install.sh slave
    echo ">>> Mesos Agent installed"
SHELL

DCOS_WORKER_PUBLIC_PROVISION = <<-SHELL
    bash dcos_install.sh slave_public
    echo ">>> Mesos Public Agent installed"
SHELL

#### Instance config definitions
##############################################

Vagrant.configure(2) do |config|
    {
        :boot => {
                  :box => BOX_NAME,
                  :ip => '192.168.65.50',
                  :memory => 1024,
                  :provision => [DCOS_BOOT_PROVISION],
                  },
        :mlb => {
                 :ip => '192.168.65.60',
                 :memory => 1024,
                 :provision => [DCOS_INSTALL_PROVISION,
                                DCOS_WORKER_ZK_PROVISION,
                                DCOS_WORKER_PUBLIC_PROVISION],
                 },
        :m1 => {
                :ip => '192.168.65.90',
                :memory => 2048,
                :provision => [DCOS_INSTALL_PROVISION,
                               DCOS_MASTER_PROVISION],
                },
        :m2 => {
                :ip => '192.168.65.91',
                :memory => 2048,
                :provision => [DCOS_INSTALL_PROVISION,
                               DCOS_MASTER_PROVISION]
               },
        :m3 => {
                :ip  => '192.168.65.92',
                :memory => 2048,
                :provision => [DCOS_INSTALL_PROVISION,
                               DCOS_MASTER_PROVISION],
                },
        :w1 => {
                :ip => '192.168.65.100',
                :memory => 3584,
                :provision => [DCOS_INSTALL_PROVISION,
                               DCOS_WORKER_ZK_PROVISION,
                               DCOS_WORKER_PROVISION],
                },
        :w2 => {
                :ip => '192.168.65.101',
                :memory => 3584,
                :provision => [DCOS_INSTALL_PROVISION,
                               DCOS_WORKER_ZK_PROVISION,
                               DCOS_WORKER_PROVISION],
                },
        :w3 => {
                :ip => '192.168.65.102',
                :memory => 3584,
                :provision => [DCOS_INSTALL_PROVISION,
                               DCOS_WORKER_ZK_PROVISION,
                               DCOS_WORKER_PROVISION],
                },
        :w4 => {
                :ip => '192.168.65.103',
                :memory => 3584,
                :provision => [DCOS_INSTALL_PROVISION,
                               DCOS_WORKER_ZK_PROVISION,
                               DCOS_WORKER_PROVISION],
                },
        :w5 => {
                :ip => '192.168.65.104',
                :memory => 3584,
                :provision => [DCOS_INSTALL_PROVISION,
                               DCOS_WORKER_ZK_PROVISION,
                               DCOS_WORKER_PROVISION],
                },
        :w6 => {
                :ip => '192.168.65.105',
                :memory => 3584,
                :provision => [DCOS_INSTALL_PROVISION,
                               DCOS_WORKER_ZK_PROVISION,
                               DCOS_WORKER_PROVISION],
                }
    }.each do |name,cfg|
        config.vm.define name do |vm_cfg|
            vm_cfg.vm.box = cfg[:box] || BOX_NAME
            vm_cfg.vm.hostname = "#{name}.dcos"
            #vm_cfg.vm.network ":private_network", ip: cfg[:ip]
            vm_cfg.vm.network :private_network,
                    ip: cfg[:ip],
                    libvirt__network_name: "dcos"
            #vm_cfg.vm.synced_folder './', '/vagrant', type: 'rsync'

            #vm_cfg.vm.provider "virtualbox" do |virtualbox|
            #  virtualbox.name = vm_cfg.vm.hostname
            #  virtualbox.memory = cfg[:memory]
            #  virtualbox.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
            #end

            vm_cfg.vm.provider "libvirt" do |libvirt|
                libvirt.cpus = 2
                libvirt.memory = cfg[:memory]
                libvirt.storage_pool_name = "tank"
                libvirt.graphics_type = "spice"
                libvirt.video_type = "qxl"
                libvirt.video_vram = 65536
                libvirt.input :type => "tablet", :bus => "usb"
            end

            if cfg[:forwards]
                cfg[:forwards].each do |from,to|
                    vm_config.vm.forward_port from, to
                end
            end

            vm_cfg.vm.provision "shell", name: "DCOS Base OS provisioning", inline: DCOS_OS_PROVISION
            vm_cfg.vm.provision "shell", name: "DCOS /etc/hosts provisioning", inline: DCOS_HOSTSFILE_PROVISION

            if cfg[:provision]
                cfg[:provision].each do |provisioning_step|
                    vm_cfg.vm.provision "shell", name: "DCOS node-specific provisioning", inline: provisioning_step
                end
            end
        end
    end
end

################# END ######################

__END__
