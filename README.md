Once the VMs are up using the Vagrant file, configure/setup each VM
1) db01 - MySql VM
   Vagrant ssh db01
   yum update -y | updates OS
   yum install epel-release | provides additional packages that are not included in the default repositories
   yum install git mariadb-server -y | installs mariadb
   systemctl start mariadb | starts service
   systemctl enable mariadb | enables service
   systemctl status mariadb | to check the status
   mysql_secure_installation | setup the Mysql

   To set Database name and users
   mysql -u root -p<password>  | to get into sql
      create database accounts
      grant all privileges on accounts.* TO '<username>'@'%' identified by '<password>;
      flush privileges;
      exit

   Download source code & initialize dtabase.
   git clone <https address> or copy to VM
   mysql -u root -p<password> accounts < <path of back up of sql>
   mysql -u root -p<password accounts
      show tables;
      exit;
   systemctl restart mariadb

2) mc01 - memcached VM
   vagrant ssh mc01
   sudo su - | switch to root
   dnf install epel-release -y
   dnf install memchached -y
   systemctl start memcached
   systemctl enable memcached
   sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached | to listen from all ip address
   systemctl restart memcached

3) RabbitMQ | dummy one
   Vagrant ssh rmq01 and update the OS
   yum install epel-release -y
   install wget -y
   cd /tmp/
   dnf -y install centos-release-rabbitmq-38
   dnf --enablerepo=centos-rabbitmq-38 -y install rabbitmq-server
   systemctl enable --now rabbitmq-server
   sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
   sudo rabbitmqctl add_user test test
   sudo rabbitmqctl set_user_tags test administrator
   sudo systemctl restart rabbitmq-server

4) Tomcat | Application
   vagrant ssh app01 and update the OS, switch to root
   yum install epel-release -y
   dnf -y install java-11-openjdk java-11-openjdk-devel
   dnf install git maven wget -y
   cd /tmp/
   wget https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.75/bin/apache-tomcat-9.0.75.tar.gz
   tar xzvf apache-tomcat-9.0.75.tar.gz
   useradd --home-dir /usr/local/tomcat --shell /sbin/nologin tomcat
   cp -r /tmp/apache-tomcat-9.0.75/* /usr/local/tomcat/
   chown -R tomcat.tomcat /usr/local/tomcat
   vi /etc/systemd/system/tomcat.service
   # Update the file with below content
      [Unit]
      Description=Tomcat
      After=network.target
      [Service]
      User=tomcat
      WorkingDirectory=/usr/local/tomcat
      Environment=JRE_HOME=/usr/lib/jvm/jre
      Environment=JAVA_HOME=/usr/lib/jvm/jre
      Environment=CATALINA_HOME=/usr/local/tomcat
      Environment=CATALINE_BASE=/usr/local/tomcat
      ExecStart=/usr/local/tomcat/bin/catalina.sh run
      ExecStop=/usr/local/tomcat/bin/shutdown.sh
      SyslogIdentifier=tomcat-%i
      [Install]
      WantedBy=multi-user.target
   systemctl daemon-reload
   systemctl start tomcat
   systemctl enable tomcat
   # download and build code and deploy
   git clone -b main https://github.com/hkhcoder/vprofile-project.git | source code
   cd vprofile-project
   vim src/main/resources/application.properties
   Update file with backend server details
   # Build code
   mvn install | should be inside the directory where config files are present pom.xml
   #Deploy artifact
   systemctl stop tomcat
   rm -rf /usr/local/tomcat/webapps/ROOT*
   cp target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
   systemctl start tomcat
   chown tomcat.tomcat /usr/local/tomcat/webapps -R
   systemctl restart tomcat

Nginx Setup
   vagrant ssh web01, update OS and switch to user
   apt install nginx -y
   vi /etc/nginx/sites-available/vproapp | Create Nginx conf file
   Update with below content
      upstream vproapp {
      server app01:8080;
      }
      server {
      listen 80;
      location / {
      proxy_pass http://vproapp;
      }
      }
   rm -rf /etc/nginx/sites-enabled/default | Remove default nginx conf
   ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp | Create link to activate website
   systemctl restart nginx
