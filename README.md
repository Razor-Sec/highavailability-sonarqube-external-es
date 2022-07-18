# INFO VM

|Name|IP|
|:---|:-----:|
|node sq1 es1| 10.8.60.220|
|node sq2 es2| 10.8.60.221|
|node sq3 es3| 10.8.60.196|

> nodes elasticsearch should be 3 nodes for minimum failover , if nodes less than 3 failover will doesn't work. As Elasticsearch is consensus based you need at least 3 master eligible nodes. Losing any of your nodes will therefore make your cluster not fully operable.

# Setup High Avaibility PostgreSQL with keepalived and replication stream

Follow this tutorial : https://github.com/Razor-Sec/highavaibility-postgresql-keepalived

# Setup hosts

```bash
vim /etc/hosts
...
10.8.60.220 es-master sonarqube-master
10.8.60.221 es-slave1 sonarqube-slave1
10.8.60.196 es-slave2 sonarqube-slave2
...
``` 

# Setup port on all nodes

```bash
firewall-cmd --add-port={9001,9300,9000}/{tcp,udp} --permanent
firewall-cmd --reload
firewall-cmd --list-ports
```

# Install elasticsearch on all nodes

```bash
# create directory for data elasticsearch
mkdir -p /opt/sq_es/{data,logs}
chown -R elasticsearch /opt/sq_es

# install openjdk-devel
dnf install java-11-openjdk-devel
java -version

# Installing Elasticsearch
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
vim /etc/yum.repos.d/elasticsearch.repo
...
[elasticsearch-7.x]
name=Elasticsearch repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
...
dnf install elasticsearch
```

# Configure elasticsearch on node es1

```bash
vim /etc/elasticsearch/elasticsearch.yml
...
#es-master

http.cors.allow-origin: '*'
cluster.name: elastic-sq-bri
node.name: es-master
node.master: true
node.data: true
node.voting_only: false
network.host: 0.0.0.0
network.publish_host: 10.8.60.220
transport.tcp.port: '9300'
discovery.seed_hosts: ["10.8.60.220", "10.8.60.221", "10.8.60.196"]
#discovery.zen.ping.unicast.hosts: ["10.8.60.220", "10.8.60.221", "10.8.60.196"]
cluster.initial_master_nodes: ["es-master", "es-slave1", "es-slave2"]
discovery.zen.minimum_master_nodes: 3
http.port: '9001'
http.cors.enabled: 'true'
path.logs: /opt/sq_es/logs
path.data: /opt/sq_es/data/es7
...

systemctl start elasticsearch.service
systemctl status elasticsearch.service
curl 0.0.0.0:9001
curl 10.8.60.220:9001
```

# Configure elasticsearch on node es2

```bash
vim /etc/elasticsearch/elasticsearch.yml
...
#es-slave1

http.cors.allow-origin: '*'
cluster.name: elastic-sq-bri
node.name: es-slave1
node.master: true
node.data: true
node.voting_only: false
network.host: 0.0.0.0
network.publish_host: 10.8.60.221
transport.tcp.port: '9300'
discovery.seed_hosts: ["10.8.60.220", "10.8.60.221", "10.8.60.196"]
#discovery.zen.ping.unicast.hosts: ["10.8.60.220", "10.8.60.221", "10.8.60.196"]
cluster.initial_master_nodes: ["es-master", "es-slave1", "es-slave2"]
discovery.zen.minimum_master_nodes: 3
http.port: '9001'
http.cors.enabled: 'true'
path.logs: /opt/sq_es/logs
path.data: /opt/sq_es/data/es7
...

systemctl start elasticsearch.service
systemctl status elasticsearch.service
curl 0.0.0.0:9001
curl 10.8.60.221:9001
```

# Configure elasticsearch on node es3

```bash
vim /etc/elasticsearch/elasticsearch.yml
...
#es-slave2

http.cors.allow-origin: '*'
cluster.name: elastic-sq-bri
node.name: es-slave2
node.master: true
node.data: true
node.voting_only: false
network.host: 0.0.0.0
network.publish_host: 10.8.60.196
transport.tcp.port: '9300'
discovery.seed_hosts: ["10.8.60.220", "10.8.60.221", "10.8.60.196"]
#discovery.zen.ping.unicast.hosts: ["10.8.60.220", "10.8.60.221", "10.8.60.196"]
cluster.initial_master_nodes: ["es-master", "es-slave1", "es-slave2"]
discovery.zen.minimum_master_nodes: 3
http.port: '9001'
http.cors.enabled: 'true'
path.logs: /opt/sq_es/logs
path.data: /opt/sq_es/data/es7
...

systemctl start elasticsearch.service
systemctl status elasticsearch.service
curl 0.0.0.0:9001
curl 10.8.60.196:9001

```

# Checking node elasticsearch
```bash
# checking nodes
curl es-master:9001/_cat/nodes?v
# checking indices
curl es-master:9001/_cat/indices?v
```

# Test create index elasticsearch
```bash
# add example docs
curl -X POST "localhost:9001/test-docs/_doc?pretty" -H 'Content-Type: application/json' -d'
{
  "@timestamp": "2099-05-06T16:21:15.000Z",
  "event": {
    "original": "192.0.2.42 - - [06/May/2099:16:21:15 +0000] \"GET /images/bg.jpg HTTP/1.0\" 200 24736"
  }
}
'

# check
curl es-master:9001/_cat/indices?v
curl -X GET "localhost:9001/test-docs/_search?pretty"
```


# Setup database for sonarqube using vip


# Install sonarqube on all nodes
```bash
# setup sysctl
sudo tee -a /etc/sysctl.conf<<EOF
vm.max_map_count=262144
fs.file-max=65536
EOF
sysctl --system

# Add user for sonar services
useradd sonar 
passwd sonar 

# Download sonarqube community
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.5.0.56709.zip
unzip sonarqube-9.5.0.56709.zip
mkdir -p /opt/sonarqube-ce
mv sonarqube-9.5.0.56709/* /opt/sonarqube-ce
vim /opt/sonarqube-ce/conf/sonar.properties
...
sonar.web.host=0.0.0.0
sonar.web.port=9000
sonar.jdbc.username=sonar
sonar.jdbc.password=redhat123
sonar.jdbc.url=jdbc:postgresql://10.8.60.218/sonarqube?currentSchema=sonarqubeschema 
# es
sonar.search.port=9001
sonar.search.host=localhost
...

# disable elasticserarch local sonarqube
cat > /opt/sonarqube-ce/elasticsearch/bin/elasticsearch << EOF
#!/bin/bash
# it's a inflate sleep
cat
EOF

# run sonarqube with service
vim /etc/systemd/system/sonarqube.service
...
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking

ExecStart=/opt/sonarqube-ce/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube-ce/bin/linux-x86-64/sonar.sh stop
LimitNOFILE=65536
LimitNPROC=4096

User=sonar
Group=sonar
Restart=always

[Install]
WantedBy=multi-user.target

...

# add permission to sonar
chown -R sonar:sonar /opt/sonarqube-ce/

# Start service Sonarqube
systemctl status sonarqube.service
systemctl enable --now sonarqube.service

```

# Checking webservice sonarqube

curl -u admin:redhat123 localhost:9000/api/system/health

# DONE :D