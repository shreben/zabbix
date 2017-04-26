# Zabbix API

### Task
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

### Report
1. Extract from Vagrantfile to provision test VM (node2.mtn.lab):
```ruby
# define node2 vm
  config.vm.define 'node2' do |node|
    node.vm.hostname = 'node2.mtn.lab'
    node.vm.network 'private_network', ip: '192.0.0.5'
    node.vm.provider 'virtualbox' do |v|
      v.name = 'node2'
      v.linked_clone = true
    end
    node.vm.provision 'shell', inline: <<-SHELL
      grep server /etc/hosts
      [ $? -ne 0 ] && echo "192.0.0.2 zabbix-server.mtn.lab" >> /etc/hosts
      yum install -y http://repo.zabbix.com/zabbix/3.2/rhel/7/x86_64/zabbix-release-3.2-1.el7.noarch.rpm
      yum install -y net-tools
      yum install -y zabbix-agent
      yum install -y python2-pip
      pip install requests
      /bin/cp /vagrant/active_zabbix_agentd.conf /etc/zabbix/zabbix_agentd.conf
      systemctl restart network
      systemctl enable zabbix-agent
      systemctl start zabbix-agent
      cd /vagrant
      ./host_auto_register
    SHELL
  end
```

2. [Host registration script](https://github.com/shreben/zabbix/blob/api/host_auto_register):

```python
#!/usr/bin/env python
# -*- coding: utf8 -*-
#
#  Written by Siarhei Hreben
#  DevOps Lab 2017
#

import requests
import socket
import os

# Define required variables
host_name = socket.gethostname()
host_ip = (os.popen('ifconfig enp0s8 | grep -e "inet\s" | awk \'{print $2}\'')).read()
host_group = 'CloudHosts'
host_template = 'MTN Lab CloudHosts'
server_url = 'http://zabbix-server.mtn.lab/api_jsonrpc.php'
content_type = 'application/json-rpc'
zabbix_user = 'Admin'
zabbix_pass = 'zabbix'
request_id = 0


# define logout function
def logout(url, r_id, token):
    logout_request = {
        "jsonrpc": "2.0",
        "method": "user.logout",
        "params": [],
        "id": r_id,
        "auth": token
    }
    request = requests.post(url, json=logout_request)
    if request.json()['result']:
        print 'Logged out.'
    else:
        print "Error while logging out: {}".format(request.json())

# initial login
request_id += 1
auth_request = {
    "jsonrpc": "2.0",
    "method": "user.login",
    "params": {
        "user": zabbix_user,
        "password": zabbix_pass,
    },
    "id": request_id
}

r = requests.post(server_url, json=auth_request)
if 'error' in r.json().keys():
    print "Error: {}".format(r.json()['error']['data'])
    exit()
else:
    auth_token = r.json()['result']
    print 'Logged in.'

# check template
request_id += 1
get_host_template = {
    "jsonrpc": "2.0",
    "method": "template.get",
    "params": {
        "output": "extend",
        "filter": {
            "host": [
                host_template
            ]
        }
    },
    "auth": auth_token,
    "id": request_id
}

r = requests.post(server_url, json=get_host_template)
if 'error' in r.json().keys():
    print "Error: {}".format(r.json()['error']['data'])
    logout(server_url, request_id, auth_token)
    exit()
else:
    if len(r.json()['result']) > 0:
        host_template_id = r.json()['result'][0]['templateid']
        print "Template {} found on the server.".format(host_template)
    else:
        print "Template {} not found on the server. Exiting...".format(host_template)
        logout(server_url, request_id, auth_token)
        exit()

# check if group exists
request_id += 1
get_group_request = {
    "jsonrpc": "2.0",
    "method": "hostgroup.get",
    "params": {
        "output": "extend",
        "filter": {
            "name": [
                host_group
            ]
        }
    },
    "auth": auth_token,
    "id": request_id
}

r = requests.post(server_url, json=get_group_request)
if 'error' in r.json().keys():
    print "Error: {}".format(r.json()['error']['data'])
    logout(server_url, request_id, auth_token)
    exit()
else:
    if len(r.json()['result']) == 0:
        # Group is missing, registering
        print "Group {} not found on the server. Creating...".format(host_group)
        request_id += 1
        create_group_request = {
        "jsonrpc": "2.0",
        "method": "hostgroup.create",
        "params": {
            "name": host_group
        },
        "auth": auth_token,
        "id": request_id
        }
        r = requests.post(server_url, json=create_group_request)
        if len(r.json()['result']['groupids']) > 0:
            host_group_id = r.json()['result']['groupids'][0]
            print "Group {} with id {} is successfully created.".format(host_group, host_group_id)
        else:
            print "Error while creating group {}: {}".format(host_group, r.json()['result'])
    else:
        # Group exists
        host_group_id = r.json()['result'][0]['groupid']
        print "Group {} exists.".format(host_group)

# check if host exists
request_id += 1
get_host_request = {
    "jsonrpc": "2.0",
    "method": "host.get",
    "params": {
        "output": "extend",
        "filter": {
            "host": [
                host_name
            ]
        }
    },
    "auth": auth_token,
    "id": request_id
}

r = requests.post(server_url, json=get_host_request)
if 'error' in r.json().keys():
    print "Error: {}".format(r.json()['error']['data'])
    logout(server_url, request_id, auth_token)
    exit()
else:
    if len(r.json()['result']) == 0:
        # Host is missing, registering
        print "Host {} is not registered. Performing registration...".format(host_name)
        request_id += 1
        register_host_request = {
            "jsonrpc": "2.0",
            "method": "host.create",
            "params": {
                "host": host_name,
                "interfaces": [
                    {
                        "type": 1,
                        "main": 1,
                        "useip": 1,
                        "ip": host_ip,
                        "dns": "",
                        "port": "10050"
                    }
                ],
                "groups": [
                    {
                        "groupid": host_group_id
                    }
                ],
                "templates": [
                    {
                        "templateid": host_template_id
                    }
                ],
                "inventory_mode": 0,
            },
            "auth": auth_token,
            "id": request_id
        }
        r = requests.post(server_url, json=register_host_request)
        if 'error' in r.json().keys():
            print "Error: {}".format(r.json()['error']['data'])
            logout(server_url, request_id, auth_token)
            exit()
        else:
            if len(r.json()['result']['hostids']) > 0:
                print "Host {} is successfully created.".format(host_name)
            else:
                print "Error while creating host {}: {}".format(host_name, r.json()['result'])
    else:
        print "Host {} exists.".format(host_name)

# all is done, logout
logout(server_url, request_id, auth_token)
```
3. Initial conditions:
* Hosts:

![](https://raw.githubusercontent.com/shreben/zabbix/api/screenshots/api_initial_hosts.png)

* Host Groups:


![](https://raw.githubusercontent.com/shreben/zabbix/api/screenshots/api_initial_hostgroups.png)

* Templates:


![](https://raw.githubusercontent.com/shreben/zabbix/api/screenshots/api_initial_templates.png)

4. Spinned up test VM. [node2_up.txt](https://github.com/shreben/zabbix/blob/api/node2_up.txt) contains vagrant output:
```
Bringing machine 'node2' up with 'virtualbox' provider...
==> node2: Cloning VM...

[K==> node2: Matching MAC address for NAT networking...
==> node2: Setting the name of the VM: node2
==> node2: Fixed port collision for 22 => 2222. Now on port 2203.
==> node2: Clearing any previously set network interfaces...
==> node2: Preparing network interfaces based on configuration...
    node2: Adapter 1: nat
    node2: Adapter 2: hostonly
==> node2: Forwarding ports...
    node2: 22 (guest) => 2203 (host) (adapter 1)
==> node2: Booting VM...
==> node2: Waiting for machine to boot. This may take a few minutes...
    node2: SSH address: 127.0.0.1:2203
    node2: SSH username: vagrant
    node2: SSH auth method: private key
    node2: Warning: Remote connection disconnect. Retrying...
    node2: Warning: Remote connection disconnect. Retrying...
    node2: Warning: Remote connection disconnect. Retrying...
    node2: Warning: Remote connection disconnect. Retrying...
    node2: 
    node2: Vagrant insecure key detected. Vagrant will automatically replace
    node2: this with a newly generated keypair for better security.
    node2: 
    node2: Inserting generated public key within guest...
    node2: Removing insecure key from the guest if it's present...
    node2: Key inserted! Disconnecting and reconnecting using new SSH key...
==> node2: Machine booted and ready!
==> node2: Checking for guest additions in VM...
==> node2: Setting hostname...
==> node2: Configuring and enabling network interfaces...
==> node2: Mounting shared folders...
    node2: /vagrant => /home/student/zabbix
==> node2: Running provisioner: shell...
    node2: Running: inline script
==> node2: Loaded plugins: fastestmirror
==> node2: Examining /var/tmp/yum-root-qfFFH9/zabbix-release-3.2-1.el7.noarch.rpm: zabbix-release-3.2-1.el7.noarch
==> node2: Marking /var/tmp/yum-root-qfFFH9/zabbix-release-3.2-1.el7.noarch.rpm to be installed
==> node2: Resolving Dependencies
==> node2: --> Running transaction check
==> node2: ---> Package zabbix-release.noarch 0:3.2-1.el7 will be installed
==> node2: --> Finished Dependency Resolution
==> node2: 
==> node2: Dependencies Resolved
==> node2: 
==> node2: ================================================================================
==> node2:  Package          Arch     Version     Repository                          Size
==> node2: ================================================================================
==> node2: Installing:
==> node2:  zabbix-release   noarch   3.2-1.el7   /zabbix-release-3.2-1.el7.noarch    21 k
==> node2: 
==> node2: Transaction Summary
==> node2: ================================================================================
==> node2: Install  1 Package
==> node2: 
==> node2: Total size: 21 k
==> node2: Installed size: 21 k
==> node2: Downloading packages:
==> node2: Running transaction check
==> node2: Running transaction test
==> node2: Transaction test succeeded
==> node2: Running transaction
==> node2:   Installing : zabbix-release-3.2-1.el7.noarch                              1/1
==> node2:  
==> node2:   Verifying  : zabbix-release-3.2-1.el7.noarch                              1/1
==> node2:  
==> node2: 
==> node2: Installed:
==> node2:   zabbix-release.noarch 0:3.2-1.el7                                             
==> node2: Complete!
==> node2: Loaded plugins: fastestmirror
==> node2: Determining fastest mirrors
==> node2:  * base: ftp.byfly.by
==> node2:  * epel: mirror.datacenter.by
==> node2:  * extras: ftp.byfly.by
==> node2:  * updates: ftp.byfly.by
==> node2: Resolving Dependencies
==> node2: --> Running transaction check
==> node2: ---> Package net-tools.x86_64 0:2.0-0.17.20131004git.el7 will be installed
==> node2: --> Finished Dependency Resolution
==> node2: 
==> node2: Dependencies Resolved
==> node2: 
==> node2: ================================================================================
==> node2:  Package         Arch         Version                          Repository  Size
==> node2: ================================================================================
==> node2: Installing:
==> node2:  net-tools       x86_64       2.0-0.17.20131004git.el7         base       304 k
==> node2: 
==> node2: Transaction Summary
==> node2: ================================================================================
==> node2: Install  1 Package
==> node2: Total download size: 304 k
==> node2: Installed size: 917 k
==> node2: Downloading packages:
==> node2: Running transaction check
==> node2: Running transaction test
==> node2: Transaction test succeeded
==> node2: Running transaction
==> node2:   Installing : net-tools-2.0-0.17.20131004git.el7.x86_64                    1/1
==> node2:  
==> node2:   Verifying  : net-tools-2.0-0.17.20131004git.el7.x86_64                    1/1
==> node2:  
==> node2: 
==> node2: Installed:
==> node2:   net-tools.x86_64 0:2.0-0.17.20131004git.el7                                   
==> node2: 
==> node2: Complete!
==> node2: Loaded plugins: fastestmirror
==> node2: Loading mirror speeds from cached hostfile
==> node2:  * base: ftp.byfly.by
==> node2:  * epel: mirror.datacenter.by
==> node2:  * extras: ftp.byfly.by
==> node2:  * updates: ftp.byfly.by
==> node2: Resolving Dependencies
==> node2: --> Running transaction check
==> node2: ---> Package zabbix-agent.x86_64 0:3.2.4-2.el7 will be installed
==> node2: --> Finished Dependency Resolution
==> node2: 
==> node2: Dependencies Resolved
==> node2: 
==> node2: ================================================================================
==> node2:  Package              Arch           Version               Repository      Size
==> node2: ================================================================================
==> node2: Installing:
==> node2:  zabbix-agent         x86_64         3.2.4-2.el7           zabbix         342 k
==> node2: 
==> node2: Transaction Summary
==> node2: ================================================================================
==> node2: Install  1 Package
==> node2: 
==> node2: Total download size: 342 k
==> node2: Installed size: 1.3 M
==> node2: Downloading packages:
==> node2: Public key for zabbix-agent-3.2.4-2.el7.x86_64.rpm is not installed
==> node2: warning: /var/cache/yum/x86_64/7/zabbix/packages/zabbix-agent-3.2.4-2.el7.x86_64.rpm: Header V4 RSA/SHA512 Signature, key ID a14fe591: NOKEY
==> node2: Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX-A14FE591
==> node2: Importing GPG key 0xA14FE591:
==> node2:  Userid     : "Zabbix LLC <packager@zabbix.com>"
==> node2:  Fingerprint: a184 8f53 52d0 22b9 471d 83d0 082a b56b a14f e591
==> node2:  Package    : zabbix-release-3.2-1.el7.noarch (installed)
==> node2:  From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX-A14FE591
==> node2: Running transaction check
==> node2: Running transaction test
==> node2: Transaction test succeeded
==> node2: Running transaction
==> node2:   Installing : zabbix-agent-3.2.4-2.el7.x86_64                              1/1
==> node2:  
==> node2:   Verifying  : zabbix-agent-3.2.4-2.el7.x86_64                              1/1
==> node2:  
==> node2: 
==> node2: Installed:
==> node2:   zabbix-agent.x86_64 0:3.2.4-2.el7                                             
==> node2: Complete!
==> node2: Loaded plugins: fastestmirror
==> node2: Loading mirror speeds from cached hostfile
==> node2:  * base: ftp.byfly.by
==> node2:  * epel: mirror.datacenter.by
==> node2:  * extras: ftp.byfly.by
==> node2:  * updates: ftp.byfly.by
==> node2: Resolving Dependencies
==> node2: --> Running transaction check
==> node2: ---> Package python2-pip.noarch 0:8.1.2-5.el7 will be installed
==> node2: --> Processing Dependency: python-setuptools for package: python2-pip-8.1.2-5.el7.noarch
==> node2: --> Running transaction check
==> node2: ---> Package python-setuptools.noarch 0:0.9.8-4.el7 will be installed
==> node2: --> Processing Dependency: python-backports-ssl_match_hostname for package: python-setuptools-0.9.8-4.el7.noarch
==> node2: --> Running transaction check
==> node2: ---> Package python-backports-ssl_match_hostname.noarch 0:3.4.0.2-4.el7 will be installed
==> node2: --> Processing Dependency: python-backports for package: python-backports-ssl_match_hostname-3.4.0.2-4.el7.noarch
==> node2: --> Running transaction check
==> node2: ---> Package python-backports.x86_64 0:1.0-8.el7 will be installed
==> node2: --> Finished Dependency Resolution
==> node2: 
==> node2: Dependencies Resolved
==> node2: 
==> node2: ================================================================================
==> node2:  Package                               Arch     Version            Repository
==> node2:                                                                            Size
==> node2: ================================================================================
==> node2: Installing:
==> node2:  python2-pip                           noarch   8.1.2-5.el7        epel   1.7 M
==> node2: Installing for dependencies:
==> node2:  python-backports                      x86_64   1.0-8.el7          base   5.8 k
==> node2:  python-backports-ssl_match_hostname   noarch   3.4.0.2-4.el7      base    12 k
==> node2:  python-setuptools                     noarch   0.9.8-4.el7        base   396 k
==> node2: 
==> node2: Transaction Summary
==> node2: ================================================================================
==> node2: Install  1 Package (+3 Dependent packages)
==> node2: Total download size: 2.1 M
==> node2: Installed size: 9.1 M
==> node2: Downloading packages:
==> node2: warning: /var/cache/yum/x86_64/7/epel/packages/python2-pip-8.1.2-5.el7.noarch.rpm: Header V3 RSA/SHA256 Signature, key ID 352c64e5: NOKEY
==> node2: Public key for python2-pip-8.1.2-5.el7.noarch.rpm is not installed
==> node2: --------------------------------------------------------------------------------
==> node2: Total                                              917 kB/s | 2.1 MB  00:02     
==> node2: Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
==> node2: Importing GPG key 0x352C64E5:
==> node2:  Userid     : "Fedora EPEL (7) <epel@fedoraproject.org>"
==> node2:  Fingerprint: 91e9 7d7c 4a5e 96f1 7f3e 888f 6a2f aea2 352c 64e5
==> node2:  Package    : epel-release-7-9.noarch (installed)
==> node2:  From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
==> node2: Running transaction check
==> node2: Running transaction test
==> node2: Transaction test succeeded
==> node2: Running transaction
==> node2:   Installing : python-backports-1.0-8.el7.x86_64                            1/4
==> node2:  
==> node2:   Installing : python-backports-ssl_match_hostname-3.4.0.2-4.el7.noarch     2/4
==> node2:  
==> node2:   Installing : python-setuptools-0.9.8-4.el7.noarch                         3/4
==> node2:  
==> node2:   Installing : python2-pip-8.1.2-5.el7.noarch                               4/4
==> node2:  
==> node2:   Verifying  : python2-pip-8.1.2-5.el7.noarch                               1/4
==> node2:  
==> node2:   Verifying  : python-setuptools-0.9.8-4.el7.noarch                         2/4
==> node2:  
==> node2:   Verifying  : python-backports-1.0-8.el7.x86_64                            3/4
==> node2:  
==> node2:   Verifying  : python-backports-ssl_match_hostname-3.4.0.2-4.el7.noarch     4/4
==> node2:  
==> node2: 
==> node2: Installed:
==> node2:   python2-pip.noarch 0:8.1.2-5.el7                                              
==> node2: 
==> node2: Dependency Installed:
==> node2:   python-backports.x86_64 0:1.0-8.el7                                           
==> node2:   python-backports-ssl_match_hostname.noarch 0:3.4.0.2-4.el7                    
==> node2:   python-setuptools.noarch 0:0.9.8-4.el7                                        
==> node2: Complete!
==> node2: Collecting requests
==> node2:   Downloading requests-2.13.0-py2.py3-none-any.whl (584kB)
==> node2: Installing collected packages: requests
==> node2: Successfully installed requests-2.13.0
==> node2: You are using pip version 8.1.2, however version 9.0.1 is available.
==> node2: You should consider upgrading via the 'pip install --upgrade pip' command.
==> node2: Created symlink from /etc/systemd/system/multi-user.target.wants/zabbix-agent.service to /usr/lib/systemd/system/zabbix-agent.service.
==> node2: Template MTN Lab CloudHosts found on the server.
==> node2: Group CloudHosts not found on the server. Creating...
==> node2: Group CloudHosts with id 11 is successfully created.
==> node2: Host node2.mtn.lab is not registered. Performing registration...
==> node2: Host node2.mtn.lab is successfully created.
==> node2: Logged out.
```
5. After VM provision:
* Hosts:

![](https://raw.githubusercontent.com/shreben/zabbix/api/screenshots/api_result_hosts.png)

* Host Groups:


![](https://raw.githubusercontent.com/shreben/zabbix/api/screenshots/api_result_hostgroups.png)

* Templates:


![](https://raw.githubusercontent.com/shreben/zabbix/api/screenshots/api_result_templates.png)

* Latest data:


![](https://raw.githubusercontent.com/shreben/zabbix/api/screenshots/api_latest_data.png)
