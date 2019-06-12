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
admin@devbox:~$ cd iosxr-devnet-clus2019/
admin@devbox:iosxr-devnet-clus2019$ 
admin@devbox:iosxr-devnet-clus2019$ tree .
.
├── ansible
│   ├── ansible_hosts
│   ├── configure_bgp_oc_netconf.yml
│   ├── docker_bringup.retry
│   ├── docker_bringup.yml
│   ├── execute_python_ztp.yml
│   ├── openr
│   │   ├── hosts_r1
│   │   ├── hosts_r2
│   │   ├── increment_ipv4_prefix1.py
│   │   ├── increment_ipv4_prefix2.py
│   │   ├── launch_openr_r1.sh
│   │   ├── launch_openr_r2.sh
│   │   ├── run_openr_r1.sh
│   │   └── run_openr_r2.sh
│   ├── set_ipv6_route.sh
│   └── xml
│       ├── r1-bgp.xml
│       └── r2-bgp.xml
├── README.md
└── ztp_hooks
    ├── automate_cli_bash.sh
    └── automate_cli_python.py

4 directories, 19 files
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


## Install Ansible on the DevBox

Let's begin by installing Ansible on the Devbox. We will go with Ansible version=2.6.0 since it has been tested with this lab.

```
admin@devbox:~$ sudo pip install ansible==2.6.0
The directory '/home/admin/.cache/pip/http' or its parent directory is not owned by the current user and the cache has been disabled. Please check the permissions and owner of that directory. If executing pip with sudo, you may want sudo's -H flag.
The directory '/home/admin/.cache/pip' or its parent directory is not owned by the current user and caching wheels has been disabled. check the permissions and owner of that directory. If executing pip with sudo, you may want sudo's -H flag.
Collecting ansible==2.6.0
  Downloading https://files.pythonhosted.org/packages/c3/af/c86d456905284ecfce79736b55942470b42fdadea9150843e2eb51c2ecae/ansible-2.6.0.tar.gz (10.7MB)
    100% |████████████████████████████████| 10.7MB 1.3MB/s 
Collecting jinja2 (from ansible==2.6.0)
  Downloading https://files.pythonhosted.org/packages/7f/ff/ae64bacdfc95f27a016a7bed8e8686763ba4d277a78ca76f32659220a731/Jinja2-2.10-py2.py3-none-any.whl (126kB)
    100% |████████████████████████████████| 133kB 20.9MB/s 
Collecting PyYAML (from ansible==2.6.0)
  Downloading https://files.pythonhosted.org/packages/9e/a3/1d13970c3f36777c583f136c136f804d70f500168edc1edea6daa7200769/PyYAML-3.13.tar.gz (270kB)
    100% |████████████████████████████████| 276kB 21.7MB/s 
Collecting paramiko (from ansible==2.6.0)
  Downloading https://files.pythonhosted.org/packages/cf/ae/94e70d49044ccc234bfdba20114fa947d7ba6eb68a2e452d89b920e62227/paramiko-2.4.2-py2.py3-none-any.whl (193kB)
    100% |████████████████████████████████| 194kB 24.0MB/s 
Collecting cryptography (from ansible==2.6.0)
  Downloading https://files.pythonhosted.org/packages/98/71/e632e222f34632e0527dd41799f7847305e701f38f512d81bdf96009bca4/cryptography-2.5-cp34-abi3-manylinux1_x86_64.whl (2.4MB)
    100% |████████████████████████████████| 2.4MB 13.8MB/s 
Requirement already satisfied: setuptools in /usr/lib/python3/dist-packages (from ansible==2.6.0) (20.7.0)
Collecting MarkupSafe>=0.23 (from jinja2->ansible==2.6.0)
  Downloading https://files.pythonhosted.org/packages/3e/a5/e188980ef1d0a4e0788b5143ea933f9afd760df38fec4c0b72b5ae3060c8/MarkupSafe-1.1.0-cp35-cp35m-manylinux1_x86_64.whl
Collecting bcrypt>=3.1.3 (from paramiko->ansible==2.6.0)
  Downloading https://files.pythonhosted.org/packages/d0/79/79a4d167a31cc206117d9b396926615fa9c1fdbd52017bcced80937ac501/bcrypt-3.1.6-cp34-abi3-manylinux1_x86_64.whl (55kB)
    100% |████████████████████████████████| 61kB 12.8MB/s 
Collecting pynacl>=1.0.1 (from paramiko->ansible==2.6.0)
  Downloading https://files.pythonhosted.org/packages/27/15/2cd0a203f318c2240b42cd9dd13c931ddd61067809fee3479f44f086103e/PyNaCl-1.3.0-cp34-abi3-manylinux1_x86_64.whl (759kB)
    100% |████████████████████████████████| 768kB 15.8MB/s 
Collecting pyasn1>=0.1.7 (from paramiko->ansible==2.6.0)
  Downloading https://files.pythonhosted.org/packages/7b/7c/c9386b82a25115cccf1903441bba3cbadcfae7b678a20167347fa8ded34c/pyasn1-0.4.5-py2.py3-none-any.whl (73kB)
    100% |████████████████████████████████| 81kB 9.8MB/s 
Collecting asn1crypto>=0.21.0 (from cryptography->ansible==2.6.0)
  Downloading https://files.pythonhosted.org/packages/ea/cd/35485615f45f30a510576f1a56d1e0a7ad7bd8ab5ed7cdc600ef7cd06222/asn1crypto-0.24.0-py2.py3-none-any.whl (101kB)
    100% |████████████████████████████████| 102kB 17.1MB/s 
Requirement already satisfied: six>=1.4.1 in /usr/lib/python3/dist-packages (from cryptography->ansible==2.6.0) (1.10.0)
Collecting cffi!=1.11.3,>=1.8 (from cryptography->ansible==2.6.0)
  Downloading https://files.pythonhosted.org/packages/59/cc/0e1635b4951021ef35f5c92b32c865ae605fac2a19d724fb6ff99d745c81/cffi-1.11.5-cp35-cp35m-manylinux1_x86_64.whl (420kB)
    100% |████████████████████████████████| 430kB 18.0MB/s 
Collecting pycparser (from cffi!=1.11.3,>=1.8->cryptography->ansible==2.6.0)
  Downloading https://files.pythonhosted.org/packages/68/9e/49196946aee219aead1290e00d1e7fdeab8567783e83e1b9ab5585e6206a/pycparser-2.19.tar.gz (158kB)
    100% |████████████████████████████████| 163kB 12.5MB/s 
Installing collected packages: MarkupSafe, jinja2, PyYAML, asn1crypto, pycparser, cffi, cryptography, bcrypt, pynacl, pyasn1, paramiko, ansible
  Running setup.py install for PyYAML ... done
  Running setup.py install for pycparser ... done
  Running setup.py install for ansible ... done
Successfully installed MarkupSafe-1.1.0 PyYAML-3.13 ansible-2.6.0 asn1crypto-0.24.0 bcrypt-3.1.6 cffi-1.11.5 cryptography-2.5 jinja2-2.10 paramiko-2.4.2 pyasn1-0.4.5 pycparser-2.19 pynacl-1.3.0
You are using pip version 10.0.1, however version 19.0.1 is available.
You should consider upgrading via the 'pip install --upgrade pip' command.
admin@devbox:~$ 
```

Check the ansible version once installation is complete:

```
admin@devbox:~$ ansible --version
ansible 2.6.0
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/admin/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python3.5/dist-packages/ansible
  executable location = /usr/local/bin/ansible
  python version = 3.5.2 (default, Nov 12 2018, 13:43:14) [GCC 5.4.0 20160609]
admin@devbox:~$ 

```


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
admin@devbox:ansible$ 
admin@devbox:ansible$ cat ansible_hosts 
[routers_shell]
r1 ansible_user="admin" ansible_password="admin" ansible_sudo_pass="admin" ansible_host=10.10.20.170 ansible_port=2222 hostname=r1 netconf_port=8321 xml_file="./xml/r1-bgp.xml" run_openr_script="./openr/run_openr_r1.sh" launch_openr_script="./openr/launch_openr_r1.sh" hosts_r="./openr/hosts_r1" increment_ipv4_prefix="./openr/increment_ipv4_prefix1.py" cron_file="./set_ipv6_route.sh"

r2 ansible_user="admin" ansible_sudo_pass="admin" ansible_password="admin" ansible_host=10.10.20.170 ansible_port=2232 hostname=r2 netconf_port=8331 xml_file="./xml/r2-bgp.xml" run_openr_script="./openr/run_openr_r2.sh" launch_openr_script="./openr/launch_openr_r2.sh" hosts_r="./openr/hosts_r2" increment_ipv4_prefix="./openr/increment_ipv4_prefix2.py" cron_file="./set_ipv6_route.sh"
admin@devbox:ansible$ 

```

When we run the playbook, wait for some time before the `ansible-playbook` returns the output of the script run on each router:


```
admin@devbox:ansible$ 
admin@devbox:ansible$ ansible-playbook -i ansible_hosts execute_python_ztp.yml

PLAY [routers_shell] ******************************************************************************************************************

TASK [debug] **************************************************************************************************************************
ok: [r2] => {
    "msg": "hostname=r2"
}
ok: [r1] => {
    "msg": "hostname=r1"
}

TASK [Copy and Execute the Python Configuration script on the router] *****************************************************************
changed: [r2]
changed: [r1]

TASK [debug] **************************************************************************************************************************
ok: [r1] => {
    "output.stdout_lines": [
        "", 
        "###### Debugs enabled ######", 
        "", 
        "", 
        "###### Using Child class method, creating a new user ######", 
        "", 
        "2019-01-30 13:41:42,652 - DebugZTPLogger - DEBUG - Config File content to be applied  !", 
        "                     username vagrant ", 
        "                     group root-lr", 
        "                     group cisco-support", 
        "                     secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1 ", 
        "                     !", 
        "                     end", 
        "2019-01-30 13:41:48,489 - DebugZTPLogger - DEBUG - Received exec command request: \"show configuration commit changes last 1\"", 
        "2019-01-30 13:41:48,489 - DebugZTPLogger - DEBUG - Response to any expected prompt \"\"", 
        "Building configuration...", 
        "2019-01-30 13:41:50,291 - DebugZTPLogger - DEBUG - Exec command output is ['!! IOS XR Configuration version = 6.4.1', 'username vagrant', 'group root-lr', 'group cisco-support', 'secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1', '!', 'end']", 
        "2019-01-30 13:41:50,291 - DebugZTPLogger - DEBUG - Config apply through file successful, last change = ['!! IOS XR Configuration version = 6.4.1', 'username vagrant', 'group root-lr', 'group cisco-support', 'secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1', '!', 'end']", 
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
        "####### Applying tpa configuration to enable docker pull from docker.io######", 
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
        "]", 
        "", 
        "####### Finally restarting the docker daemon to make sure changes take effect######", 
        "", 
        "", 
        "###### Successfully restarted the docker daemon, response: ######", 
        "", 
        "['ztp-user connected from 127.0.0.1 using console on r1',", 
        " '\\x1b[?7hsysadmin-vm:0_RP0# run ssh 10.0.2.16 service docker restart',", 
        " 'Wed Jan  30 13:42:16.788 UTC',", 
        " 'docker stop/waiting',", 
        " 'docker start/running, process 18840',", 
        " 'sysadmin-vm:0_RP0#']", 
        "", 
        "###### return value in json: ######", 
        "", 
        "[", 
        "    \"ztp-user connected from 127.0.0.1 using console on r1\", ", 
        "    \"\\u001b[?7hsysadmin-vm:0_RP0# run ssh 10.0.2.16 service docker restart\", ", 
        "    \"Wed Jan  30 13:42:16.788 UTC\", ", 
        "    \"docker stop/waiting\", ", 
        "    \"docker start/running, process 18840\", ", 
        "    \"sysadmin-vm:0_RP0#\"", 
        "]", 
        "Sleeping for about 30 seconds, waiting for the docker daemon to be up", 
        "", 
        "#######Pulling the docker image for Open/R ######", 
        "", 
        "{'status': 1, 'output': 'Using default tag: latest\\n', 'error': 'Error response from daemon: Get https://registry-1.docker.io/v2/akshshar/openr-xr/manifests/latest: Get https://auth.docker.io/token?scope=repository%3Aakshshar%2Fopenr-xr%3Apull&service=registry.docker.io: net/http: request canceled while waiting for connection\\n'}"
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
        "2019-01-30 13:41:42,159 - DebugZTPLogger - DEBUG - Config File content to be applied  !", 
        "                     username vagrant ", 
        "                     group root-lr", 
        "                     group cisco-support", 
        "                     secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1 ", 
        "                     !", 
        "                     end", 
        "2019-01-30 13:41:47,312 - DebugZTPLogger - DEBUG - Received exec command request: \"show configuration commit changes last 1\"", 
        "2019-01-30 13:41:47,313 - DebugZTPLogger - DEBUG - Response to any expected prompt \"\"", 
        "Building configuration...", 
        "2019-01-30 13:41:49,057 - DebugZTPLogger - DEBUG - Exec command output is ['!! IOS XR Configuration version = 6.4.1', 'username vagrant', 'group root-lr', 'group cisco-support', 'secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1', '!', 'end']", 
        "2019-01-30 13:41:49,057 - DebugZTPLogger - DEBUG - Config apply through file successful, last change = ['!! IOS XR Configuration version = 6.4.1', 'username vagrant', 'group root-lr', 'group cisco-support', 'secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1', '!', 'end']", 
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
        "####### Applying tpa configuration to enable docker pull from docker.io######", 
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
        "]", 
        "", 
        "####### Finally restarting the docker daemon to make sure changes take effect######", 
        "", 
        "", 
        "###### Successfully restarted the docker daemon, response: ######", 
        "", 
        "['ztp-user connected from 127.0.0.1 using console on r2',", 
        " '\\x1b[?7hsysadmin-vm:0_RP0# run ssh 10.0.2.16 service docker restart',", 
        " 'Wed Jan  30 13:42:14.295 UTC',", 
        " 'docker stop/waiting',", 
        " 'docker start/running, process 13980',", 
        " 'sysadmin-vm:0_RP0#']", 
        "", 
        "###### return value in json: ######", 
        "", 
        "[", 
        "    \"ztp-user connected from 127.0.0.1 using console on r2\", ", 
        "    \"\\u001b[?7hsysadmin-vm:0_RP0# run ssh 10.0.2.16 service docker restart\", ", 
        "    \"Wed Jan  30 13:42:14.295 UTC\", ", 
        "    \"docker stop/waiting\", ", 
        "    \"docker start/running, process 13980\", ", 
        "    \"sysadmin-vm:0_RP0#\"", 
        "]", 
        "Sleeping for about 30 seconds, waiting for the docker daemon to be up", 
        "", 
        "#######Pulling the docker image for Open/R ######", 
        "", 
        "{'status': 0, 'output': 'Using default tag: latest\\nlatest: Pulling from akshshar/openr-xr\\nDigest: sha256:0d81b575830fe776739f960870652c7d9da601eaf32f68fa5569e852a2c5d4b0\\nStatus: Image is up to date for akshshar/openr-xr:latest\\n', 'error': ''}"
    ]
}

PLAY RECAP ****************************************************************************************************************************
r1                         : ok=3    changed=1    unreachable=0    failed=0   
r2                         : ok=3    changed=1    unreachable=0    failed=0   

admin@devbox:ansible$ 
admin@devbox:ansible$
```



Perfect! We're all set for the next section of the lab where will look to levarage IOS-XR Yang models to configure BGP and set up a telemetry session.
{: .notice--success}
