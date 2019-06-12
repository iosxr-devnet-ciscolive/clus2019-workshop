---
published: true
date: '2018-06-12 10:13 -0400'
title: 'Step 3:  Combining Telemetry with Ansible and Open/R '
author: Akshat Sharma
tags:
  - iosxr
  - cisco
  - devnet
  - programmability
  - clus2019
excerpt: >-
  Combining IOS-XR Telemetry and Service layer APIs to solve a remediation use
  case
---
{% include toc %}

## To Know More

There are several tools in the industry that allow you to play around with YANG models on IOS-XR and other Network OS stacks.

In this workshop we will look at a few of them:

* The Ansible `netconf_config` module to configure a BGP session on the routers. <https://docs.ansible.com/ansible/2.4/netconf_config_module.html>  

* Yang Development Kit (YDK) to configure Telemetry and interfaces on the routers: <http://ydk.io>
* Your own python Telemetry client which we will use to extract python data coming from YANG paths set up by YDK:
<https://learninglabs.cisco.com/tracks/iosxr-programmability/iosxr-streaming-telemetry/03-iosxr-02-telemetry-python/step/1>. Also, a lot more on Telemetry here: <https://xrdocs.io/telemetry/>

     
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

     
     
## Keep running the Telemetry client

Before we begin Step 3 of this workshop where we combine IOS-XR Telemetry, Ansible, Open/R and Service-Layer APIs in XR, make sure you have run through Steps 1 and 2 already so that you have the Telemetry client already listening to BGP Session updates from router r1.  

To recap, at the end of Step 2, you must have one terminal window connected to your devbox where you have a client receiving data from router r1's BGP Session (Currently in idle state). If not, please start it again as shown below. 



```
admin@devbox:python$ 
admin@devbox:python$ export SERVER_IP=10.10.20.170
admin@devbox:python$ export SERVER_PORT=57021
admin@devbox:python$ 
admin@devbox:python$ ./telemetry_client_json.py 
Using GRPC Server IP(10.10.20.170) Port(57021)
{
   "encoding_path": "Cisco-IOS-XR-ipv4-bgp-oper:bgp/instances/instance/instance-active/default-vrf/sessions/session", 
   "subscription_id_str": "1", 
   "collection_start_time": 1548827148102, 
   "msg_timestamp": 1548827148108, 
   "collection_end_time": 1548827148108, 
   "node_id_str": "r1", 
   "data_json": [
      {
         "keys": {
            "neighbor-address": "60.1.1.1", 
            "instance-name": "default"
         }, 
         "timestamp": 1548827148107, 
         "content": {
            "messages-queued-in": 0, 
            "is-local-address-configured": false, 
            "local-as": 65000, 
            "nsr-state": "bgp-nbr-nsr-st-none", 
            "description": "", 
            "connection-remote-address": {
               "ipv4-address": "60.1.1.1", 
               "afi": "ipv4"
            }, 
            "messages-queued-out": 0, 
            "connection-state": "bgp-st-idle", 
            "speaker-id": 0, 
            "vrf-name": "default", 
            "remote-as": 65000, 
            "postit-pending": false, 
            "nsr-enabled": true, 
            "connection-local-address": {
               "ipv4-address": "0.0.0.0", 
               "afi": "ipv4"
            }
         }
      }
   ], 
   "collection_id": 16
}


```


## Open/R Deployment

With the Telemetry client running, we can now progress to the next set of steps where we deploy Open/R as an application on both the routers.  

The basic deployment of Open/R we intend to achieve is represented below:

![openr-b2b.png]({{site.baseurl}}/images/openr-b2b.png)



### View the Open/R config files 

We will use Ansible to deploy Open/R as docker instances to the routers r1 and r2. In addition to the docker instances, Open/R requires some configuration files to be present (much like an ISIS or BGP config). We will utilize Ansible to push these files to the routers as well.


To view the relevant config files, **open a new shell** on the devbox with the telemetry client running in the earlier shell. 

Drop into the `ansible/openr` folder in the original `iosxr-devnet-clus2019` git repository you cloned earlier: 


```
admin@devbox:~$ cd iosxr-devnet-clus2019/
admin@devbox:iosxr-devnet-clus2019$ ls
ansible  README.md  ydk  ztp_hooks
admin@devbox:iosxr-devnet-clus2019$ 
admin@devbox:iosxr-devnet-clus2019$ 
admin@devbox:iosxr-devnet-clus2019$ cd ansible/openr/
admin@devbox:openr$ 
admin@devbox:openr$ ls
hosts_r1  increment_ipv4_prefix1.py  launch_openr_r1.sh  run_openr_r1.sh
hosts_r2  increment_ipv4_prefix2.py  launch_openr_r2.sh  run_openr_r2.sh
admin@devbox:openr$ 


```

1. The `hosts_rtr*` files are used to set up `/etc/hosts` inside the docker container that we launch using Ansible. This setting is important for Open/R to work.

2. The `launch_openr_*.sh` scripts contain the actual docker run command which will launch the container and mount relevant config files into it.

3. The `run_openr_*.sh` script contains the configuration knobs for Open/R to know what routes to advertise to other Open/R instances on the network, the gRPC port for Service-Layer API to use while running locally on the router, the interfaces of XR that it should send its hellos out on, etc.

4. The `increment_ipv4_prefix*.py` script is an optional script we use to push a large set of routes into the configuration file for Open/R to advertise. 





### Ansible Playbook to deploy Open/R


The Ansible playbook we intend to use can be found in the `ansible` directory  of the `iosxr-devnet-clus2019` git repository:  


```
admin@devbox:~$ cd iosxr-devnet-clus2019/ansible/
admin@devbox:ansible$ ls
ansible_hosts          configure_bgp_netconf.yml  openr              setup_dependencies.yml       yang_config
ansible_netconf.yml    docker_bringup.yml         openr_setup.yml    setup_insecure_registry.yml
combined_playbook.yml  execute_python_ztp.yml     set_ipv6_route.sh  xml
admin@devbox:ansible$ 
admin@devbox:ansible$ 
admin@devbox:ansible$ cat openr_setup.yml 
---
- import_playbook: setup_insecure_registry.yml 
- import_playbook: docker_bringup.yml 
admin@devbox:ansible$ 
```
This is also a combined playbook that combines the following playbooks:

1. Playbook to setup a local docker registry on the devbox:
   We set up a private insecure registry on the devbox and pull the open/R docker image from dockerhub, ready to be downloaded by the routers:


```
admin@devbox:ansible$ 
admin@devbox:ansible$ cat setup_insecure_registry.yml 
---
- hosts: devbox-host 
  gather_facts: no
  become: yes

  tasks:

  - name: Install the docker module for Ansible 
    pip:
      name:  docker
      state: present

  - name: Create mount volume for registry
    file:
      path: /tmp/registry
      state: directory

  - name: Deploy the local registry server
    docker_container:
        name: registry
        image: registry:2
        volumes:  /tmp/registry:/var/lib/registry
        ports:
        - "5000:5000"
        restart_policy: always
        state: started
        force_kill: yes
        interactive: yes
        restart: yes

  - name: Tag and push openr image to local registry
    docker_image:
      name: akshshar/openr-xr 
      repository: localhost:5000/openr-xr
      tag: latest
      push: yes
admin@devbox:ansible$ 
```

2. Docker bringup playbook that runs against each router to fetch the docker image from the registry we set up with the first playbook above, fetches the required config files, cleans up any running containers and then launches the open/R containers on each router:


```
admin@devbox:ansible$ cat docker_bringup.yml 
---
- hosts: routers_shell
  gather_facts: no
  become: yes

  vars:
    connect_vars:
       host: "{{ ansible_host }}"
       username: "{{ ansible_user }}"
       password: "{{ ansible_ssh_pass }}"
    up: "sudo -i /misc/app_host/launch_openr_{{ inventory_hostname }}.sh"
    down: "sudo -i docker rm -f openr"

  tasks:


  - name: Copy modified docker sysconfig file to rtr
    copy:
      src: "{{ docker_sysconfig_file }}"
      dest: "/misc/app_host/etc/sysconfig/docker"
      owner: root
      group: root
      mode: a+x

  - name: Copy over modified mounted docker sysconfig file to host docker sysconfig
    shell: source /pkg/bin/ztp_helper.sh && echo -ne "run ssh 10.0.2.16 cp /misc/app_host/etc/sysconfig/docker /etc/sysconfig/docker\n" | xrcmd "admin"

  - name: Restart the docker daemon
    shell: source /pkg/bin/ztp_helper.sh && echo -ne "run ssh 10.0.2.16 service docker restart\n" | xrcmd "admin"

  # Pause for 5 minutes to build app cache.
  - name: Pause the playbook for about 30 seconds as docker daemon restarts due to sysconfig change
    pause:
      seconds: 30


  - name: Copy run_openr script to rtr
    copy:
      src: "{{ run_openr_script }}"
      dest: "/misc/app_host/"
      owner: root 
      group: root 
      mode: a+x 

  - name: Copy hosts_r file to rtr
    copy:
      src: "{{ hosts_r }}"
      dest: "/misc/app_host/"
      owner: root 
      group: root 
      mode: a+x 

  - name: Copy launch_openr script to rtr
    copy:
      src: "{{ launch_openr_script }}"
      dest: "/misc/app_host/"
      owner: root 
      group: root 
      mode: a+x 

  - name: Copy increment_ipv4 script to rtr
    copy:
      src: "{{ increment_ipv4_prefix }}"
      dest: "/misc/app_host/"
      owner: root 
      group: root 
      mode: a+x 

  - name: Copy cron file to rtr (CSCvh76067)
    copy:
      src: "{{ cron_file }}"
      dest: "/misc/app_host/ipv6_fe80_route_append.sh"
      owner: root
      group: root
      mode: a+x

  - name: Set up Cronjob (CSCvh76067)
    cron:
      name: set up ipv6 fe80::/64 routes 
      weekday: "*"
      minute: "*"
      hour: "*"
      user: root
      job: "/misc/app_host/ipv6_fe80_route_append.sh"
      cron_file: ansible_ipv6_fe80_route_append


  - name: Check docker container is running
    shell: sudo -i docker inspect --format={{ '{{.State.Running}}' }}  openr
    args:
      executable: /bin/bash
    register: status
    ignore_errors: yes
  - debug: var=output.stdout_lines

 
  - name: Clean up docker container if running 
    shell: "{{ down }}"
    args:
      executable: /bin/bash
    register: output
    when: status.stdout == "true"
  - debug: var=output.stdout_lines


  - name: Bring up the docker container 
    shell: "{{ up }}"
    args:
      executable: /bin/bash
    register: output
    ignore_errors: yes
  - debug: var=output.stdout_lines  
admin@devbox:ansible$ 
```


As can be seen above, the first 4 tasks of the playbook copy over the relevant config files and scripts for Open/R into `/misc/app_host` directory on the routers. This particular directory is mounted into the docker containers when we launch the container to make the config files available to Open/R inside the docker container. 

The 5th task is only required for this particular version of the IOS-XR software (6.4.1) we are using in the lab due to the bug: `CSCvh76067` which prevents `fe80::/64` routes from being installed for all interfaces in the kernel. This is fixed in subsequent releases of IOS-XR.
We transport a cron job onto the router to fix this issue for this lab.

The next 3 tasks are used to check if Open/R is already running, clean up if it is and then launch the docker container using the `/misc/app_host/launch_openr.sh` we transfered using task 3. 




### Run the Ansible Playbook

Run the playbook as shown below: 


```

admin@devbox:ansible$ ansible-playbook -i ansible_hosts docker_bringup.yml 

PLAY [routers_shell] ******************************************************************************************************************

TASK [Copy run_openr script to rtr] ***************************************************************************************************
ok: [r1]
ok: [r2]

TASK [Copy hosts_r file to rtr] *******************************************************************************************************
ok: [r1]
ok: [r2]

TASK [Copy launch_openr script to rtr] ************************************************************************************************
ok: [r2]
ok: [r1]

TASK [Copy increment_ipv4 script to rtr] **********************************************************************************************
ok: [r1]
ok: [r2]

TASK [Copy cron file to rtr (CSCvh76067)] *********************************************************************************************
ok: [r1]
ok: [r2]

TASK [Set up Cronjob (CSCvh76067)] ****************************************************************************************************
ok: [r1]
ok: [r2]

TASK [Check docker container is running] **********************************************************************************************
 [WARNING]: Consider using 'become', 'become_method', and 'become_user' rather than running sudo

changed: [r1]
changed: [r2]

TASK [debug] **************************************************************************************************************************
ok: [r2] => {
    "output.stdout_lines": "VARIABLE IS NOT DEFINED!"
}
ok: [r1] => {
    "output.stdout_lines": "VARIABLE IS NOT DEFINED!"
}

TASK [Clean up docker container if running] *******************************************************************************************
changed: [r2]
changed: [r1]

TASK [debug] **************************************************************************************************************************
ok: [r2] => {
    "output.stdout_lines": [
        "openr"
    ]
}
ok: [r1] => {
    "output.stdout_lines": [
        "openr"
    ]
}

TASK [Bring up the docker container] **************************************************************************************************
changed: [r1]
changed: [r2]

TASK [debug] **************************************************************************************************************************
ok: [r1] => {
    "output.stdout_lines": [
        "4737c257d5fabf230b06aacd019bcc51e6af2e476befdf7774cd67766e5d1f06"
    ]
}
ok: [r2] => {
    "output.stdout_lines": [
        "10ea9e574a27b6132b139d233b52675ba3e8323fbc45c35c36faff71e3f55648"
    ]
}

PLAY RECAP ****************************************************************************************************************************
r1                         : ok=12   changed=3    unreachable=0    failed=0   
r2                         : ok=12   changed=3    unreachable=0    failed=0   

admin@devbox:ansible$ 
admin@devbox:ansible$ 


```


Perfect! Let's check if the Open/R instances are now running on the routers properly:


Log into router r1 and drop into "bash":

```
RP/0/RP0/CPU0:r1#bash
Wed Jan 30 12:41:59.867 UTC


[r1:~]$ 
[r1:~]$ 
```

Issue a `docker ps` command to check that the container is running: 

```
[r1:~]$ 
[r1:~]$ docker ps
CONTAINER ID        IMAGE                          COMMAND                  CREATED             STATUS              PORTS               NAMES
a06e2dbefbff        192.168.122.11:5000/openr-xr   "/bin/bash -c '/root/"   2 minutes ago       Up 2 minutes                            openr
[r1:~]$
```


Enter the docker container shell and exec into the `global-vrf` network namespace where open/R as a process is launched:



```
[r1:~]$ 
[r1:~]$ docker exec -it openr bash
root@r1:/# 
root@r1:/# 
root@r1:/# ip netns exec global-vrf bash
root@r1:/# 


```

Issue the Open/R CLI commands (breeze commands) to check that the adjacency is properly established with router r2:

```

root@r1:/# 
root@r1:/# breeze kvstore adj


> r1's adjacencies, version: 30, Node Label: 1, Overloaded?: False
Neighbor    Local Interface    Remote Interface      Metric    Weight    Adj Label  NextHop-v4    NextHop-v6               Uptime
r2          Gi0_0_0_0          Gi0_0_0_0                  7         1        50009  10.1.1.20     fe80::5054:ff:fe93:8ab0  1m53s
r2          Gi0_0_0_1          Gi0_0_0_1                  7         1        50010  11.1.1.20     fe80::5054:ff:fe93:8ab1  1m53s
r2          Gi0_0_0_2          Gi0_0_0_2                  6         1        50011  12.1.1.20     fe80::5054:ff:fe93:8ab2  1m53s


root@r1:/# 
root@r1:/# 

```

Check the number of routes learnt by Open/R in its FIB:

```
root@r1:/# breeze fib counters

== r1's Fib counters  ==

fibagent.num_of_routes : 1004

root@r1:/#


```


Now exit back out from the container shell and from XR bash shell and issue a "show route" in the XR CLI of router r1:


```
root@r1:/# 
root@r1:/# exit
exit
root@r1:/# 
root@r1:/# exit
exit
[r1:~]$ 
[r1:~]$ 
[r1:~]$ exit
logout

RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#show route
route  router  
RP/0/RP0/CPU0:r1#show route
Wed Jan 30 12:47:08.968 UTC

Codes: C - connected, S - static, R - RIP, B - BGP, (>) - Diversion path
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - ISIS, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, su - IS-IS summary null, * - candidate default
       U - per-user static route, o - ODR, L - local, G  - DAGR, l - LISP
       A - access/subscriber, a - Application route
       M - mobile route, r - RPL, t - Traffic Engineering, (!) - FRR Backup path

Gateway of last resort is 192.168.122.1 to network 0.0.0.0

S*   0.0.0.0/0 [1/0] via 192.168.122.1, 01:15:05
C    10.1.1.0/24 is directly connected, 01:07:20, GigabitEthernet0/0/0/0
L    10.1.1.10/32 is directly connected, 01:07:20, GigabitEthernet0/0/0/0
C    11.1.1.0/24 is directly connected, 01:07:20, GigabitEthernet0/0/0/1
L    11.1.1.10/32 is directly connected, 01:07:20, GigabitEthernet0/0/0/1
C    12.1.1.0/24 is directly connected, 01:07:20, GigabitEthernet0/0/0/2
L    12.1.1.10/32 is directly connected, 01:07:20, GigabitEthernet0/0/0/2
L    50.1.1.1/32 is directly connected, 01:07:20, Loopback0
a    60.1.1.1/32 [99/0] via 11.1.1.20, 00:00:04, GigabitEthernet0/0/0/1
a    120.1.1.0/24 [99/0] via 11.1.1.20, 00:00:03, GigabitEthernet0/0/0/1
a    120.1.2.0/24 [99/0] via 11.1.1.20, 00:00:03, GigabitEthernet0/0/0/1
a    120.1.3.0/24 [99/0] via 11.1.1.20, 00:00:03, GigabitEthernet0/0/0/1
a    120.1.4.0/24 [99/0] via 11.1.1.20, 00:00:04, GigabitEthernet0/0/0/1
a    120.1.5.0/24 [99/0] via 11.1.1.20, 00:00:03, GigabitEthernet0/0/0/1
a    120.1.6.0/24 [99/0] via 11.1.1.20, 00:00:03, GigabitEthernet0/0/0/1
a    120.1.7.0/24 [99/0] via 11.1.1.20, 00:00:04, GigabitEthernet0/0/0/1
a    120.1.8.0/24 [99/0] via 11.1.1.20, 00:00:03, GigabitEthernet0/0/0/1
a    120.1.9.0/24 [99/0] via 11.1.1.20, 00:00:04, GigabitEthernet0/0/0/1
a    120.1.10.0/24 [99/0] via 11.1.1.20, 00:00:04, GigabitEthernet0/0/0/1
a    120.1.11.0/24 [99/0] via 11.1.1.20, 00:00:03, GigabitEthernet0/0/0/1
a    120.1.12.0/24 [99/0] via 11.1.1.20, 00:00:03, GigabitEthernet0/0/0/1
a    120.1.13.0/24 [99/0] via 11.1.1.20, 00:00:04, GigabitEthernet0/0/0/1
a    120.1.14.0/24 [99/0] via 11.1.1.20, 00:00:04, GigabitEthernet0/0/0/1
 --More-- 


```

Awesome! Routes are learnt by Open/R from its neighbor running on r2 and these routes are programmed into Router r1's RIB using the Service-Layer API. These routes show up as application routes (`a` routes in the `show route` output above).


## Establishing the BGP session


Now that Open/R has learnt the routes from router r2, particularly the loopback0 route (60.1.1.1) from r2, the BGP session between the two routers should now be established:

```
RP/0/RP0/CPU0:r1#show  bgp sessions 
Wed Jan 30 12:49:57.225 UTC

Neighbor        VRF                   Spk    AS   InQ  OutQ  NBRState     NSRState
60.1.1.1        default                 0 65000     0     0  Established  None
RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#

```


### Check the Output of the Telemetry client 


With the BGP session established, you should now see the following change in the output of the telemetry client we were running initially


```
{
   "encoding_path": "Cisco-IOS-XR-ipv4-bgp-oper:bgp/instances/instance/instance-active/default-vrf/sessions/session", 
   "subscription_id_str": "1", 
   "collection_start_time": 1548852684404, 
   "msg_timestamp": 1548852684408, 
   "collection_end_time": 1548852684408, 
   "node_id_str": "r1", 
   "data_json": [
      {
         "keys": {
            "neighbor-address": "60.1.1.1", 
            "instance-name": "default"
         }, 
         "timestamp": 1548852684407, 
         "content": {
            "messages-queued-in": 0, 
            "is-local-address-configured": false, 
            "local-as": 65000, 
            "nsr-state": "bgp-nbr-nsr-st-none", 
            "description": "", 
            "connection-remote-address": {
               "ipv4-address": "60.1.1.1", 
               "afi": "ipv4"
            }, 
            "messages-queued-out": 0, 
            "connection-state": "bgp-st-estab", 
            "speaker-id": 0, 
            "vrf-name": "default", 
            "remote-as": 65000, 
            "postit-pending": false, 
            "nsr-enabled": true, 
            "connection-local-address": {
               "ipv4-address": "50.1.1.1", 
               "afi": "ipv4"
            }
         }
      }
   ], 
   "collection_id": 33
}


```



The `"connection-state": "bgp-st-estab"` in the output above has changed from `"connection-state": "bgp-st-idle"`. Open/R acts as an IGP, learning the loopback routes used by BGP to establish the required session!
{: .notice--success}



## Conclusion

To wrap up, this is what we achieved as part of this workshop:

* Used ZTP CLI hooks in bash to configure grpc, interfaces and loopbacks on each router
* Scaled up the setup by switching to ansible to execute the python ZTP CLI script to configure a new user and configure routes required to allow the docker daemon to download an Open/R image subsequently
* Used Ansible to with `netconf_config` module to send XML encoded BGP YANG model snippet to configure I-BGP on both the routers over netconf.
*  Used YDK to configure Telemetry on Router r1 using the OpenConfig Telemetry Yang Model.
*  Started a simple Telemetry python client that is able to dial-in and receive telemetry data about the BGP session in 15 second intervals
*  Used Ansible to setup and launch the Open/R docker containers on each router
*  Utilized the Service-Layer API intergration of Open/R to program routes into the IOS-XR RIB and act as an IGP
*  With the help of the routes learned by Open/R, the BGP session was able to come up and get established and the same was reflected in the telemetry data received by the telemetry client.
