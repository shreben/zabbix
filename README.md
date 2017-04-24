# Zabbix Java Monitoring
You should install and configure Zabbix server and agents.

Testing Infrastructure:
+ Vagrantfile to spin up 2 VMs (virtualbox):
+ zabbix server, provisioned by Vagrant provisioner
+ Zabbix agents on both VMs, provisioned by Vagrant provisioner
+ Install Tomcat 7 on 2nd VM

Tasks:
+ Configure Zabbix to examine Java parameters via Java Gateway (http://jmxmonitor.sourceforge.net/jmx.html)
+ Configure triggers to alert once these parameters changed.

