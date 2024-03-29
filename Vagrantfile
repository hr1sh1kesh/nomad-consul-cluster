# -*- mode: ruby -*-
# vi: set ft=ruby :

$base = <<BASE
# Update apt and get dependencies
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y unzip curl vim jq netcat \
    apt-transport-https \
    ca-certificates \
    software-properties-common
BASE

$docker_install = <<DOCKER_INSTALL
echo "Installing Docker..."
if [[ -f /etc/apt/sources.list.d/docker.list ]]; then
    echo "Docker repository already installed; Skipping"
else
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    sudo apt-get update
fi
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y docker-ce

# Restart docker to make sure we get the latest version of the daemon if there is an upgrade
sudo service docker restart

# Make sure we can actually use docker as the vagrant user
sudo usermod -aG docker vagrant
DOCKER_INSTALL

$java_install = <<JAVA_INSTALL 
sudo apt-get update -y
sudo apt-get install -y default-jdk
JAVA_INSTALL

$dnsmasq = <<DNSMASQ
YUM=$(which yum 2>/dev/null)
APT_GET=$(which apt-get 2>/dev/null)
if [[ ! -z ${YUM} ]]; then
  logger "Installing dnsmasq"
  sudo yum install -q -y dnsmasq
elif [[ ! -z ${APT_GET} ]]; then
  logger "Installing dnsmasq"
  sudo apt-get -qq -y update
  sudo apt-get install -qq -y dnsmasq-base dnsmasq
else
  logger "Dnsmasq not installed due to OS detection failure"
  exit 1;
fi

logger "Configuring dnsmasq to forward .consul requests to consul port 8600"
sudo sh -c 'echo "server=/consul/127.0.0.1#8600" >> /etc/dnsmasq.d/consul'
sudo systemctl enable dnsmasq
sudo systemctl restart dnsmasq
DNSMASQ

$nomad_download = <<NOMAD_DOWNLOAD
NOMAD_VERSION="1.0.1"

echo "Fetching Nomad..."
if [[ ! -f /vagrant/nomad ]] ; then
    cd /vagrant/
    sudo curl  https://releases.hashicorp.com/nomad/${NOMAD_VERSION}/nomad_${NOMAD_VERSION}_linux_amd64.zip -o nomad.zip
    sudo unzip nomad.zip
fi
NOMAD_DOWNLOAD

$consul_download = <<CONSUL_DOWNLOAD 
CONSUL_VERSION="1.9.0"

 echo "Fetching Consul..."
if [[ ! -f /vagrant/consul ]] ; then
    cd /vagrant/
    sudo curl https://releases.hashicorp.com/consul/${CONSUL_VERSION}/consul_${CONSUL_VERSION}_linux_amd64.zip -o consul.zip
    sudo unzip consul.zip
fi
CONSUL_DOWNLOAD

$vault_download = <<VAULT_DOWNLOAD
VAULT_VERSION="1.2.2"

if [[ ! -f /vagrant/vault ]] ; then
    cd /vagrant/
    sudo curl https://releases.hashicorp.com/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_linux_amd64.zip -o vault.zip
    sudo unzip vault.zip
fi
VAULT_DOWNLOAD

$nomad_install = <<NOMAD_INSTALL
echo "Installing Nomad..."
sudo install /vagrant/nomad /usr/bin/nomad
(
cat <<-EOF
[Unit]
Description=nomad agent
Requires=network-online.target
After=network-online.target

[Service]
Restart=on-failure
ExecStart=/usr/bin/nomad agent -config /etc/nomad.d/
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
EOF
) | sudo tee /etc/systemd/system/nomad.service

cd /tmp
sudo mkdir -p /etc/nomad.d
sudo chmod a+w /etc/nomad.d

# Set hostname's IP to made advertisement Just Work
sudo sed -i -e "s/.*nomad.*/$(ip route get 1 | awk '{print $NF;exit}') nomad/" /etc/hosts

nomad -autocomplete-install
sudo rm -f /vagrant/nomad.zip
NOMAD_INSTALL

$consul_install = <<CONSUL_INSTALL
echo "Installing Consul..."
sudo install /vagrant/consul /usr/bin/consul
(
cat <<-EOF
	[Unit]
	Description=consul agent
	Requires=network-online.target
	After=network-online.target

	[Service]
	Restart=on-failure
	ExecStart=/usr/bin/consul agent -config-dir=/etc/consul.d
	ExecReload=/bin/kill -HUP $MAINPID

	[Install]
	WantedBy=multi-user.target
EOF
) | sudo tee /etc/systemd/system/consul.service

sudo mkdir -p /etc/consul.d
sudo chmod a+w /etc/consul.d
sudo rm -f /vagrant/consul.zip
CONSUL_INSTALL

$vault_install = <<VAULT_INST
echo "Installing Vault..."

sudo install /vagrant/vault /usr/local/bin/vault
(
cat <<-EOF
  [Unit]
  Description=vault agent
  Requires=consul-online.target
  After=consul-online.target
  [Service]
  Restart=on-failure
  User=vault
  Group=vault
  PermissionsStartOnly=true
  ExecStartPre=/sbin/setcap 'cap_ipc_lock=+ep' /usr/local/bin/vault
  ExecStart=/usr/local/bin/vault server -config /etc/vault.d
  ExecReload=/bin/kill -HUP $MAINPID
  KillSignal=SIGINT
  [Install]
  WantedBy=multi-user.target
EOF
) | sudo tee /etc/systemd/system/vault.service

sudo chmod 0755 /usr/local/bin/vault
sudo chown root:root /usr/local/bin/vault
sudo /usr/sbin/groupadd --force --system vault
if ! getent passwd vault >/dev/null ; then
        sudo /usr/sbin/adduser \
          --system \
          --no-create-home \
          --shell /bin/false \
          vault  >/dev/null
fi

sudo mkdir -pm 0755 /etc/vault.d/
sudo chmod a+w /etc/vault.d
sudo rm -f /vagrant/vault.zip
VAULT_INST



$vault_env = <<VAULT_ENV
sudo cat << EOF > /etc/profile.d/vault.sh
export VAULT_ADDR="http://active.vault.service.consul:8200"
export VAULT_SKIP_VERIFY=true
EOF
VAULT_ENV

$nomad_config = <<NOMAD_CONFIG
INSTANCE_IP="$(/sbin/ifconfig eth1 | grep 'inet' | awk '{print $2}' | head -n 1)"
#export ROOT_TOKEN=$(consul kv get service/vault/root-token)

sudo cat << EOF > /etc/nomad.d/config.hcl
data_dir = "/opt/nomad/data"
#bind_addr = "0.0.0.0"
advertise {
  http = "$INSTANCE_IP"
  rpc  = "$INSTANCE_IP"
  serf = "$INSTANCE_IP" 
}
log_level = "DEBUG"
server {
  enabled = true
  bootstrap_expect = 3
}
client {
  enabled = true
  options {
    "docker.cleanup.image"   = "0"
    "driver.raw_exec.enable" = "1"
  }
}
consul {
  address = "127.0.0.1:8500"
}
acl {
  enabled = true
  token_ttl = "30s"
  policy_ttl = "60s"
}
EOF
NOMAD_CONFIG

$vault_config = <<VAULT_CONFIG
sudo cat << EOF > /etc/vault.d/config.hcl
backend "consul" {
  address = "127.0.0.1:8500"
  path = "vault"
}
listener "tcp" {
  address = "0.0.0.0:8200"
  tls_disable = 1
}
ui = true
EOF
VAULT_CONFIG

$consul_config = <<CONSUL_CONFIG
sudo cat << EOF > /etc/consul.d/config.json
{
  "server": true,
  "bootstrap_expect": 3,
  "leave_on_terminate": true,
  "advertise_addr": "$(/sbin/ifconfig eth1 | grep 'inet' | awk '{print $2}' | head -n 1)",
  "retry_join": ["192.168.50.150","192.168.50.151","192.168.50.152"],
  "data_dir": "/opt/consul/data",
  "client_addr": "0.0.0.0",
  "log_level": "INFO",
  "ui": true
}
EOF
CONSUL_CONFIG

$consul_online_target = <<CONSUL_ONLINE_TARGET
sudo cat << EOF > /etc/systemd/system/consul-online.target
[Unit]
Description=Consul Online
RefuseManualStart=true
EOF
CONSUL_ONLINE_TARGET

$consul_online_service = <<CONSUL_ONLINE_SERVICE
sudo cat << EOF > /etc/systemd/system/consul-online.service
[Unit]
Description=Consul Online
Requires=consul.service
After=consul.service
[Service]
Type=oneshot
ExecStart=/usr/bin/consul-online.sh
User=consul
Group=consul
[Install]
WantedBy=consul-online.target multi-user.target
EOF
CONSUL_ONLINE_SERVICE

$consul_online_script = <<CONSUL_ONLINE_SCRIPT
sudo cat << EOF > /usr/bin/consul-online.sh
#!/usr/bin/env bash
set -e
set -o pipefail
CONSUL_ADDRESS=${1:-"127.0.0.1:8500"}

function waitForConsulToBeAvailable() {
  local consul_addr=$1
  local consul_leader_http_code
  consul_leader_http_code=$(curl --silent --output /dev/null --write-out "%{http_code}" "${consul_addr}/v1/operator/raft/configuration") || consul_leader_http_code=""
  while [ "x${consul_leader_http_code}" != "x200" ] ; do
    echo "Waiting for Consul to get a leader..."
    sleep 5
    consul_leader_http_code=$(curl --silent --output /dev/null --write-out "%{http_code}" "${consul_addr}/v1/operator/raft/configuration") || consul_leader_http_code=""
  done
}
waitForConsulToBeAvailable "${CONSUL_ADDRESS}"
EOF
CONSUL_ONLINE_SCRIPT

$consul_template_install = <<CONSUL_TEMPLATE_INSTALL 
CONSUL_TEMPLATE_VERSION=${VERSION:-"0.19.4"}
CONSUL_TEMPLATE_ZIP="consul-template_${CONSUL_TEMPLATE_VERSION}_linux_amd64.zip"
CONSUL_TEMPLATE_URL=${URL:-"https://releases.hashicorp.com/consul-template/${CONSUL_TEMPLATE_VERSION}/${CONSUL_TEMPLATE_ZIP}"}
#CONSUL_TEMPLATE_USER=${USER:-"consul-template"}
#CONSUL_TEMPLATE_GROUP=${GROUP:-"consul-template"}
CONFIG_DIR="/etc/consul-template.d"
DATA_DIR="/opt/consul-template/data"
DOWNLOAD_DIR="/tmp"
curl --silent --output ${DOWNLOAD_DIR}/${CONSUL_TEMPLATE_ZIP} ${CONSUL_TEMPLATE_URL}
sudo unzip -o ${DOWNLOAD_DIR}/${CONSUL_TEMPLATE_ZIP} -d /usr/local/bin/
sudo chmod 0755 /usr/local/bin/consul-template
sudo chown root:root /usr/local/bin/consul-template
sudo mkdir -pm 0755 ${CONFIG_DIR} ${DATA_DIR}
sudo chown -R root:root ${CONFIG_DIR} ${DATA_DIR}
CONSUL_TEMPLATE_INSTALL

$consul_template_service = <<CONSUL_TEMPLATE_SERVICE
# Setup Nomad-Vault config for Consul Template
echo "Installing Systemd service..."
sudo mkdir -p /etc/systemd/system/consul-template.d/templates
sudo bash -c "cat >/etc/systemd/system/consul-template.service" << 'EOF'
[Unit]
Description=consul-template agent
Requires=network-online.target
After=network-online.target consul.service
[Service]
EnvironmentFile=-/ramdisk/client_token
Restart=on-failure
ExecStart=/usr/local/bin/consul-template -template "/tmp/nomad-vault.hcl.tpl:/etc/nomad.d/nomad-vault.hcl:service nomad restart"
KillSignal=SIGINT
[Install]
WantedBy=multi-user.target
EOF
sudo chmod 044 /etc/systemd/system/consul-template.service

CONSUL_TEMPLATE_SERVICE

$consul_template_config = <<CONSUL_TEMPLATE_CONFIG 
sudo cat << 'EOF' > /tmp/nomad-vault.hcl.tpl
vault {
  enabled = true
  create_from_role = "nomad-cluster"
  address = "http://active.vault.service.consul:8200"
  token = "{{ key "service/vault/nomad-token" }}"
}
EOF
CONSUL_TEMPLATE_CONFIG

$mysql_install = <<MYSQL_INSTALL
sudo apt-get install -y mariadb-server
sudo mysqladmin -u root password R00tPassword

sudo mysql -u root -p'R00tPassword' << EOF
GRANT ALL PRIVILEGES ON *.* TO 'vaultadmin'@'%' IDENTIFIED BY 'vaultadminpassword' WITH GRANT OPTION;
CREATE DATABASE app;
FLUSH PRIVILEGES;
EOF
sudo sed -i 's/bind-address/#bind-address/g' /etc/mysql/mariadb.conf.d/50-server.cnf
sudo service mysql restart;

echo '{"service": {"name": "db", "tags": ["mysql"], "port":3306}}' | sudo tee /etc/consul.d/mysql.json
sudo service consul restart
MYSQL_INSTALL

Vagrant.configure(2) do |config|

  config.vm.define "node1" do |node1|
    node1.vm.network :private_network, ip: "192.168.50.150"
    node1.vm.box = "bento/ubuntu-18.04" # 18.04 LTS
    node1.vm.hostname = "node1"
    node1.vm.network "private_network", type: "dhcp"
    
    node1.vm.network :forwarded_port, guest: 3646, host: 3646, auto_correct: true
    node1.vm.network :forwarded_port, guest: 3200, host: 3200, auto_correct: true
    node1.vm.network :forwarded_port, guest: 3500, host: 3500, auto_correct: true

    node1.vm.network :forwarded_port, guest: 8500, host: 8500, auto_correct: true
    node1.vm.network :forwarded_port, guest: 4646, host: 4646, auto_correct: true
    node1.vm.network :forwarded_port, guest: 8080, host: 8080, auto_correct: true
    node1.vm.network :forwarded_port, guest: 9998, host: 9998, auto_correct: true
    node1.vm.network :forwarded_port, guest: 9999, host: 9999, auto_correct: true
    node1.vm.network :forwarded_port, guest: 80,   host: 9080, auto_correct: true
    node1.vm.network :forwarded_port, guest: 443,  host: 9443, auto_correct: true
    node1.vm.post_up_message = "
    Nomad has been provisioned and is available at the following web address:
    http://localhost:4646/ui/     <<----  Primary Nomad UI (node1)
    Nomad has Consul installed as well with web UI available at the following web address:
    http://localhost:8500/ui/     <<----  Primary Consul UI (node1)
    Primary Vault node has been provisioned and is available at the following web address:
    http://localhost:8200/ui/     <<----  Primary Vault UI (node3)

    Nomad node2 has been provisioned and is available at the following web address:
    http://localhost:5646/ui/     <<----  Nomad UI (node2)
    Nomad node3 has been provisioned and is available at the following web address:
    http://localhost:6646/ui/     <<----  Nomad UI (node3)"
  end

  config.vm.define "node2" do |node2|
    node2.vm.network :private_network, ip: "192.168.50.151"
    node2.vm.box = "bento/ubuntu-18.04" # 18.04 LTS
    node2.vm.hostname = "node2"
    node2.vm.network "private_network", type: "dhcp"
    node2.vm.network :forwarded_port, guest: 8500, host: 8501, auto_correct: true
    node2.vm.network :forwarded_port, guest: 8200, host: 8201, auto_correct: true
    node2.vm.network :forwarded_port, guest: 4646, host: 5646, auto_correct: true
  end

  config.vm.define "node3" do |node3|
    node3.vm.network :private_network, ip: "192.168.50.152"
    node3.vm.box = "bento/ubuntu-18.04" # 18.04 LTS
    node3.vm.hostname = "node3"
    node3.vm.network "private_network", type: "dhcp"
    node3.vm.network :forwarded_port, guest: 8200, host: 8200, auto_correct: true
    node3.vm.network :forwarded_port, guest: 8500, host: 8502, auto_correct: true
    node3.vm.network :forwarded_port, guest: 8200, host: 8200, auto_correct: true
    node3.vm.network :forwarded_port, guest: 4646, host: 6646, auto_correct: true
    ### Start Vault on Node3
    # config.vm.provision "mysql", type: "shell", inline: $mysql_install, privileged: false
    # config.vm.provision "enable vault", type: "shell", inline: "sudo systemctl enable vault.service"
    # config.vm.provision "start vault", type: "shell", inline: "sudo systemctl start vault"
    # config.vm.provision "shell", path: "vault_init_and_unseal.sh",
    #   privileged: true,
    #   env: {"PATH" => "/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/root/bin"}
    # config.vm.provision "shell", path: "vault_nomad_integration.sh",
    #   privileged: true,
    #   env: {"PATH" => "/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/root/bin"}
  end

  config.vm.synced_folder "../../", "/vagrant"

  config.vm.provision "base", type: "shell", inline: $base, privileged: false
  config.vm.provision "docker",type: "shell", inline: $docker_install, privileged: false
  config.vm.provision "java",type: "shell", inline: $java_install, privileged: false
  config.vm.provision "dnsmasq",type: "shell", inline: $dnsmasq, privileged: false

  config.vm.provision "download nomad",type: "shell", inline: $nomad_download, privileged: false
  config.vm.provision "download consul",type: "shell", inline: $consul_download, privileged: false
#  config.vm.provision "download vault",type: "shell", inline: $vault_download, privileged: false

  config.vm.provision "install nomad",type: "shell", inline: $nomad_install, privileged: false
  config.vm.provision "install consul",type: "shell", inline: $consul_install, privileged: false
#  config.vm.provision "install vault",type: "shell", inline: $vault_install, privileged: false

  config.vm.provision "configure nomad",type: "shell", inline: $nomad_config, privileged: false
  config.vm.provision "configure consul",type: "shell", inline: $consul_config, privileged: false
#  config.vm.provision "configure vault",type: "shell", inline: $vault_config, privileged: false
 
  config.vm.provision "consul-online-target", type: "shell", inline: $consul_online_target
  config.vm.provision "consul-online-service", type: "shell", inline: $consul_online_service
  config.vm.provision "consul-online-script",  type: "shell", inline: $consul_online_script

  config.vm.provision "enable consul", type: "shell", inline: "sudo systemctl enable consul.service"
  config.vm.provision "start consul", type: "shell", inline: "sudo systemctl start consul"
  config.vm.provision "sleep for Consul", type: "shell", inline: "sleep 5s"
  config.vm.provision "enable nomad", type: "shell", inline: "sudo systemctl enable nomad.service"
  config.vm.provision "start nomad", type: "shell", inline: "sudo systemctl start nomad"

  config.vm.provision "install consul-template", type: "shell", inline: $consul_template_install
  config.vm.provision "install consul-template service", type: "shell", inline: $consul_template_service
  config.vm.provision "Configure consul-template Nomad", type: "shell", inline: $consul_template_config
  config.vm.provision "enable consul-template", type: "shell", inline: "sudo systemctl enable consul-template.service"
  config.vm.provision "start consul-template", type: "shell", inline: "sudo systemctl start consul-template"

  config.vm.provision "restart docker", type: "shell", inline: "sudo systemctl restart docker"

  
  config.vm.provision "config vault env variables", type: "shell", inline: $vault_env


  # Increase memory for Virtualbox
  config.vm.provider "virtualbox" do |vb|
        vb.cpus = "2"
        vb.memory = "2048"
  end

  # Increase memory for VMware
  ["vmware_fusion", "vmware_workstation"].each do |p|
    config.vm.provider p do |v|
      v.vmx["memsize"] = "2048"
    end
  end
end