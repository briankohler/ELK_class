# -*- mode: ruby -*-
# vi: set ft=ruby :


Vagrant.configure("2") do |config|
  config.vm.box = "motus/dockerdev"
  config.vm.hostname = "escluster.vm.local"
  config.vm.post_up_message =  "Elasticsearch is running on 3 nodes.  We have a load balancer forwarding ES ports (9200 and 9300) to each of the 3 ES nodes.  The ES nodes have ElasticsearchHead and KOPF plugins installed.  These are available at http://localhost:9200/_plugin/kopf and http://localhost:9200/_plugin/head.  Kibana is running on http://localhost:5601.  Additionally, logstash is installed on the ES nodes themselves, with a configuration to input from Redis on localhost and output to elasticsearch.  As an example application, Apache is installed on the vagrant VM hosting the standard Welcome page.  Logstash is installed and configured to input from the apache access log file, parse the log lines using the built-in Apache grok patterns, and send to the Redis load-balanced endpoint. Visit localhost:8080 to generate some logs. All data and configuration files are stored in the GIT repo and can be modified via your preferred text editor on the host system." 
  config.vm.network "forwarded_port", guest: 5601, host: 5601
  config.vm.network "forwarded_port", guest: 9200, host: 9200
  config.vm.network "forwarded_port", guest: 9300, host: 9300
  config.vm.network "forwarded_port", guest: 6379, host: 6379
  config.vm.network "forwarded_port", guest: 80, host:8080
  config.vm.network "forwarded_port", guest: 1936, host: 1936

  config.vm.network "private_network", ip: "192.168.33.10"

  config.vm.provider "virtualbox" do |vb|
     vb.memory = "4096"
     vb.cpus = "4"
  end

  config.vm.provision "shell", inline: <<-SHELL
     echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
     echo "exclude=.at" | sudo tee -a /etc/yum/pluginconf.d/fastestmirror.conf
     sudo yum makecache fast
     sudo yum install -y yum-utils device-mapper-persistent-data lvm2 epel-release git httpd wget
     sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
     sudo yum-config-manager --enable docker-ce-edge
     sudo yum makecache fast
     sudo yum install -y docker-ce python2-pip
     sudo pip install pip --upgrade
     sudo pip install docker-compose
     sudo /sbin/service docker start
     cd /
     sudo wget https://download.elastic.co/logstash/logstash/packages/debian/logstash-2.4.0_all.deb
     sudo ln -s /vagrant/elasticsearch /home/vagrant/elasticsearch
     cd /home/vagrant/elasticsearch
     echo
     echo
     echo "Launching 3-node ES cluster, Kibana, and haproxy"
     echo
     echo
     sudo docker-compose up -d
     echo
     echo
     sudo docker-compose exec -T es01 ./bin/plugin install org.codelibs/elasticsearch-reindexing/2.3.0
     sudo docker-compose exec -T es02 ./bin/plugin install org.codelibs/elasticsearch-reindexing/2.3.0
     sudo docker-compose exec -T es03 ./bin/plugin install org.codelibs/elasticsearch-reindexing/2.3.0
     sudo docker-compose restart es01
     sleep 20
     sudo docker-compose restart es02
     sleep 20
     sudo docker-compose restart es03
     sleep 20
     echo
     echo "Installing ESHead and KOPF plugins on each ES node"
     echo
     echo
     sudo docker-compose exec -T es01 ./bin/plugin install lmenezes/elasticsearch-kopf/v2.1.1
     sudo docker-compose exec -T es01 ./bin/plugin install mobz/elasticsearch-head
     sudo docker-compose exec -T es02 ./bin/plugin install lmenezes/elasticsearch-kopf/v2.1.1
     sudo docker-compose exec -T es02 ./bin/plugin install mobz/elasticsearch-head
     sudo docker-compose exec -T es03 ./bin/plugin install lmenezes/elasticsearch-kopf/v2.1.1
     sudo docker-compose exec -T es03 ./bin/plugin install mobz/elasticsearch-head
     echo
     echo
     echo "installing logstash on es01 to run as an indexer, installing redis on es01.  Logstash indexer will pull from redis on localhost and push into ES"
     echo
     echo
     sudo docker-compose exec -T es01 apt-get update
     sudo docker-compose exec -T es01 apt-get install -y init-system-helpers logrotate cron libpopt0 redis-server
     sudo docker-compose exec -T es01 dpkg -i /logstash-2.4.0_all.deb
     sudo docker-compose exec -T es01 sed -i 's/127.0.0.1/0.0.0.0/g' /etc/redis/redis.conf
     sudo docker-compose exec -T es01 redis-server /etc/redis/redis.conf --daemonize yes
     echo
     echo
     echo "installing logstash on es02 to run as an indexer, installing redis on es02.  Logstash indexer will pull from redis on localhost and push into ES"
     echo
     echo
     sudo docker-compose exec -T es02 apt-get update
     sudo docker-compose exec -T es02 apt-get install -y init-system-helpers logrotate cron libpopt0 redis-server
     sudo docker-compose exec -T es02 dpkg -i /logstash-2.4.0_all.deb
     sudo docker-compose exec -T es02 sed -i 's/127.0.0.1/0.0.0.0/g' /etc/redis/redis.conf
     sudo docker-compose exec -T es02 redis-server /etc/redis/redis.conf --daemonize yes
     echo
     echo
     echo "installing logstash on es03 to run as an indexer, installing redis on es03.  Logstash indexer will pull from redis on localhost and push into ES"
     echo
     echo
     sudo docker-compose exec -T es03 apt-get update
     sudo docker-compose exec -T es03 apt-get install -y init-system-helpers logrotate cron libpopt0 redis-server
     sudo docker-compose exec -T es03 dpkg -i /logstash-2.4.0_all.deb
     sudo docker-compose exec -T es03 sed -i 's/127.0.0.1/0.0.0.0/g' /etc/redis/redis.conf
     sudo docker-compose exec -T es03 redis-server /etc/redis/redis.conf --daemonize yes
     echo
     echo
     echo "starting logstash indexers on each of the ES nodes"
     echo
     echo
     sudo docker-compose exec -T -d es01 /opt/logstash/bin/logstash -f /etc/logstash/conf.d/logstash.conf --debug &
     sudo docker-compose exec -T -d es02 /opt/logstash/bin/logstash -f /etc/logstash/conf.d/logstash.conf --debug &
     sudo docker-compose exec -T -d es03 /opt/logstash/bin/logstash -f /etc/logstash/conf.d/logstash.conf --debug &
     echo
     echo
     echo "installing logstash on the vagrant machine to act as a node sending logs to elasticsearch via redis output"
     echo
     echo
     sudo ln -s /vagrant/elasticsearch/logstash.repo /etc/yum.repos.d/logstash.repo
     sudo yum install -y java logstash
     sudo ln -s /vagrant/elasticsearch/logstash.conf /etc/logstash/conf.d/logstash.conf
     sudo sed -i 's/LS_USER=logstash/LS_USER=root/g' /etc/init.d/logstash
     sudo sed -i 's/LS_GROUP=logstash/LS_GROUP=root/g' /etc/init.d/logstash
     sudo systemctl daemon-reload
     echo
     echo
     echo "Grabbing most up-to-date grok patterns from git"
     echo 
     echo
     sudo git clone https://github.com/logstash-plugins/logstash-patterns-core.git /tmp/patterns
     sudo mv /tmp/patterns/patterns /etc/logstash/
     echo
     echo
     echo "starting apache on vagrant machine"
     echo
     echo
     sudo /sbin/service httpd start
     curl -s localhost:80
     curl -s localhost:80
     echo
     echo
     echo "starting logstash on vagrant machine.  Visit localhost:8080 to generate logs"
     echo
     echo
     sudo /etc/init.d/logstash start
  SHELL
end

