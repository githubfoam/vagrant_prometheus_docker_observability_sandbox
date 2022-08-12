# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.

# Minikube
KUBERNETES_VERSION = ENV['KUBERNETES_VERSION'] || "1.16.3" # OK
# X Exiting due to GUEST_MISSING_CONNTRACK: Sorry, Kubernetes 1.24.3 requires conntrack to be installed in root's path
# KUBERNETES_VERSION = ENV['KUBERNETES_VERSION'] || "1.24.3" # NOT OK conntrack missing

$prometheus_mysql_exporter_script = <<-SCRIPT

echo "current user is $(whoami)"
echo "current directory is $(pwd)"

# containers are in the same Docker network called “db_network”
docker network create db_network
docker network ls

docker run -d \
--name mysql80 \
--publish 3306 \
--network db_network \
--restart unless-stopped \
--env MYSQL_ROOT_PASSWORD=mypassword \
--volume mysql80-datadir:/var/lib/mysql \
mysql:8 \
--default-authentication-plugin=mysql_native_password


docker run -d \
--name mysql57 \
--publish 3306 \
--network db_network \
--restart unless-stopped \
--env MYSQL_ROOT_PASSWORD=mypassword \
--volume mysql57-datadir:/var/lib/mysql \
mysql:5.7

#Verify if MySQL servers are running
docker ps | grep mysql


#Exposing Docker Metrics to Prometheus
#https://docs.docker.com/config/daemon/prometheus/
cat | sudo tee /etc/docker/daemon.json << EOF
{
  "experimental": true,
  "metrics-addr": "192.168.53.9:9323"
}
EOF
cat /etc/docker/daemon.json

systemctl restart docker
systemctl status docker

#Deploying MySQL Exporter

# the mysqld exporter requires a MySQL user to be used for monitoring purposes
#create the monitoring user
cat | sudo tee /tmp/mysql80.sql << EOF
CREATE USER 'exporter'@'%' IDENTIFIED BY 'exporterpassword' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';
EOF
cat /tmp/mysql80.sql
stat /tmp/mysql80.sql

# the mysqld exporter requires a MySQL user to be used for monitoring purposes
#create the monitoring user
cat | sudo tee /tmp/mysql57.sql << EOF
CREATE USER 'exporter'@'%' IDENTIFIED BY 'exporterpassword' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';
EOF
cat  /tmp/mysql57.sql
stat /tmp/mysql57.sql

docker exec -i mysql80 mysql -u root -pmypassword </tmp/mysql80.sql
docker exec -i mysql57 mysql -u root -pmypassword </tmp/mysql57.sql


#Deploying MySQL Exporter

docker run -d \
--name mysql80-exporter \
--publish 9104 \
--network db_network \
--restart always \
--env DATA_SOURCE_NAME="exporter:exporterpassword@(mysql80:3306)/" \
prom/mysqld-exporter:latest \
--collect.info_schema.processlist \
--collect.info_schema.innodb_metrics \
--collect.info_schema.tablestats \
--collect.info_schema.tables \
--collect.info_schema.userstats \
--collect.engine_innodb_status


docker run -d \
--name mysql57-exporter \
--publish 9104 \
--network db_network \
--restart always \
-e DATA_SOURCE_NAME="exporter:exporterpassword@(mysql57:3306)/" \
prom/mysqld-exporter:latest \
--collect.info_schema.processlist \
--collect.info_schema.innodb_metrics \
--collect.info_schema.tablestats \
--collect.info_schema.tables \
--collect.info_schema.userstats \
--collect.engine_innodb_status

#Verify if MySQL servers and MySQL Exporters are running
docker ps | grep mysql


cat | sudo tee /tmp/prometheus.yml << EOF
global:
  scrape_interval:     5s
  scrape_timeout:      3s
  evaluation_interval: 5s

# Our alerting rule files
rule_files:
  - "alert.rules"

# Scrape endpoints
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'mysql'
    static_configs:
      - targets: ['mysql57-exporter:9104','mysql80-exporter:9104']

  - job_name: 'docker'
    static_configs:
      - targets: ['192.168.53.9:9323']

  - job_name: 'docker-remote'
    static_configs:
        - targets: ['192.168.53.10:9323']

EOF

cat /tmp/prometheus.yml #verify


docker run -d \
--name prometheus-server \
--publish 9090:9090 \
--network db_network \
--restart unless-stopped \
--mount type=volume,src=prometheus-data,target=/prometheus \
--mount type=bind,src=/tmp/prometheus.yml,target=/etc/prometheus/prometheus.yml \
prom/prometheus

#Verify if MySQL servers,MySQL Exporters and prometheus  are running
docker container ls

#the Prometheus web UI.
#http://192.168.53.9:9090/

# mysql exporter metrics
# http://192.168.53.9:49155/metrics
# http://192.168.53.9:49156/metrics

#docker metrics
# http://192.168.53.9:9323/metrics


# handling manually the remote docker engine
# steps to let prometheus detect remote docker engine
# on vg-pro-gr-lk-03 (192.168.53.10)
# docker container ls
# docker container stop prometheus-server
# docker container start prometheus-server


#Configure the Prometheus data source.
#The URL is the private IP of Prometheus
# http://http://192.168.53.9:9090
docker run -d \
--name grafana-server \
--network db_network \
--restart unless-stopped \
-p 3000:3000 grafana/grafana


#Verify if MySQL servers,MySQL Exporters,Prometheus and Grafana  are running
docker container ls

SCRIPT




$ubuntu_docker_script = <<-SCRIPT

echo "current user is $(whoami)"
echo "current directory is $(pwd)"

# vg-pro-gr-lk01: Package 'docker.io' is not installed, so not removed
# vg-pro-gr-lk01: E: Unable to locate package docker
# vg-pro-gr-lk01: E: Unable to locate package docker-engine
# The SSH command responded with a non-zero exit status. Vagrant
# assumes that this means the command failed. The output for this command
# should be in the log above. Please read the output to determine what
# went wrong.
# Uninstall old versions
# apt-get remove docker docker-engine docker.io containerd runc -y

# Set up the repository
apt-get update -y
apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release -y

# Add Docker’s official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# set up the stable repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null


# Install Docker Engine
apt-get update -y
apt-get install \
    docker-ce \
    docker-ce-cli \
    containerd.io -y


docker --version

# Verify that Docker Engine is installed correctly
docker run hello-world

# Post-installation steps for Linux
# Manage Docker as a non-root user

# Create the docker group
groupadd docker
# Add your user to the docker group
# usermod -aG docker $USER # by default run by root
usermod -aG docker vagrant

#docker compose
apt-get install docker-compose -y

SCRIPT

$Prometheus_Docker_Monitoring_script = <<-SCRIPT

# containers are in the same Docker network called “db_network”
docker network create observability_network
docker network ls

echo "============== Exposing Docker Metrics to Prometheus =============="
#https://docs.docker.com/config/daemon/prometheus/
cat | sudo tee /etc/docker/daemon.json << EOF
{
  "experimental": true,
  "metrics-addr": "192.168.53.11:9323"
}
EOF

cat /etc/docker/daemon.json

systemctl restart docker
systemctl status docker

echo "============== Deploy cadvisor =============="
docker run -d \
--restart always \
--name cadvisor \
--network observability_network \
-p 8080:8080 \
-v "/:/rootfs:ro" \
-v "/var/run:/var/run:rw" \
-v "/sys:/sys:ro" \
-v "/var/lib/docker/:/var/lib/docker:ro" \
gcr.io/cadvisor/cadvisor:latest

docker container ls


echo "============== Deploy prometheus =============="

cd /tmp
cat | sudo tee /tmp/prometheus.yml << EOF
global:
  scrape_interval:     5s
  scrape_timeout:      3s
  evaluation_interval: 5s

# Our alerting rule files
rule_files:
  - "alert.rules"

# Scrape endpoints
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: cadvisor
    scrape_interval: 5s
    static_configs:
      - targets: ['192.168.53.11:8080']
        labels:
          group: 'cadvisor'
  - job_name: 'docker-local'
    static_configs:
      - targets: ['192.168.53.11:9323']
  - job_name: 'docker-remote'
    static_configs:
      - targets: ['192.168.53.12:9323']

EOF

cat /tmp/prometheus.yml #verify

docker run -d \
--name prometheus-server \
--publish 9090:9090 \
--network observability_network \
--restart unless-stopped \
--mount type=volume,src=prometheus-data,target=/prometheus \
--mount type=bind,src=/tmp/prometheus.yml,target=/etc/prometheus/prometheus.yml \
prom/prometheus

docker container ls

echo "============== Deploy grafana =============="

docker run -d \
--name grafana-server \
--network observability_network \
--restart unless-stopped \
-p 3000:3000 \
grafana/grafana


echo "============== Deploy apache test server =============="

docker run -d \
--name apache-server \
--network observability_network \
-p 80:80 \
httpd

docker container ls

SCRIPT

# script on vg-pro-gr-lk-05 (192.168.53.12)
$prometheus_remote_script = <<-SCRIPT

#Exposing Docker Metrics to remote Prometheus server
#https://docs.docker.com/config/daemon/prometheus/
cat | sudo tee /etc/docker/daemon.json << EOF
{
  "experimental": true,
  "metrics-addr": "192.168.53.12:9323"
}
EOF
cat /etc/docker/daemon.json

systemctl restart docker
systemctl status docker

docker network create observability_network_remote
docker network ls


echo "============== Deploy cadvisor =============="
docker run -d \
--restart always \
--name cadvisor-remote \
--network observability_network_remote \
-p 8080:8080 \
-v "/:/rootfs:ro" \
-v "/var/run:/var/run:rw" \
-v "/sys:/sys:ro" \
-v "/var/lib/docker/:/var/lib/docker:ro" \
gcr.io/cadvisor/cadvisor:latest



echo "============== Deploy apache test server =============="

docker run -d \
--name apache-server-remote \
--network observability_network_remote \
--restart unless-stopped \
-p 80:80 \
httpd


echo "============== Deploy mysql server without mysql-exporter=============="

docker run -d \
--name mysql80 \
--publish 3306 \
--network observability_network_remote \
--restart unless-stopped \
--env MYSQL_ROOT_PASSWORD=mypassword \
--volume mysql80-datadir:/var/lib/mysql \
mysql:8 \
--default-authentication-plugin=mysql_native_password

docker container ls
SCRIPT

Vagrant.configure("2") do |config|

  config.vm.provider "virtualbox" do |vb|
    vb.gui = false
    vb.memory = "1024"
    vb.cpus = 2

  end



    config.vm.define "vg-pro-gr-lk-04" do |kalicluster|
      # hhttps://app.vagrantup.com/ubuntu/boxes/jammy64
      kalicluster.vm.box = "ubuntu/jammy64" #22.04
      # https://app.vagrantup.com/ubuntu/boxes/focal64
      # kalicluster.vm.box = "ubuntu/focal64" #Official Ubuntu 20.04 LTS (Focal Fossa) builds
      # https://app.vagrantup.com/ubuntu/boxes/xenial64
      # kalicluster.vm.box = "ubuntu/bionic64" #18.04
      # https://app.vagrantup.com/ubuntu/boxes/xenial64
      # kalicluster.vm.box = "ubuntu/xenial64" #16.04
      kalicluster.vm.hostname = "vg-pro-gr-lk-04"
      #bridged network,DHCP disabled, manual IP assignment
      # kalicluster.vm.network "public_network", ip: "10.10.8.67"
      #bridged network,DHCP enabled,auto IP assignment
      # kalicluster.vm.network "public_network"
      kalicluster.vm.network "private_network", ip: "192.168.53.11"
      # kalicluster.vm.network "forwarded_port", guest: 80, host: 81
      #Disabling the default /vagrant share can be done as follows:
      # kalicluster.vm.synced_folder ".", "/vagrant", disabled: true
      kalicluster.vm.provider "virtualbox" do |vb|
          vb.name = "vbox-pro-gr-lk-04"
          vb.cpus = 2
          vb.memory = 2048
          vb.gui = false
      end
      kalicluster.vm.provision "shell", inline: $ubuntu_docker_script
      # kalicluster.vm.provision "shell", inline: $prometheus_mysql_exporter_script
      kalicluster.vm.provision "shell", inline: $Prometheus_Docker_Monitoring_script
    end

    config.vm.define "vg-pro-gr-lk-05" do |kalicluster|
      # hhttps://app.vagrantup.com/ubuntu/boxes/jammy64
      kalicluster.vm.box = "ubuntu/jammy64" #22.04
      # https://app.vagrantup.com/ubuntu/boxes/focal64
      # kalicluster.vm.box = "ubuntu/focal64" #Official Ubuntu 20.04 LTS (Focal Fossa) builds
      # https://app.vagrantup.com/ubuntu/boxes/xenial64
      # kalicluster.vm.box = "ubuntu/bionic64" #18.04
      # https://app.vagrantup.com/ubuntu/boxes/xenial64
      # kalicluster.vm.box = "ubuntu/xenial64" #16.04
      kalicluster.vm.hostname = "vg-pro-gr-lk-05"
      #bridged network,DHCP disabled, manual IP assignment
      # kalicluster.vm.network "public_network", ip: "10.10.8.67"
      #bridged network,DHCP enabled,auto IP assignment
      # kalicluster.vm.network "public_network"
      kalicluster.vm.network "private_network", ip: "192.168.53.12"
      # kalicluster.vm.network "forwarded_port", guest: 80, host: 81
      #Disabling the default /vagrant share can be done as follows:
      # kalicluster.vm.synced_folder ".", "/vagrant", disabled: true
      kalicluster.vm.provider "virtualbox" do |vb|
          vb.name = "vbox-pro-gr-lk-05"
          vb.cpus = 2
          vb.memory = 1024
          vb.gui = false
      end
      kalicluster.vm.provision "shell", inline: $ubuntu_docker_script
      kalicluster.vm.provision "shell", inline: $prometheus_remote_script

    end

end
