# Zabbix API
You should develop a script (on Python 2.x) which registers given host in Zabbix.

Testing Infrastructure:
+ Vagrantfile to spin up 2 VMs (virtualbox):
+ zabbix server, provisioned by Vagrant provisioner
+ Linux VM with zabbix agent, script for registration on zabbix server, all provisioned by Vagrant provisioner

Registering Script requirements:
+ Written on Python 2.x
+ Starts at VM startup or on provision phase
+ Host registered in Zabbix server should have Name = Hostname (not IP)
+ Host registered in Zabbix server should belong to "CloudHosts" group
+ Host registered in Zabbix server should be linked with Custom template
+ This script should create group “CloudHosts” id it doesn’t exist

