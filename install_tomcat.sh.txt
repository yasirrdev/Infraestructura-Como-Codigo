#!/bin/bash

sudo sed -i 's/^\$nrconf{restart} = "i";/\$nrconf{restart} = "a";/' /etc/needrestart/needrestart.conf
sudo sed -i 's/^\$nrconf{kernelhints} = -1;/\$nrconf{kernelhints} = -1;/' /etc/needrestart/needrestart.conf
sudo sed -i 's/^\$nrconf{checks} = 1;/\$nrconf{checks} = 0;/' /etc/needrestart/needrestart.conf

useradd -m -d /opt/tomcat -U -s /bin/false tomcat

apt update && apt upgrade -y
apt install openjdk-21-jdk wget -y

wget https://dlcdn.apache.org/tomcat/tomcat-11/v11.0.2/bin/apache-tomcat-11.0.2.tar.gz -P /tmp
mkdir -p /opt/tomcat
tar xzvf /tmp/apache-tomcat-11.0.2.tar.gz -C /opt/tomcat --strip-components=1

chown -R tomcat:tomcat /opt/tomcat
chmod -R u+x /opt/tomcat/bin

sed -i '/<\/tomcat-users>/i \
<role rolename="manager-gui" />\n<user username="manager" password="manager_password" roles="manager-gui" />\n<role rolename="admin-gui" />\n<user username="admin" password="admin_password" roles="manager-gui,admin-gui" />' /opt/tomcat/conf/tomcat-users.xml

sed -i '/<Valve /,/\/>/ s|<Valve|<!--<Valve|; /<Valve /,/\/>/ s|/>|/>-->|' /opt/tomcat/webapps/manager/META-INF/context.xml
sed -i '/<Valve /,/\/>/ s|<Valve|<!--<Valve|; /<Valve /,/\/>/ s|/>|/>-->|' /opt/tomcat/webapps/host-manager/META-INF/context.xml

cat << EOF > /etc/systemd/system/tomcat.service
[Unit]
Description=Apache Tomcat 11 Web Application Container
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat
Environment="JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64"
Environment="CATALINA_PID=/opt/tomcat/temp/tomcat.pid"
Environment="CATALINA_HOME=/opt/tomcat"
Environment="CATALINA_BASE=/opt/tomcat"
ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start tomcat
systemctl enable tomcat
