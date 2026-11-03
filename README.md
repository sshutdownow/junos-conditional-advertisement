# junos-pyez-event-script-2RE
JunOS event script on python that works on 2RE router 
In my AS I have 2 BGP routers (ASBR) thay are connected to the same IX. When some kind of disaster occurs with one of the channels to IX, the other channel to IX is overloaded. Because IX is the single AS, traffic to/from it do not distributed via other Internet channles. To overcome this situation I have developed solutions that is relay on [Junos Event Script feature](https://www.juniper.net/documentation/us/en/software/junos/automation-scripting/topics/concept/junos-script-automation-event-script-overview.html):
* I have created BGP seesion between my BGP border routers through both channels to IX and monitor its state,
* when there is some kind of disaster with one of the channels to IX, BGP session is down and than script **bgp_down** is activated, it adds BGP communities that says IX's route servers do not annonce our AS to some ASes and traffic from these ASes shall move from IX to other Internet channels that my AS have.
* When BGP session is UP (that means both channles to IX are UP) **bgp_up** script is activated, it removes communities that were set by **bgp_down** script.

Installation
------------
1. Download scripts.
2. Modify it to yours needs and.
3. Copy scripts to Juniper devices.
5. Modify configuration on Juniper devices to run scripts on events.

Official documentation how to [store and enable Junos Automation Scripts](https://www.juniper.net/documentation/us/en/software/junos/automation-scripting/topics/task/junos-script-automation-script-storing-enabling.html) 
I do it in this way:
```
ip@mx-1> start shell user root
Password:
root@mmx-1:/var/home/ip # cd /var/db/scripts/event
root@mx-1:/var/db/scripts/event # vi bgp_up.py
ip@mx-1# commit synchronize scripts
re0:
configuration check succeeds
re1:
commit complete
re0:
commit complete
{master}[edit]
ip@mx-1# exit
Exiting configuration mode
ip@mx-1> request system scripts event-scripts reload
```
### Requirements

This script works on Juniper's routing engune(RE) and tested to work on Junos version 21.4R3-S4.9 limited.

# JunOS Conditional Advertisement
Another approach is based on Junos [Conditional Advertisement](https://networkingdocs.hpe.com/documentation/us/en/software/junos/routing-policy/bgp/topics/concept/conditional-prefix-installing-overview.html) feature. Everything is done within the Juniper configuration, so you do not need to drop into the shell or copy Python scripts onto the router.

Common part:
```
set policy-options policy-statement NO_ROUTES term default then reject
```
On ASBR #1:
```
set policy-options policy-statement EXP-COND-ROUTE term ISP-iface from protocol direct
set policy-options policy-statement EXP-COND-ROUTE term ISP-iface from route-filter {{ peer1.ip_network }} exact
set policy-options policy-statement EXP-COND-ROUTE term ISP-iface then accept

set protocols bgp group BGPMON type internal
edit protocols bgp group BGPMON neighbor {{ peer2.local_ip }}
set description "iBGP to peer #2 via ISP"
set local-address {{ peer1.local_ip }}
set import NO_ROUTES
set authentication-key {{ bgp_md5_key }}
set export EXP-COND-ROUTE
set export NO_ROUTES
top
```
On ASBR #2:
```
edit policy-options policy-statement IMP-COND-ROUTE
set term isp-up from protocol bgp
set term isp-up from route-filter {{ peer1.ip_network }} exact
set term isp-up then accept
top

edit protocols bgp group BGPMON neighbor {{ peer1.local_ip }}
set description "iBGP to peer #1 via ISP"
set local-address {{ peer2.local_ip }}
set import IMP-COND-ROUTE
set import NO_ROUTES
set authentication-key {{ bgp_md5_key }}
set export NO_ROUTES
top

edit protocols bgp group UPLINKS4 neighbor {{ peer2.ip }}
set description "ISP UPLINK #2"
set local-address {{ peer2.local_ip }}
set import pol-reject-martians4
set import IMP-UPSTREAM-FIRST
set import IMP-RPKI-VALIDATION
set import IMP-ISP
set authentication-key {{ bgp_md5_key }}
set export EXP-FLOWSPEC
set export EXP-ISP
set export EXP-UPSTREAM-FINAL
set peer-as {{ peer2.asn }}
top
edit policy-options condition CND-ISP-UP
set if-route-exists address-family inet {{ peer1.ip_network }}
set if-route-exists address-family inet table inet.0
top
edit policy-options policy-statement EXP-ISP
/* ... */
set term prepend-cond from community community-mynet
set term prepend-cond from condition CND-ISP-UP
set term prepend-cond then metric 2147483647
set term prepend-cond then community add COMMUNITY_ISP_CONTROL
set term prepend-cond then as-path-prepend 31214
set term prepend-cond then next policy

set term isp from community community-mynet
set term isp then metric 0
set term isp then community add COMMUNITY_ISP_CONTROL
set term isp then next policy
/* ... */
set term implicit-deny then reject
top
```

How to check (on ASBR #2), if condition is active:
```
show policy conditions CND-ISP-UP detail
Condition CND-ISP-UP (static), event: Existence of a route in a specific routing table
Dependent routes:
 XX.YY.64.0/20, generation 2400367

Condition state:
Checked: True
Route state : Active
```

### Copyright

  Copyright (c) 2024-2026 Igor Popov

License
-------
   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.

### Authors

  Igor Popov
  (ipopovi |at| gmail |dot| com)
