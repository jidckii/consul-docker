# Install consul agent in docker container  
[![Build Status](https://travis-ci.org/jidckii/consul-docker.svg?branch=master)](https://travis-ci.org/jidckii/consul-docker)  
#### Requirements  

- Docker Engine  
- sudo user

---
The entire configuration for the agent is described as the native json within the  
variable `consul_local_config`.  
**By default, this variable is not defined.** (To exclude a de-job with an empty config.)  
Details about all options in the documentation:     
<https://www.consul.io/docs/agent/options.html>  

To help in automatic configuration on many hosts there are auxiliary variables:  
`consul_advertise_addr` - by default is NULL.  
Gets the public ip host at the time deploy.  
An automatic lookup will only occur if the variable is NULL.  
Specifies for the `"advertise_addr"` option.  

`consul_bind_addr` - by default is `"{{ansible_default_ipv4.address}}"`.  
Specifies for the `"bind_addr"` option.

`consul_node_name` - by default is `"{{inventory_hostname}}"`.  
Specifies for the `"node_name"`.

`consul_retry_join` - by default is NULL.  
Helps with automatic generation of the list of hosts from the inventory for connections to the cluster.  
Specifies for the `"retry_join"`.  
Example you create group in inventory which contains a list of agents in the server role:  
```
[consul-servers]  
192.168.56.101  
192.168.56.102  
192.168.56.103  
```
Next, the variable should get this group as a json array, example:  
`consul_retry_join: '{% for host in groups["consul-servers"] %}"{{host}}"{% if not loop.last %}, {% endif %} {% endfor %}'`  
or :
```
{% for host in groups["consul-servers"] %}
"{{host}}"
  {% if not loop.last %}
  , 
  {% endif %} 
{% endfor %}
```
Can also use tags in AWS for [Cloud Auto-joining](https://www.consul.io/docs/agent/cloud-auto-join.html).

More about [Magic Variables](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#magic-variables-and-how-to-access-information-about-other-hosts).

Also supported the graceful leave and undeploy agent from the host,  
using the variable `undeploy_consul: "yes"`(default "no").

Variable `redeploy_consul: "yes"`(default "no") correctly deletes and re-installs the agent.  
(helps with debugging)  

#### Example of a playbook for deploying an agent in the role of SERVER:
```
- hosts: consul-servers
  become: yes
  tasks:

    - name: Install consul-server in docker
      tags: ["consul-server"]
      include_role:
        name: consul-docker
      vars:
        consul_bind_addr: "{{ansible_eth1.ipv4.address}}"
        consul_retry_join: '{% for host in groups["consul-servers"] %}"{{host}}"{% if not loop.last %}, {% endif %} {% endfor %}'
        consul_local_config: {
          "advertise_addr": "{{consul_advertise_addr}}",
          "bind_addr": "{{consul_bind_addr}}",
          "bootstrap_expect": 3,
          "client_addr": "0.0.0.0",
          "datacenter": "motorsport",
          "node_name": "{{consul_node_name}}",
          "server": true,
          "rejoin_after_leave": true,
          "ui": true,
          "retry_join": "[{{ consul_retry_join }}]"
        }
```

#### Example of a playbook for deploying an agent in the role of CLIENT:
---
```
- hosts: consul-clients
  become: yes
  tasks:

    - name: Install consul-client in docker
      tags: ["consul-client"]
      include_role:
        name: consul-docker
      vars:
        consul_local_config: {
            "advertise_addr": "{{consul_advertise_addr}}",
            "bind_addr": "{{consul_bind_ip}}",
            "datacenter": "motorsport",
            "node_name": "{{consul_node_name}}",
            "rejoin_after_leave": true,
            "retry_join": [ 
              "192.168.56.101",
              "192.168.56.102",
              "192.168.56.103"
              ]
          }

```

#### Example of registering a service using ansible:
---
*More about the module and dependencies:*     
<https://docs.ansible.com/ansible/latest/modules/consul_module.html?highlight=consul>  

###### Registrations:
```
     - name: Register Service node_exporter in consul
       consul:
         state: present
         service_name: node_exporter
         service_port: 9100
         tags:
           - dev
```
**revove service option**
`state: absent`

#### About registering services using the HTTP API via payload:
---
<https://www.consul.io/api/agent/service.html>  

### Required ports:

*Server RPC is used for communication between Consul clients and servers for internal*     
*request forwarding.*    
EXPOSE 8300

*Serf LAN and WAN (WAN is used only by Consul servers) are used for gossip between*    
*Consul agents. LAN is within the datacenter and WAN is between just the Consul*    
*servers in all datacenters.*    
EXPOSE 8301 8301/udp 8302 8302/udp

*HTTP and DNS (both TCP and UDP) are the primary interfaces that applications*    
*use to interact with Consul.*     
EXPOSE 8500 8600 8600/udp
