---
published: true
date: '2018-06-12 09:51 -0400'
title: 'Step 2: Playing with IOS-XR YANG Models'
author: Akshat Sharma
tags:
  - iosxr
  - cisco
  - devnet
  - cleur2019
excerpt: >-
  Learn to use Ansible , YDK and Telemetry to solve several provisioning and
  monitoring tasks. We run some clients from written from scratch to help the
  reader grasp these new techniques.
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
><hhttps://iosxr-devnet-ciscolive.github.io/clus2019-workshop/assets/CLUS-19-Akshat-IOS-XR-Programmability.pdf>
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

     


## Ansible netconf_config module

Hop into the devbox and browse to the ansible directory:


```
admin@devbox:~$ cd iosxr-devnet-clus2019/
admin@devbox:iosxr-devnet-clus2019$ ls
ansible  README.md  ydk  ztp_hooks
admin@devbox:iosxr-devnet-clus2019$ 
admin@devbox:iosxr-devnet-clus2019$ cd ansible/
admin@devbox:ansible$ 
admin@devbox:ansible$ ls
ansible_hosts          configure_bgp_netconf.yml  openr              setup_dependencies.yml       yang_config
ansible_netconf.yml    docker_bringup.yml         openr_setup.yml    setup_insecure_registry.yml
combined_playbook.yml  execute_python_ztp.yml     set_ipv6_route.sh  xml
admin@devbox:ansible$ 

```

We will be using the playbook: `ansible_netconf.yml` to configure BGP on routers r1 and r2:

The playbook is dumped below: 

```
admin@devbox:ansible$ 
admin@devbox:ansible$ cat ansible_netconf.yml 
---
- import_playbook: setup_dependencies.yml
- import_playbook: configure_bgp_netconf.yml
admin@devbox:ansible$ 
```  

It can be seen that it is a combined playbook which in turn executes two other playbooks, expanded below:

```
admin@devbox:ansible$ 
admin@devbox:ansible$ 
admin@devbox:ansible$ cat setup_dependencies.yml 
---
- hosts: devbox-host
  gather_facts: no
  become: yes

  tasks:

  - name: Install ncclient 
    pip:
      name:  ncclient
      state: present
admin@devbox:ansible$ 
admin@devbox:ansible$ 
```  


The `setup_dependencies.yml` playbook installs `ncclient` on the devbox host machine using `pip`. This is the dependency required to execute the `netconf_config` module on the devbox. We let ansible handle the dependencies in an automated fashion.


```
admin@devbox:ansible$ 
admin@devbox:ansible$ cat configure_bgp_netconf.yml 
---
- hosts: routers_shell
  connection: local
  gather_facts: no

  tasks:

  - name: Configure BGP on the router
    netconf_config:
      host: "{{ ansible_host }}"
      port: "{{ netconf_port }}"
      username: "{{ ansible_user }}"
      password: "{{ ansible_password }}"
      hostkey_verify: no
      xml: "{{ lookup('file', xml_file) }}"
admin@devbox:ansible$ 
admin@devbox:ansible$ 
```  

The second playbook is `configure_bgp_netconf.yml` which utilizes the `netconf_config` module and accepts tparameters (netconf_ip, netconf_port, username, password and the xml file) to leverage netconf to apply xml encoded YANG modeled configuration on the router.


The xml file used by the above playbook to configure BGP on router r1 is shown below. 
This XML data utilizes the IOS-XR BGP Config YANG model:  `Cisco-IOS-XR-ipv4-bgp-cfg`.


```
admin@devbox:ansible$ cat xml/r1-bgp.xml 
<config>
  <bgp xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-ipv4-bgp-cfg">
   <instance>
    <instance-name>default</instance-name>
    <instance-as>
     <as>0</as>
     <four-byte-as>
      <as>65000</as>
      <bgp-running></bgp-running>
      <default-vrf>
       <global>
        <router-id>50.1.1.1</router-id>
        <global-afs>
         <global-af>
          <af-name>ipv4-unicast</af-name>
          <enable></enable>
         </global-af>
        </global-afs>
       </global>
       <bgp-entity>
        <neighbors>
         <neighbor>
          <neighbor-address>60.1.1.1</neighbor-address>
          <remote-as>
           <as-xx>0</as-xx>
           <as-yy>65000</as-yy>
          </remote-as>
          <update-source-interface>Loopback0</update-source-interface>
          <neighbor-afs>
           <neighbor-af>
            <af-name>ipv4-unicast</af-name>
            <activate></activate>
           </neighbor-af>
          </neighbor-afs>
         </neighbor>
        </neighbors>
       </bgp-entity>
      </default-vrf>
     </four-byte-as>
    </instance-as>
   </instance>
      </bgp>
    </config>
admin@devbox:ansible$ 
```

Similarly, the XML file for router r2:  


```
admin@devbox:ansible$ 
admin@devbox:ansible$ 
admin@devbox:ansible$ cat xml/r2-bgp.xml 
<config>
  <bgp xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-ipv4-bgp-cfg"> 
   <instance> 
    <instance-name>default</instance-name> 
    <instance-as> 
     <as>0</as> 
     <four-byte-as> 
      <as>65000</as> 
      <bgp-running></bgp-running> 
      <default-vrf> 
       <global> 
        <router-id>60.1.1.1</router-id> 
        <global-afs> 
         <global-af> 
          <af-name>ipv4-unicast</af-name> 
          <enable></enable> 
         </global-af> 
        </global-afs> 
       </global> 
       <bgp-entity> 
        <neighbors> 
         <neighbor> 
          <neighbor-address>50.1.1.1</neighbor-address> 
          <remote-as> 
           <as-xx>0</as-xx> 
           <as-yy>65000</as-yy> 
          </remote-as> 
          <update-source-interface>Loopback0</update-source-interface> 
          <neighbor-afs> 
           <neighbor-af> 
            <af-name>ipv4-unicast</af-name> 
            <activate></activate> 
           </neighbor-af> 
          </neighbor-afs> 
         </neighbor> 
        </neighbors> 
       </bgp-entity> 
      </default-vrf> 
     </four-byte-as> 
    </instance-as> 
   </instance> 
  </bgp>
</config>
admin@devbox:ansible$ 
```




### Execute the Ansible playbook to configure BGP

Execute the Ansible playbook to configure BGP on the two routers:



```
admin@devbox:ansible$ ansible-playbook -i ansible_hosts ansible_netconf.yml 

PLAY [devbox-host] *************************************************************************************************************

TASK [Install ncclient] ********************************************************************************************************
changed: [devbox]

PLAY [routers_shell] ***********************************************************************************************************

TASK [Configure BGP on the router] *********************************************************************************************
changed: [r1]
changed: [r2]

PLAY RECAP *********************************************************************************************************************
devbox                     : ok=1    changed=1    unreachable=0    failed=0   
r1                         : ok=1    changed=1    unreachable=0    failed=0   
r2                         : ok=1    changed=1    unreachable=0    failed=0   

admin@devbox:ansible$ 
```

### Check the BGP configuration on the routers

Hop over to router r1 and see that BGP has been configured:  


```
AKSHSHAR-M-33WP:~ akshshar$ ssh -p 2221 admin@10.10.20.170


--------------------------------------------------------------------------
  Router 1 (Cisco IOS XR Sandbox)
--------------------------------------------------------------------------


Password: 


RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#show run router bgp
Wed Jan 30 04:32:27.173 UTC
router bgp 65000
 bgp router-id 50.1.1.1
 address-family ipv4 unicast
 !
 neighbor 60.1.1.1
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
  !
 !
!

RP/0/RP0/CPU0:r1#


```

Similarly, on router r2:  


```

RP/0/RP0/CPU0:r2#show run router bgp
Wed Jan 30 04:29:24.722 UTC
% No such configuration item(s)

RP/0/RP0/CPU0:r2#show run router bgp
Wed Jan 30 04:32:14.159 UTC
router bgp 65000
 bgp router-id 60.1.1.1
 address-family ipv4 unicast
 !
 neighbor 50.1.1.1
  remote-as 65000
  update-source Loopback0
  address-family ipv4 unicast
  !
 !
!

RP/0/RP0/CPU0:r2#


```


## YDK

To know more about YDK, head over to <http://ydk.io>

### YDK python script to configure Model-Driven Telemetry

YDK-py is already installed in the devbox for you. 

Jump into the `ydk` directory in the clone github repository:  


```
admin@devbox:~$ cd ~/iosxr-devnet-clus2019/
admin@devbox:iosxr-devnet-clus2019$ cd ydk/
admin@devbox:ydk$ 
```

>Writing your own python script with YDK to interact with Yang models on IOS-XR and the other Vendor OSes is straightforward and you will find tons of resources in the git repository:
https://github.com/CiscoDevNet/ydk-py-samples with hundreds of examples across different models supported by IOS-XR.
  
The YDK script used for this purpose is shown below:
  
```
admin@devbox:ydk$ cat configure_telemetry_openconfig.py 
#!/usr/bin/env python
#
# Copyright 2016 Cisco Systems, Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

"""
Create configuration for model openconfig-telemetry.

usage: configure_telemetry_openconfig.py [-h] [-v] device

positional arguments:
  device         NETCONF device (ssh://user:password@host:port)

optional arguments:
  -h, --help     show this help message and exit
  -v, --verbose  print debugging messages
"""

from argparse import ArgumentParser
from urlparse import urlparse

from ydk.services import CRUDService
from ydk.providers import NetconfServiceProvider
from ydk.models.openconfig import openconfig_telemetry \
    as oc_telemetry
import logging


def config_telemetry_system(telemetry_system):
    """Add config data to telemetry_system object."""
    #sensor-group
    sensor_group = telemetry_system.sensor_groups.SensorGroup()


    sensor_group.sensor_group_id = "BGPSession"

    sensor_path = sensor_group.sensor_paths.SensorPath()
    sensor_path.path = "Cisco-IOS-XR-ipv4-bgp-oper:bgp/instances/instance/instance-active/default-vrf/sessions"
    sensor_group.sensor_paths.sensor_path.append(sensor_path)
    telemetry_system.sensor_groups.sensor_group.append(sensor_group)

    sensor_path = sensor_group.sensor_paths.SensorPath()
    sensor_path.path = "Cisco-IOS-XR-ipv4-bgp-oper:bgp/instances/instance/instance-active/default-vrf/process-info"
    sensor_group.sensor_paths.sensor_path.append(sensor_path)
    telemetry_system.sensor_groups.sensor_group.append(sensor_group)


    sensor_group = telemetry_system.sensor_groups.SensorGroup()
    sensor_group.sensor_group_id = "IPV6Neighbor"

    sensor_path = sensor_group.sensor_paths.SensorPath()
    sensor_path.path = "Cisco-IOS-XR-ipv6-nd-oper:ipv6-node-discovery/nodes/node/neighbor-interfaces/neighbor-interface/host-addresses/host-address"
    sensor_group.sensor_paths.sensor_path.append(sensor_path)
    telemetry_system.sensor_groups.sensor_group.append(sensor_group)


    #subscription
    subscription = telemetry_system.subscriptions.persistent.Subscription()
    subscription.subscription_id = 1
    sensor_profile = subscription.sensor_profiles.SensorProfile()
    sensor_profile.sensor_group = "BGPSession"
    sensor_profile.config.sensor_group = "BGPSession"
    sensor_profile.config.sample_interval = 15000
    subscription.sensor_profiles.sensor_profile.append(sensor_profile)
    telemetry_system.subscriptions.persistent.subscription.append(subscription)

    subscription = telemetry_system.subscriptions.persistent.Subscription()
    subscription.subscription_id = 2
    sensor_profile = subscription.sensor_profiles.SensorProfile()
    sensor_profile.sensor_group = "IPV6Neighbor"
    sensor_profile.config.sensor_group = "IPV6Neighbor"
    sensor_profile.config.sample_interval = 15000
    subscription.sensor_profiles.sensor_profile.append(sensor_profile)
    telemetry_system.subscriptions.persistent.subscription.append(subscription)

if __name__ == "__main__":
    """Execute main program."""
    parser = ArgumentParser()
    parser.add_argument("-v", "--verbose", help="print debugging messages",
                        action="store_true")
    parser.add_argument("device",
                        help="NETCONF device (ssh://user:password@host:port)")
    args = parser.parse_args()
    device = urlparse(args.device)

    # log debug messages if verbose argument specified
    if args.verbose:
        logger = logging.getLogger("ydk")
        logger.setLevel(logging.INFO)
        handler = logging.StreamHandler()
        formatter = logging.Formatter(("%(asctime)s - %(name)s - "
                                      "%(levelname)s - %(message)s"))
        handler.setFormatter(formatter)
        logger.addHandler(handler)

    # create NETCONF provider
    provider = NetconfServiceProvider(address=device.hostname,
                                      port=device.port,
                                      username=device.username,
                                      password=device.password,
                                      protocol=device.scheme)
    # create CRUD service
    crud = CRUDService()

    telemetry_system = oc_telemetry.TelemetrySystem()  # create object
    config_telemetry_system(telemetry_system)  # add object configuration

    # create configuration on NETCONF device
    crud.create(provider, telemetry_system)

    exit()
# End of script
admin@devbox:ydk$ 


```


### Execute the YDK script:


We intend to configure Model Driven Telemetry on router r1 so that it is ready to stream data associated with BGP Sessions, BGP process and IPv6 neighbor data in the above example.
The Telemetry client we write later will try and receive the BGP session information from the router.  

Execute the YDK script, passing to it the credentials and connection details for Router r1 and its netconf port:

Pass the `-v` option to the script to dump the requests/responses as the script executes.

```
admin@devbox:ydk$ 
admin@devbox:ydk$ ./configure_telemetry_openconfig.py -v ssh://vagrant:vagrant@10.10.20.170:8321
2019-01-29 21:02:48,375 - ydk - INFO - Path where models are to be downloaded: /home/admin/.ydk/10.10.20.170_8321
2019-01-29 21:02:48,386 - ydk - INFO - Connected to 10.10.20.170 on port 8321 using ssh with timeout of -1
2019-01-29 21:02:48,396 - ydk - INFO - Executing CRUD create operation on [openconfig-telemetry:telemetry-system]
2019-01-29 21:02:52,735 - ydk - INFO - =============Generating payload to send to device=============
2019-01-29 21:02:52,735 - ydk - INFO - 
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<edit-config xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <target>
    <candidate/>
  </target>
  <error-option>rollback-on-error</error-option>
  <config><telemetry-system xmlns="http://openconfig.net/yang/telemetry" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" nc:operation="merge">
  <sensor-groups>
    <sensor-group>
      <sensor-group-id>BGPSession</sensor-group-id>
      <sensor-paths>
        <sensor-path>
          <path>Cisco-IOS-XR-ipv4-bgp-oper:bgp/instances/instance/instance-active/default-vrf/sessions</path>
        </sensor-path>
        <sensor-path>
          <path>Cisco-IOS-XR-ipv4-bgp-oper:bgp/instances/instance/instance-active/default-vrf/process-info</path>
        </sensor-path>
      </sensor-paths>
    </sensor-group>
    <sensor-group>
      <sensor-group-id>IPV6Neighbor</sensor-group-id>
      <sensor-paths>
        <sensor-path>
          <path>Cisco-IOS-XR-ipv6-nd-oper:ipv6-node-discovery/nodes/node/neighbor-interfaces/neighbor-interface/host-addresses/host-address</path>
        </sensor-path>
      </sensor-paths>
    </sensor-group>
  </sensor-groups>
  <subscriptions>
    <persistent>
      <subscription>
        <subscription-id>1</subscription-id>
        <sensor-profiles>
          <sensor-profile>
            <sensor-group>BGPSession</sensor-group>
            <config>
              <sensor-group>BGPSession</sensor-group>
              <sample-interval>15000</sample-interval>
            </config>
          </sensor-profile>
        </sensor-profiles>
      </subscription>
      <subscription>
        <subscription-id>2</subscription-id>
        <sensor-profiles>
          <sensor-profile>
            <sensor-group>IPV6Neighbor</sensor-group>
            <config>
              <sensor-group>IPV6Neighbor</sensor-group>
              <sample-interval>15000</sample-interval>
            </config>
          </sensor-profile>
        </sensor-profiles>
      </subscription>
    </persistent>
  </subscriptions>
</telemetry-system>
</config>
</edit-config>
</rpc>
2019-01-29 21:02:52,738 - ydk - INFO - 

2019-01-29 21:02:52,789 - ydk - INFO - =============Reply payload received from device=============
2019-01-29 21:02:52,789 - ydk - INFO - 
<?xml version="1.0"?>
<rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="1">
  <ok/>
</rpc-reply>

2019-01-29 21:02:52,791 - ydk - INFO - 

2019-01-29 21:02:52,791 - ydk - INFO - =============Executing commit=============
2019-01-29 21:02:52,791 - ydk - INFO - 
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <commit/>
</rpc>

2019-01-29 21:02:53,092 - ydk - INFO - =============Reply payload received from device=============
2019-01-29 21:02:53,092 - ydk - INFO - 
<?xml version="1.0"?>
<rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="2">
  <ok/>
</rpc-reply>

2019-01-29 21:02:53,093 - ydk - INFO - 

2019-01-29 21:02:53,094 - ydk - INFO - Operation succeeded
2019-01-29 21:02:53,094 - ydk - INFO - Disconnected from device
admin@devbox:ydk$ 

```


### Check the Telemetry configuration on router r1


```
AKSHSHAR-M-33WP:~ akshshar$ ssh -p 2221 admin@10.10.20.170


--------------------------------------------------------------------------
  Router 1 (Cisco IOS XR Sandbox)
--------------------------------------------------------------------------


Password: 


RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#show  running-config telemetry model-driven 
Wed Jan 30 05:05:03.128 UTC
telemetry model-driven
 sensor-group BGPSession
  sensor-path Cisco-IOS-XR-ipv4-bgp-oper:bgp/instances/instance/instance-active/default-vrf/sessions
  sensor-path Cisco-IOS-XR-ipv4-bgp-oper:bgp/instances/instance/instance-active/default-vrf/process-info
 !
 sensor-group IPV6Neighbor
  sensor-path Cisco-IOS-XR-ipv6-nd-oper:ipv6-node-discovery/nodes/node/neighbor-interfaces/neighbor-interface/host-addresses/host-address
 !
 subscription 1
  sensor-group-id BGPSession sample-interval 15000
 !
 subscription 2
  sensor-group-id IPV6Neighbor sample-interval 15000
 !
!

RP/0/RP0/CPU0:r1#
```



### Verify that the Telemetry sensor-paths are resolved


```
RP/0/RP0/CPU0:r1#show telemetry model-driven subscription 1
Wed Jan 30 05:07:56.424 UTC
Subscription:  1
-------------
  State:       NA
  Sensor groups:
  Id: BGPSession
    Sample Interval:      15000 ms
    Sensor Path:          Cisco-IOS-XR-ipv4-bgp-oper:bgp/instances/instance/instance-active/default-vrf/sessions
    Sensor Path State:    Resolved
    Sensor Path:          Cisco-IOS-XR-ipv4-bgp-oper:bgp/instances/instance/instance-active/default-vrf/process-info
    Sensor Path State:    Resolved

  Collection Groups:
  ------------------
  No active collection groups

RP/0/RP0/CPU0:r1#show telemetry model-driven subscription 2
Wed Jan 30 05:07:59.312 UTC
Subscription:  2
-------------
  State:       NA
  Sensor groups:
  Id: IPV6Neighbor
    Sample Interval:      15000 ms
    Sensor Path:          Cisco-IOS-XR-ipv6-nd-oper:ipv6-node-discovery/nodes/node/neighbor-interfaces/neighbor-interface/host-addresses/host-address
    Sensor Path State:    Resolved

  Collection Groups:
  ------------------
  No active collection groups

RP/0/RP0/CPU0:r1#

```


Awesome! You're now ready to stream telemetry data from the above sensor-paths. Let's proceed to run a custom python Telemetry client.
{: .notice--success}






## Writing your own Python Telemetry client

The Telemetry client is written based on the same concept described in the following learning lab:
<https://learninglabs.cisco.com/tracks/iosxr-programmability/iosxr-streaming-telemetry/03-iosxr-02-telemetry-python/step/1>

We make it slightly simpler and instead of requesting telemetry data in a GPB format, we simply request json and dump it. 

Before we start, install the dependencies required to connect to the router over gRPC to receive the data.  

### Install dependencies for the gRPC telemetry client


```
admin@devbox:~$ sudo pip2 install grpcio-tools==1.7.0 googleapis-common-protos
The directory '/home/admin/.cache/pip/http' or its parent directory is not owned by the current user and the cache has been disabled. Please check the permissions and owner of that directory. If executing pip with sudo, you may want sudo's -H flag.
The directory '/home/admin/.cache/pip' or its parent directory is not owned by the current user and caching wheels has been disabled. check the permissions and owner of that directory. If executing pip with sudo, you may want sudo's -H flag.
Collecting grpcio-tools==1.7.0
  Downloading https://files.pythonhosted.org/packages/0e/c3/d9a9960f12e0bab789da875b1c9a3eb348b51fa3af9544c1edd1f7ef6000/grpcio_tools-1.7.0-cp27-cp27mu-manylinux1_x86_64.whl (21.3MB)
    100% |████████████████████████████████| 21.3MB 50kB/s 
Collecting googleapis-common-protos
  Downloading https://files.pythonhosted.org/packages/61/29/1549f61917eadd11650e42b78b4afcfe9cb467157af4510ab8cb59535f14/googleapis-common-protos-1.5.6.tar.gz
Requirement already satisfied (use --upgrade to upgrade): protobuf>=3.3.0 in /usr/local/lib/python2.7/dist-packages (from grpcio-tools==1.7.0)
Requirement already satisfied (use --upgrade to upgrade): grpcio>=1.7.0 in /usr/local/lib/python2.7/dist-packages (from grpcio-tools==1.7.0)
Requirement already satisfied (use --upgrade to upgrade): setuptools in /usr/lib/python2.7/dist-packages (from protobuf>=3.3.0->grpcio-tools==1.7.0)
Requirement already satisfied (use --upgrade to upgrade): six>=1.9 in /usr/lib/python2.7/dist-packages (from protobuf>=3.3.0->grpcio-tools==1.7.0)
Requirement already satisfied (use --upgrade to upgrade): enum34>=1.0.4 in /usr/lib/python2.7/dist-packages (from grpcio>=1.7.0->grpcio-tools==1.7.0)
Requirement already satisfied (use --upgrade to upgrade): futures>=2.2.0 in /usr/local/lib/python2.7/dist-packages (from grpcio>=1.7.0->grpcio-tools==1.7.0)
Installing collected packages: grpcio-tools, googleapis-common-protos
  Running setup.py install for googleapis-common-protos ... done
Successfully installed googleapis-common-protos-1.5.6 grpcio-tools-1.7.0
You are using pip version 8.1.1, however version 19.0.1 is available.
You should consider upgrading via the 'pip install --upgrade pip' command.
admin@devbox:~$ 
```


### Clone the Telemetry gRPC collectors git repo

We have published a few samples for c++ and python to help developers write their own gRPC based collectors to receive telemetry data from IOS-XR.  

The proto files that we use to create bindings and then write our own clients are published here: <https://github.com/cisco/bigmuddy-network-telemetry-proto/> 

To start, first clone the telemetry-grpc-collectors repo into the devbox home directory:


```
admin@devbox:~$ 
admin@devbox:~$ git clone --recursive https://github.com/ios-xr/telemetry-grpc-collectors
Cloning into 'telemetry-grpc-collectors'...
remote: Enumerating objects: 16, done.
remote: Counting objects: 100% (16/16), done.
remote: Compressing objects: 100% (11/11), done.
remote: Total 137 (delta 4), reused 15 (delta 4), pack-reused 121
Receiving objects: 100% (137/137), 3.67 MiB | 2.79 MiB/s, done.
Resolving deltas: 100% (62/62), done.
Checking connectivity... done.
Submodule 'bigmuddy-network-telemetry-proto' (https://github.com/cisco/bigmuddy-network-telemetry-proto) registered for path 'bigmuddy-network-telemetry-proto'
Cloning into 'bigmuddy-network-telemetry-proto'...
remote: Enumerating objects: 24542, done.
remote: Total 24542 (delta 0), reused 0 (delta 0), pack-reused 24542
Receiving objects: 100% (24542/24542), 6.06 MiB | 3.14 MiB/s, done.
Resolving deltas: 100% (8337/8337), done.
Checking connectivity... done.
Submodule path 'bigmuddy-network-telemetry-proto': checked out '4419cd20fb73f05d059a37fa3e41fe55f02a528f'
admin@devbox:~$ 


```


### Build the Bindings for Model Driven Telemetry gRPC clients

Hop into the `build/python/` directory and generate the required bindings from the proto files:

```
admin@devbox:~$ 
admin@devbox:~$ cd telemetry-grpc-collectors/build/python/
admin@devbox:python$ 
admin@devbox:python$ 
admin@devbox:python$ ./gen-mdt-dialin-bindings.sh 
Generating Python bindings...Done
admin@devbox:python$ 
admin@devbox:python$ tree src/genpy/
src/genpy/
├── __init__.py
├── mdt_grpc_dialin
│   ├── __init__.py
│   ├── mdt_grpc_dialin_pb2_grpc.py
│   └── mdt_grpc_dialin_pb2.py
├── mdt_grpc_dialout
│   ├── __init__.py
│   ├── mdt_grpc_dialout_pb2_grpc.py
│   └── mdt_grpc_dialout_pb2.py
├── telemetry_pb2_grpc.py
└── telemetry_pb2.py

2 directories, 9 files
admin@devbox:python$ 

```


### Running a simple python Telemetry client

Hop into the `clients/python/` directory and dump the `telemetry_client_json.py` script which is written to connect to an IOS-XR router over gRPC, request a json data-stream and dump it to the screen. 

The script is dumped below.

```
admin@devbox:~$ 
admin@devbox:~$ cd ~/telemetry-grpc-collectors/clients/python/
admin@devbox:python$ ls
telemetry_client_json.py  telemetry_client.py
admin@devbox:python$ cat telemetry_client_json.py 
#!/usr/bin/env python

# Standard python libs
import os,sys
sys.path.append("../../build/python/src/genpy")

import ast, pprint 
import pdb
import yaml, json
import telemetry_pb2
from mdt_grpc_dialin import mdt_grpc_dialin_pb2
from mdt_grpc_dialin import mdt_grpc_dialin_pb2_grpc
import grpc
 
#
# Get the GRPC Server IP address and port number
#
def get_server_ip_port():
    # Get GRPC Server's IP from the environment
    if 'SERVER_IP' not in os.environ.keys():
        print("Need to set the SERVER_IP env variable e.g.")
        print("export SERVER_IP='10.30.110.214'")
        os._exit(0)
    
    # Get GRPC Server's Port from the environment
    if 'SERVER_PORT' not in os.environ.keys():
        print("Need to set the SERVER_PORT env variable e.g.")
        print("export SERVER_PORT='57777'")
        os._exit(0)
    
    return (os.environ['SERVER_IP'], int(os.environ['SERVER_PORT']))


#
# Setup the GRPC channel with the server, and issue RPCs
#
if __name__ == '__main__':
    server_ip, server_port = get_server_ip_port()

    print("Using GRPC Server IP(%s) Port(%s)" %(server_ip, server_port))

    # Create the channel for gRPC.
    channel = grpc.insecure_channel(str(server_ip)+":"+str(server_port))

    unmarshal = True

    # Ereate the gRPC stub.
    stub = mdt_grpc_dialin_pb2_grpc.gRPCConfigOperStub(channel)

    metadata = [('username', 'vagrant'), ('password', 'vagrant')]
    Timeout = 3600*24*365 # Seconds

    sub_args = mdt_grpc_dialin_pb2.CreateSubsArgs(ReqId=99, encode=4, subidstr='1')
    stream = stub.CreateSubs(sub_args, timeout=Timeout, metadata=metadata)
    for segment in stream:
        if not unmarshal:
            print(segment)
        else:
            # Go straight for telemetry data
            telemetry_pb = telemetry_pb2.Telemetry()

            encoding_path = 'Cisco-IOS-XR-ipv4-bgp-oper:bgp/instances/'+\
                            'instance/instance-active/default-vrf/sessions/session'


            try:
                # Return in JSON format instead of protobuf.
                if json.loads(segment.data)["encoding_path"] == encoding_path: 
                    print(json.dumps(json.loads(segment.data), indent=3))
            except Exception as e: 
                print("Failed to receive data, error: " +str(e))
    os._exit(0)
admin@devbox:python$ 
```


Finally, open up a new shell and run this script to start receiving data from IOS-XR.
Make sure you export the connection details for the gRPC server running on router r1 before running the script.

```
export SERVER_IP=10.10.20.170
export SERVER_PORT=57021

```
{: .notice--info}


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



Perfect! Now, we're all set up to deploy Open/R as an application on the two routers and test out Application-hosting, packet-io and Service-Layer APIs in XR together.
{: .notice--success}
