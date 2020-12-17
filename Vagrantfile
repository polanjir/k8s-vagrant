# -*- mode: ruby -*-
# vi: set ft=ruby :

box = "ubuntu/focal64"
box_version = "20201210.0.0"

k8s_version = "1.20.0-00"
docker_version = "19.03"

servers = [
    {
        :name => "k8s-master",
        :type => "master",
        :ip_addr => "192.168.205.10",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "k8s-worker-1",
        :type => "worker",        
        :ip_addr => "192.168.205.11",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "k8s-worker-2",
        :type => "worker",
        :ip_addr => "192.168.205.12",
        :mem => "2048",
        :cpu => "2"
    }
]

$configureBox = <<-SCRIPT
    export DEBIAN_FRONTEND=noninteractive

    # Install Docker
    apt-get update
    apt-get install -y apt-transport-https ca-certificates curl software-properties-common
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
    add-apt-repository "deb https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") $(lsb_release -cs) stable"
    docker_version=$(apt-cache madison docker-ce | grep "#{docker_version}" | head -1 | awk '{print $3}')
    apt-get update && apt-get install -y docker-ce=$docker_version docker-ce-cli=$docker_version containerd.io
    apt-mark hold docker-ce docker-ce-cli

    # Run docker commands as vagrant user
    usermod -aG docker vagrant

    # Change default cgroup driver to systemd
    cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
    systemctl restart docker

    # Install kubeadm, kubelet, kubectl
    apt-get install -y apt-transport-https curl
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    add-apt-repository "deb https://apt.kubernetes.io/ kubernetes-xenial main"

    apt-get update
    apt-get install -y kubelet=#{k8s_version} kubeadm=#{k8s_version} kubectl=#{k8s_version}
    apt-mark hold kubelet kubeadm kubectl

    # kubelet requires swap off
    swapoff -a
    sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

    # Set node-ip
    echo 'KUBELET_EXTRA_ARGS=--node-ip=$IP_ADDR' > /etc/default/kubelet
    systemctl restart kubelet
SCRIPT

$configureMaster = <<-SCRIPT
    # Install k8s master
    kubeadm init \
        --apiserver-advertise-address=$IP_ADDR \
        --apiserver-cert-extra-sans=$IP_ADDR \
        --node-name $(hostname -s) \
        --pod-network-cidr=10.244.0.0/16

    # Copy kube config
    sudo --user=vagrant mkdir -p /home/vagrant/.kube
    cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
    chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

    # Install Pod network addon
    export KUBECONFIG=/etc/kubernetes/admin.conf
    # FLANNEL
    kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
    # CALICO
    # kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

    mkdir -p /vagrant/tmp
    kubeadm token create --print-join-command > /vagrant/tmp/kubeadm-join.sh

    cp /etc/kubernetes/admin.conf /vagrant/tmp
SCRIPT

$configureWorker = <<-SCRIPT
    # Join the node to the existing K8S cluster
    $SHELL /vagrant/tmp/kubeadm-join.sh
SCRIPT

Vagrant.configure("2") do |config|
    config.vagrant.plugins = ["vagrant-vbguest"]

    servers.each do |opts|
        config.vm.define opts[:name] do |config|

            config.vm.box = box
            config.vm.box_version = box_version
            config.vm.hostname = opts[:name]
            config.vm.network :private_network, ip: opts[:ip_addr]
            config.ssh.insert_key = false
            config.ssh.forward_agent = true

            config.vbguest.auto_update = false # Disable VirtualBox Guest Additions

            config.vm.provider "virtualbox" do |virtualbox|
                virtualbox.memory = opts[:mem]
                virtualbox.cpus = opts[:cpu]
                virtualbox.customize ["modifyvm", :id, "--groups", "/K8S-cluster"]
            end

            config.vm.provision "shell", inline: $configureBox, env: {"IP_ADDR" => opts[:ip_addr]}

            if opts[:type] == "master"
                config.vm.provision "shell", inline: $configureMaster, env: {"IP_ADDR" => opts[:ip_addr]}
            else
                config.vm.provision "shell", inline: $configureWorker
            end
        end
    end
end 