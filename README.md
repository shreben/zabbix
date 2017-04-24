# Zabbix Web Monitoring

Testing Infrastructure:
+ Vagrantfile to spin up 2 VMs (virtualbox):
zabbix server, provisioned by Vagrant provisioner
Zabbix agents on both VMs, provisioned by Vagrant provisioner
+ Install Tomcat 7 on 2nd VM, deploy any “hello world” application

Tasks:
Configure WEB check:
+ Scenario to test Tomcat availability as well as Application heath
+ Configure Triggers to alert once WEB resources become unavailable
