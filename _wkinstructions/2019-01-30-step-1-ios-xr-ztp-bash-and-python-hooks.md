---
published: true
date: '2018-06-12 08:47 -0400'
title: 'Step 1: IOS-XR ZTP Bash and Python hooks (+Ansible!)'
author: Akshat Sharma
tags:
  - iosxr
  - cisco
  - devnet
  - CLUS2018
  - programmability
excerpt: Using ZTP hooks in bash and python to automate IOS-XR cli
---


{% include toc %}

## To Know More

As part of the Zero-touch provisioning infrastructure, IOS-XR provides the option to automate CLI operations in either bash or python in the IOS-XR shell environment. This enables a large variety of integrations - including native scripts and offbox integrations with tools like Ansible

Complete CLI support: 
*  Automate CLI operations such as "show commands", "config merge", "config replace"
*  Available in bash and python: Choose the scripting language you prefer and integrate with ztp, cronjobs, onbox-apps and more


>Check out:
>
>1) [IOS-XR Bash ZTP hooks](https://xrdocs.io/software-management/tutorials/2016-08-26-working-with-ztp/#ztp_helpersh)
>2) [IOS-XR Python ZTP library](https://xrdocs.io/software-management/tutorials/2016-08-26-working-with-ztp/#ztp_helpersh)


>Connect to your Pod first! Make sure your Anyconnect VPN connection to the Pod assigned to you is active. 
>
> If you haven't connected yet, check out the instructions to do so here: 
><https://iosxr-devnet-ciscolive.github.io/clus2019-workshop/assets/CLUS-19-Akshat-IOS-XR-Programmability.pdf>
>
>
> Once you're connected, use the following instructions to connect to the individual nodes.
> The instructions in the workshop will simply refer to the Name of the box to connect without
> repeating the connection details and credentials. So refer back to this list when you need it.
>  
>
> The 3 nodes in the topology are: 
> 
><p style="font-size: 16px;"><b>Development Linux System (DevBox)</b></p> 
>      IP Address: 10.10.20.170
>      Username/Password: [admin/admin]
>      SSH Port: 2211
> 
>
><p style="font-size: 16px;"><b>IOS-XRv9000 R1: (Router r1)</b></p> 
>
>     IP Address: 10.10.20.170  
>     Username/Password: [admin/admin]   
>     Management IP: 10.10.20.170  
>     XR SSH Port: 2221    
>     NETCONF Port: 8321   
>     gRPC Port: 57021  
>     XR-Bash SSH Port: 2222    
>
>
><p style="font-size: 16px;"><b>IOS-XRv9000 R2:  (Router r2)</b></p> 
>
>     IP Address: 10.10.20.170   
>     Username/Password: [admin/admin]   
>     Management IP: 10.10.20.170   
>     XR SSH Port: 2231    
>     NETCONF Port: 8331   
>     gRPC Port: 57031    
>     XR-Bash SSH Port: 2232
{: .notice--info}


The Topology in use is shown below:
![topology_devnet.png]({{site.baseurl}}/images/topology_devnet.png)




## SSH into the devbox 

Drop into the devbox using the credentials above and clone the following git repository:
<https://github.com/akshshar/iosxr-devnet-clus2019>


```

AKSHSHAR-M-33WP:~ akshshar$ 
AKSHSHAR-M-33WP:~ akshshar$ ssh -p 2211 admin@10.10.20.170
admin@10.10.20.170's password: 
Last login: Tue Jan 29 18:33:43 2019 from 192.168.122.1
admin@devbox:~$ 
admin@devbox:~$ 
admin@devbox:~$ 
admin@devbox:~$ git clone https://github.com/akshshar/iosxr-devnet-clus2019.git
Cloning into 'iosxr-devnet-clus2019'...
remote: Enumerating objects: 33, done.
remote: Counting objects: 100% (33/33), done.
remote: Compressing objects: 100% (23/23), done.
remote: Total 33 (delta 8), reused 33 (delta 8), pack-reused 0
Unpacking objects: 100% (33/33), done.
Checking connectivity... done.
admin@devbox:~$ 
admin@devbox:~$ 

```

You should see the following files:

```
admin@devbox:iosxr-devnet-clus2019$ tree .
.
├── ansible
│   ├── ansible_hosts
│   ├── ansible_netconf.yml
│   ├── combined_playbook.yml
│   ├── configure_bgp_netconf.yml
│   ├── docker_bringup.yml
│   ├── execute_python_ztp.yml
│   ├── openr
│   │   ├── docker_base.sysconfig
│   │   ├── docker.sysconfig
│   │   ├── hosts_r1
│   │   ├── hosts_r2
│   │   ├── increment_ipv4_prefix1.py
│   │   ├── increment_ipv4_prefix2.py
│   │   ├── launch_openr_r1.sh
│   │   ├── launch_openr_r2.sh
│   │   ├── run_openr_r1.sh
│   │   └── run_openr_r2.sh
│   ├── openr_setup.yml
│   ├── set_ipv6_route.sh
│   ├── setup_dependencies.yml
│   ├── setup_insecure_registry.yml
│   ├── xml
│   │   ├── r1-bgp.xml
│   │   └── r2-bgp.xml
│   └── yang_config
│       ├── r1.xml
│       └── r2.xml
├── README.md
├── ydk
│   └── configure_telemetry_openconfig.py
└── ztp_hooks
    ├── automate_cli_bash.sh
    └── automate_cli_python.py

6 directories, 28 files
admin@devbox:iosxr-devnet-clus2019$ 
```


cd into the ztp_hooks directory and you should a couple of files we will deal with in this section

```
admin@devbox:iosxr-devnet-clus2019$ 
admin@devbox:iosxr-devnet-clus2019$ cd ztp_hooks/
admin@devbox:ztp_hooks$ ls
automate_cli_bash.sh  automate_cli_python.py
admin@devbox:ztp_hooks$ 

```

## Bash ZTP script
The bash script will utilize the ZTP bash hooks in IOS-XR to configure `Loopback0` and `grpc` on each router.
Further it will configure and bring-up interfaces Gig0/0/0/0, Gig0/0/0/1 and Gig0/0/0/2 on each router. These are the b2b connected interfaces of the router r1 and r2.

>To learn about all the ZTP Bash hooks available in IOS-XR use the following learning lab on DevNet:
><https://learninglabs.cisco.com/tracks/iosxr-programmability/iosxr-cli-automation/01-iosxr-01-cli-automation-bash/step/1>
{: .notice--warning}



### Transfer the bash script to Router r1 over SSH

password for `Router r1` is `admin`
{: .notice--info}  


```
admin@devbox:ztp_hooks$ 
admin@devbox:ztp_hooks$ pwd
/home/admin/iosxr-devnet-clus2019/ztp_hooks
admin@devbox:ztp_hooks$ 
admin@devbox:ztp_hooks$ scp -P 2221 automate_cli_bash.sh  admin@10.10.20.170:/misc/scratch/


--------------------------------------------------------------------------
  Router 1 (Cisco IOS XR Sandbox)
--------------------------------------------------------------------------


Password: 
automate_cli_bash.sh                                                                                   100%  838     0.8KB/s   00:00    
Connection to 10.10.20.170 closed by remote host.
admin@devbox:ztp_hooks$ 

```


### Execute the bash script on router r1 over SSH

```
admin@devbox:ztp_hooks$ ssh -p 2221  admin@10.10.20.170 run /misc/scratch/automate_cli_bash.sh


--------------------------------------------------------------------------
  Router 1 (Cisco IOS XR Sandbox)
--------------------------------------------------------------------------


Password: 


Wed Jan 30 06:03:41.016 UTC
Building configuration...
!! IOS XR Configuration version = 6.4.1
interface Loopback0
 ipv4 address 50.1.1.1 255.255.255.255
!
interface GigabitEthernet0/0/0/0
 ipv4 address 10.1.1.10 255.255.255.0
 ipv6 enable
!
interface GigabitEthernet0/0/0/1
 ipv4 address 11.1.1.10 255.255.255.0
 ipv6 enable
!
interface GigabitEthernet0/0/0/2
 ipv4 address 12.1.1.10 255.255.255.0
 ipv6 enable
!
grpc
 port 57777
 no-tls
 service-layer
 !
!
end

admin@devbox:ztp_hooks$ 
```



Let's do the same for router r2.

### Transfer the bash script to Router r2 over SSH



```
admin@devbox:ztp_hooks$ pwd
/home/admin/iosxr-devnet-clus2019/ztp_hooks
admin@devbox:ztp_hooks$ 
admin@devbox:ztp_hooks$ 
admin@devbox:ztp_hooks$ scp -P 2231 automate_cli_bash.sh  admin@10.10.20.170:/misc/scratch/


--------------------------------------------------------------------------
  Router 2 (Cisco IOS XR Sandbox)
--------------------------------------------------------------------------


Password: 
automate_cli_bash.sh                                                                                   100%  838     0.8KB/s   00:00    
Connection to 10.10.20.170 closed by remote host.
admin@devbox:ztp_hooks$ 



```

### Execute the bash script on router r2 over SSH


```
admin@devbox:ztp_hooks$ ssh -p 2231  admin@10.10.20.170 run /misc/scratch/automate_cli_bash.sh


--------------------------------------------------------------------------
  Router 2 (Cisco IOS XR Sandbox)
--------------------------------------------------------------------------


Password: 


Wed Jan 30 06:03:59.093 UTC
Building configuration...
!! IOS XR Configuration version = 6.4.1
interface Loopback0
 ipv4 address 60.1.1.1 255.255.255.255
!
interface GigabitEthernet0/0/0/0
 ipv4 address 10.1.1.20 255.255.255.0
 ipv6 enable
!
interface GigabitEthernet0/0/0/1
 ipv4 address 11.1.1.20 255.255.255.0
 ipv6 enable
!
interface GigabitEthernet0/0/0/2
 ipv4 address 12.1.1.20 255.255.255.0
 ipv6 enable
!
grpc
 port 57777
 no-tls
 service-layer
 !
!
end

admin@devbox:ztp_hooks$ 
 
```




Great, so we know how to run ZTP-API based scripts on the box. In the steps above, we did so over SSH.
But it can soon grow to be cumbersome, if you have to manually transfer the scripts to each router in the topology and then execute over SSH. Further, the manual password entry greatly reduces the speed at which these actions can be performed and is not a scalable automation technique.
So for the python ZTP scripts we will deal with below, we scale out the process using Ansible
{: .notice--success}




## Python ZTP hooks

The python ZTP hooks script we intend to use is under `ztp_hooks/` directory in the git repository we cloned earlier:

```
admin@devbox:~$ 
admin@devbox:~$ cd ~/iosxr-devnet-clus2019/
admin@devbox:iosxr-devnet-clus2019$ cd ztp_hooks/
admin@devbox:ztp_hooks$ ls
automate_cli_bash.sh  automate_cli_python.py
admin@devbox:ztp_hooks$

```

Open up the script to understand the code as the execution takes place. You can open up another ssh session to the devbox in a separate terminal tab for this purpose.

The script will use different ZTP python APIs in IOS-XR to do CLI operations such as `xrcmd`(Show commands) and `xrapply`(Merge configuration). `xrreplace` is not shown but it can be used to the Replace existing configuration with a specified snippet.

Eventually the script will push the following configuration on each router:

```

!! IOS XR Configuration version = 6.4.1
domain name-server 8.8.8.8
tpa
 vrf default
  address-family ipv4
   default-route mgmt
   update-source dataports MgmtEth0/RP0/CPU0/0
  !
 !
!
end


```

Further the script will restart the docker daemon on the host for the routing changes to take effect and finally pull the docker image for Open/R to be run in the last section of the workshop.


### Execute python ZTP script using Ansible


Hop into the `ansible/` directory of the git repository we cloned earlier. The ansible playbook we intend to use is shown below (`execute_python_ztp.yml`):


### Dump the contents of the Ansible playbook

```
admin@devbox:~$ cd iosxr-devnet-clus2019/
admin@devbox:iosxr-devnet-clus2019$ ls
ansible  README.md  ztp_hooks
admin@devbox:iosxr-devnet-clus2019$ cd ansible/
admin@devbox:ansible$ 
admin@devbox:ansible$ cat execute_python_ztp.yml 
---
- hosts: routers_shell 
  strategy: debug
  become: yes
  gather_facts: no

  tasks:
  - debug: msg="hostname={{hostname}}"
  - name: Copy and Execute the Python Configuration script on the router
    script: ../ztp_hooks/automate_cli_python.py
    register: output

  - debug:
        var: output.stdout_lines
admin@devbox:ansible$ 


```

### Run the Ansible playbook 

**IMPORTANT:** Before you run the ansible playbook, make sure you set the ANSIBLE_HOST_KEY_CHECKING 
environment variable to false to allow Ansible to easily connect without being stalled by key
checking requirements for the two routers. This can also be set in the ansible_cfg file instead.
```
admin@devbox:ansible$ 
admin@devbox:ansible$ export ANSIBLE_HOST_KEY_CHECKING=False
```  
{: .notice--danger} . 

Now, execute the ansible playbook, which will automatically transfer the python script to the shell of each router based on the `ansible_hosts` file which stores the credentials and connection information.


```
admin@devbox:ansible$ cat ansible_hosts 
[routers_shell]
r1 ansible_user="admin" ansible_password="admin" ansible_sudo_pass="admin" ansible_host=10.10.20.170 ansible_port=2222 hostname=r1 netconf_port=8321 xml_file="./xml/r1-bgp.xml" run_openr_script="./openr/run_openr_r1.sh" launch_openr_script="./openr/launch_openr_r1.sh" hosts_r="./openr/hosts_r1" increment_ipv4_prefix="./openr/increment_ipv4_prefix1.py" cron_file="./set_ipv6_route.sh" base_xml_file="./yang_config/r1.xml" docker_sysconfig_file="./openr/docker.sysconfig"

r2 ansible_user="admin" ansible_sudo_pass="admin" ansible_password="admin" ansible_host=10.10.20.170 ansible_port=2232 hostname=r2 netconf_port=8331 xml_file="./xml/r2-bgp.xml" run_openr_script="./openr/run_openr_r2.sh" launch_openr_script="./openr/launch_openr_r2.sh" hosts_r="./openr/hosts_r2" increment_ipv4_prefix="./openr/increment_ipv4_prefix2.py" cron_file="./set_ipv6_route.sh" base_xml_file="./yang_config/r2.xml" docker_sysconfig_file="./openr/docker.sysconfig"

[devbox-host]
devbox ansible_user="admin" ansible_password="admin" ansible_sudo_user="admin" ansible_sudo_pass="admin" ansible_host=localhost
admin@devbox:ansible$ 

```

When we run the playbook, wait for some time before the `ansible-playbook` returns the output of the script run on each router:


```
admin@devbox:ansible$ ansible-playbook -i ansible_hosts execute_python_ztp.yml

PLAY [routers_shell] ***********************************************************************************************************

TASK [debug] *******************************************************************************************************************
ok: [r2] => {
    "msg": "hostname=r2"
}
ok: [r1] => {
    "msg": "hostname=r1"
}

TASK [Copy and Execute the Python Configuration script on the router] **********************************************************
changed: [r2]
changed: [r1]

TASK [debug] *******************************************************************************************************************
ok: [r1] => {
    "output.stdout_lines": [
        "", 
        "###### Debugs enabled ######", 
        "", 
        "", 
        "###### Using Child class method, creating a new user ######", 
        "", 
        "2019-06-12 17:46:43,331 - DebugZTPLogger - DEBUG - Config File content to be applied  !", 
        "                     username vagrant ", 
        "                     group root-lr", 
        "                     group cisco-support", 
        "                     secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1 ", 
        "                     !", 
        "                     end", 
        "2019-06-12 17:46:48,353 - DebugZTPLogger - DEBUG - Received exec command request: \"show configuration commit changes last 1\"", 
        "2019-06-12 17:46:48,353 - DebugZTPLogger - DEBUG - Response to any expected prompt \"\"", 
        "Building configuration...", 
        "2019-06-12 17:46:49,783 - DebugZTPLogger - DEBUG - Exec command output is ['!! IOS XR Configuration version = 6.4.1', 'username vagrant', 'group root-lr', 'group cisco-support', 'secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1', '!', 'end']", 
        "2019-06-12 17:46:49,783 - DebugZTPLogger - DEBUG - Config apply through file successful, last change = ['!! IOS XR Configuration version = 6.4.1', 'username vagrant', 'group root-lr', 'group cisco-support', 'secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1', '!', 'end']", 
        "", 
        "###### New user successfully created, return value: ######", 
        "", 
        "['!! IOS XR Configuration version = 6.4.1',", 
        " 'username vagrant',", 
        " 'group root-lr',", 
        " 'group cisco-support',", 
        " 'secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1',", 
        " '!',", 
        " 'end']", 
        "", 
        "###### return value in json: ######", 
        "", 
        "[", 
        "    \"!! IOS XR Configuration version = 6.4.1\", ", 
        "    \"username vagrant\", ", 
        "    \"group root-lr\", ", 
        "    \"group cisco-support\", ", 
        "    \"secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1\", ", 
        "    \"!\", ", 
        "    \"end\"", 
        "]", 
        "", 
        "###### Debugs Disabled  ######", 
        "", 
        "", 
        "###### Applying an incorrect config  ######", 
        "", 
        "", 
        "###### Failed to apply configuration, error is:######", 
        "", 
        "['!! SYNTAX/AUTHORIZATION ERRORS: This configuration failed due to',", 
        " '!! one or more of the following reasons:',", 
        " '!!  - the entered commands do not exist,',", 
        " '!!  - the entered commands have errors in their syntax,',", 
        " '!!  - the software packages containing the commands are not active,',", 
        " '!!  - the current user is not a member of a task-group that has',", 
        " '!!    permissions to use the commands.',", 
        " 'domain nameserver 8.8.8.8']", 
        "", 
        "###### error in json: ######", 
        "", 
        "[", 
        "    \"!! SYNTAX/AUTHORIZATION ERRORS: This configuration failed due to\", ", 
        "    \"!! one or more of the following reasons:\", ", 
        "    \"!!  - the entered commands do not exist,\", ", 
        "    \"!!  - the entered commands have errors in their syntax,\", ", 
        "    \"!!  - the software packages containing the commands are not active,\", ", 
        "    \"!!  - the current user is not a member of a task-group that has\", ", 
        "    \"!!    permissions to use the commands.\", ", 
        "    \"domain nameserver 8.8.8.8\"", 
        "]", 
        "", 
        "###### Applying the correct config  ######", 
        "", 
        "Building configuration...", 
        "", 
        "###### Successfully applied configuration, checking last commit######", 
        "", 
        "Building configuration...", 
        "['!! IOS XR Configuration version = 6.4.1',", 
        " 'domain name-server 8.8.8.8',", 
        " 'end']", 
        "", 
        "###### last commit in json: ######", 
        "", 
        "[", 
        "    \"!! IOS XR Configuration version = 6.4.1\", ", 
        "    \"domain name-server 8.8.8.8\", ", 
        "    \"end\"", 
        "]", 
        "", 
        "####### Applying tpa configuration to enable internet access from linux shell via management port ######", 
        "", 
        "Building configuration...", 
        "", 
        "###### tpa config successfully applied, response: ######", 
        "", 
        "['!! IOS XR Configuration version = 6.4.1',", 
        " 'tpa',", 
        " 'vrf default',", 
        " 'address-family ipv4',", 
        " 'default-route mgmt',", 
        " 'update-source dataports MgmtEth0/RP0/CPU0/0',", 
        " '!',", 
        " '!',", 
        " '!',", 
        " 'end']", 
        "", 
        "###### return value in json: ######", 
        "", 
        "[", 
        "    \"!! IOS XR Configuration version = 6.4.1\", ", 
        "    \"tpa\", ", 
        "    \"vrf default\", ", 
        "    \"address-family ipv4\", ", 
        "    \"default-route mgmt\", ", 
        "    \"update-source dataports MgmtEth0/RP0/CPU0/0\", ", 
        "    \"!\", ", 
        "    \"!\", ", 
        "    \"!\", ", 
        "    \"end\"", 
        "]"
    ]
}
ok: [r2] => {
    "output.stdout_lines": [
        "", 
        "###### Debugs enabled ######", 
        "", 
        "", 
        "###### Using Child class method, creating a new user ######", 
        "", 
        "2019-06-12 17:46:43,222 - DebugZTPLogger - DEBUG - Config File content to be applied  !", 
        "                     username vagrant ", 
        "                     group root-lr", 
        "                     group cisco-support", 
        "                     secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1 ", 
        "                     !", 
        "                     end", 
        "2019-06-12 17:46:48,309 - DebugZTPLogger - DEBUG - Received exec command request: \"show configuration commit changes last 1\"", 
        "2019-06-12 17:46:48,310 - DebugZTPLogger - DEBUG - Response to any expected prompt \"\"", 
        "Building configuration...", 
        "2019-06-12 17:46:49,873 - DebugZTPLogger - DEBUG - Exec command output is ['!! IOS XR Configuration version = 6.4.1', 'username vagrant', 'group root-lr', 'group cisco-support', 'secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1', '!', 'end']", 
        "2019-06-12 17:46:49,873 - DebugZTPLogger - DEBUG - Config apply through file successful, last change = ['!! IOS XR Configuration version = 6.4.1', 'username vagrant', 'group root-lr', 'group cisco-support', 'secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1', '!', 'end']", 
        "", 
        "###### New user successfully created, return value: ######", 
        "", 
        "['!! IOS XR Configuration version = 6.4.1',", 
        " 'username vagrant',", 
        " 'group root-lr',", 
        " 'group cisco-support',", 
        " 'secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1',", 
        " '!',", 
        " 'end']", 
        "", 
        "###### return value in json: ######", 
        "", 
        "[", 
        "    \"!! IOS XR Configuration version = 6.4.1\", ", 
        "    \"username vagrant\", ", 
        "    \"group root-lr\", ", 
        "    \"group cisco-support\", ", 
        "    \"secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1\", ", 
        "    \"!\", ", 
        "    \"end\"", 
        "]", 
        "", 
        "###### Debugs Disabled  ######", 
        "", 
        "", 
        "###### Applying an incorrect config  ######", 
        "", 
        "", 
        "###### Failed to apply configuration, error is:######", 
        "", 
        "['!! SYNTAX/AUTHORIZATION ERRORS: This configuration failed due to',", 
        " '!! one or more of the following reasons:',", 
        " '!!  - the entered commands do not exist,',", 
        " '!!  - the entered commands have errors in their syntax,',", 
        " '!!  - the software packages containing the commands are not active,',", 
        " '!!  - the current user is not a member of a task-group that has',", 
        " '!!    permissions to use the commands.',", 
        " 'domain nameserver 8.8.8.8']", 
        "", 
        "###### error in json: ######", 
        "", 
        "[", 
        "    \"!! SYNTAX/AUTHORIZATION ERRORS: This configuration failed due to\", ", 
        "    \"!! one or more of the following reasons:\", ", 
        "    \"!!  - the entered commands do not exist,\", ", 
        "    \"!!  - the entered commands have errors in their syntax,\", ", 
        "    \"!!  - the software packages containing the commands are not active,\", ", 
        "    \"!!  - the current user is not a member of a task-group that has\", ", 
        "    \"!!    permissions to use the commands.\", ", 
        "    \"domain nameserver 8.8.8.8\"", 
        "]", 
        "", 
        "###### Applying the correct config  ######", 
        "", 
        "Building configuration...", 
        "", 
        "###### Successfully applied configuration, checking last commit######", 
        "", 
        "Building configuration...", 
        "['!! IOS XR Configuration version = 6.4.1',", 
        " 'domain name-server 8.8.8.8',", 
        " 'end']", 
        "", 
        "###### last commit in json: ######", 
        "", 
        "[", 
        "    \"!! IOS XR Configuration version = 6.4.1\", ", 
        "    \"domain name-server 8.8.8.8\", ", 
        "    \"end\"", 
        "]", 
        "", 
        "####### Applying tpa configuration to enable internet access from linux shell via management port ######", 
        "", 
        "Building configuration...", 
        "", 
        "###### tpa config successfully applied, response: ######", 
        "", 
        "['!! IOS XR Configuration version = 6.4.1',", 
        " 'tpa',", 
        " 'vrf default',", 
        " 'address-family ipv4',", 
        " 'default-route mgmt',", 
        " 'update-source dataports MgmtEth0/RP0/CPU0/0',", 
        " '!',", 
        " '!',", 
        " '!',", 
        " 'end']", 
        "", 
        "###### return value in json: ######", 
        "", 
        "[", 
        "    \"!! IOS XR Configuration version = 6.4.1\", ", 
        "    \"tpa\", ", 
        "    \"vrf default\", ", 
        "    \"address-family ipv4\", ", 
        "    \"default-route mgmt\", ", 
        "    \"update-source dataports MgmtEth0/RP0/CPU0/0\", ", 
        "    \"!\", ", 
        "    \"!\", ", 
        "    \"!\", ", 
        "    \"end\"", 
        "]"
    ]
}

PLAY RECAP *********************************************************************************************************************
r1                         : ok=3    changed=1    unreachable=0    failed=0   
r2                         : ok=3    changed=1    unreachable=0    failed=0   

admin@devbox:ansible$ 

```



Perfect! We're all set for the next section of the lab where will look to levarage IOS-XR Yang models to configure BGP and set up a telemetry session.
{: .notice--success}
