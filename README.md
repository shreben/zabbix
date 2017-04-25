# Zabbix Java Monitoring

### Task
You should install and configure Zabbix server and agents.

Testing Infrastructure:
+ Vagrantfile to spin up 2 VMs (virtualbox):
+ zabbix server, provisioned by Vagrant provisioner
+ Zabbix agents on both VMs, provisioned by Vagrant provisioner
+ Install Tomcat 7 on 2nd VM

Tasks:
+ Configure Zabbix to examine Java parameters via Java Gateway (http://jmxmonitor.sourceforge.net/jmx.html)
+ Configure triggers to alert once these parameters changed.

### Report
1. Extract from Vagrantfile:
```ruby
# Define tomcat VM
  config.vm.define "tomcat" do |tomcat|
    tomcat.vm.hostname="tomcat.mtn.lab"
    tomcat.vm.network "private_network", ip: "192.0.0.4"
    tomcat.vm.provider "virtualbox" do |vb|
      vb.name = "tomcat"
      vb.gui = false
      vb.linked_clone = true
    end
    tomcat.vm.provision "shell", inline: <<-SHELL
      yum install -y zabbix-agent
      /bin/cp /vagrant/passive_zabbix_agentd.conf /etc/zabbix/zabbix_agentd.conf
      yum install -y java-1.8.0-openjdk 2>&1 > /dev/null
      wget -qN https://archive.apache.org/dist/tomcat/tomcat-8/v8.5.11/bin/apache-tomcat-8.5.11.tar.gz -P /tmp
      tar zxf /tmp/apache-tomcat-8.5.11.tar.gz -C /opt/
      ln -s /opt/apache-tomcat-8.5.11 /opt/tomcat
      /bin/cp /vagrant/setenv.sh /opt/tomcat/bin
      /bin/cp /vagrant/server.xml /opt/tomcat/conf
      /bin/cp /vagrant/hudson.* /opt/tomcat/webapps
      chmod +x /opt/tomcat/bin/setenv.sh
      groupadd tomcat
      useradd -s /bin/nologin -d/opt/tomcat -g tomcat tomcat
      grep server /etc/hosts
      [ $? -ne 0 ] && echo "192.0.0.2 zabbix-server.mtn.lab" >> /etc/hosts
      systemctl network restart
      systemctl enable zabbix-agent
      systemctl start zabbix-agent
      sudo -s "/opt/tomcat/bin/startup.sh" -u tomcat
    SHELL
  end
```
2. Test Application

![](https://github.com/shreben/zabbix/blob/java/screenshots/java_hudson_main_page.png)

3. Installed and configured zabbix-java-gateway
* installation
```
yum install -y zabbix-java-gateway
```
* zabbix_java_gateway.conf
```
LISTEN_IP="0.0.0.0"
LISTEN_PORT=10052
PID_FILE="/var/run/zabbix/zabbix_java.pid"
START_POLLERS=5
TIMEOUT=3
```
* zabbix_server.conf
```
JavaGateway=127.0.0.1
JavaGatewayPort=10052
StartJavaPollers=10
```
4. Enabled jmx on tomcat
* tomcat_home/conf/server.xml
```
<Listener className="org.apache.catalina.mbeans.JmxRemoteLifecycleListener"
            rmiRegistryPortPlatform="10001" rmiServerPortPlatform="10002" rmiBindAddress="192.0.0.4"/>
```
* tomcat_home/bin/setenv.sh
```
CATALINA_OPTS="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false"
```
5. Added and configured node tomcat.mtn.lab in zabbix:

![](https://github.com/shreben/zabbix/blob/java/screenshots/java_tomcat_config.png)

6. Created custom template MTN Lab Java Checks containing two applications:
* MTN Lab Java App

![](https://github.com/shreben/zabbix/blob/java/screenshots/java_java_app.png)

* MTN Lab Hudson App

![](https://github.com/shreben/zabbix/blob/java/screenshots/java_hudson_app.png)

5. Monitoring -> Latest Data

![](https://github.com/shreben/zabbix/blob/java/screenshots/java_latest_data.png)


