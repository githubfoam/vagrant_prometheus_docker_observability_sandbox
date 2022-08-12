# vagrant-prometheus-mysql-docker-standalone

~~~~


in that order

>vagrant destroy -f vg-pro-gr-lk-05
>vagrant up vg-pro-gr-lk-05

>vagrant destroy -f vg-pro-gr-lk-04
>vagrant up vg-pro-gr-lk-04

Prometheus GUI
http://192.168.53.11:9090/
Grafana (admin/admin)
http://192.168.53.11:3000/
http://192.168.53.11:9090/targets

Prometheus metrics
http://192.168.53.11:9090/metrics
cadvisor
http://192.168.53.11:8080/metrics
local docker metrics
http://192.168.53.11:9323/metrics
remote docker metrics
http://192.168.53.12:9323/metrics


remote
vagrant@vg-pro-gr-lk-05:~$ docker container ls
CONTAINER ID   IMAGE                    COMMAND                  CREATED          STATUS         PORTS                                                    NAMES
af1ac903e07d   google/cadvisor:latest   "/usr/bin/cadvisor -…"   6 seconds ago    Up 4 seconds   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp                cadvisor-remote
76a9653d78b0   mysql:8                  "docker-entrypoint.s…"   16 minutes ago   Up 7 minutes   33060/tcp, 0.0.0.0:49153->3306/tcp, :::49153->3306/tcp   mysql80
c65e9e04fa59   httpd                    "httpd-foreground"       42 minutes ago   Up 6 minutes   0.0.0.0:80->80/tcp, :::80->80/tcp                        apache-server-remote


Prometheus GUI - Graph
container_memory_usage_bytes{name=~"apache-server"} #OK
container_memory_usage_bytes{name=~"apache-server-remote"} # NOT OK
container_memory_usage_bytes{name=~"mysql80"} # NOT OK

engine_daemon_network_actions_seconds_count
engine_daemon_engine_cpus_cpus{instance="192.168.53.12:9323", job="docker-remote"}
~~~~
