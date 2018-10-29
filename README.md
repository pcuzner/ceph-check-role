# ceph_check_role  
This project provides a custom ansible module. It's goal is to validate a candidate Ceph host, against the intended Ceph roles and where problems are found pass them back, so they can be consumed by other ansible tasks or systems that run playbooks programmitically.  

## Requirements
- python 2.7 or python 3.x
- ansible 2.6 or above

## Tested Against
- RHEL 7.4 : ansible 2.6.5, python 2.7.5
- Fedora 28: ansible 2.7.0, python 2.7.15

## Custom Module Description
The module takes the following parameters;  

| Name | Description | Required | Default |
|------|-------------|----------|---------|
| mode | describes the usage of the cluster, either prod or dev | No | prod |
| deployment | describes the type of deployment, either rpm or container | No | rpm |
| roles | A comma separated string that describes the intended Ceph roles that the host should support (mons, osds, rgws, iscsigws, mdss) | Yes | NONE |

### Invocation Example
```
  tasks:
    - name: check host configuration against desired ceph role(s)
      ceph_check_role:
        role: "{{ inventory[inventory_hostname] }}"
        mode: prod
        deployment: rpm
      register: result
```  
*An example playbook is provided called ```checkrole.yml``` which illustrates the format of the inventory variable used in the above example.* 

## Validation Logic
The basis of the checks is the host configuration data that ansible provides with it's "gather_facts" process. These 'facts' are gathered by the module itself using the same collectors that Ansible's ```setup``` module uses. The host facts are analysed against the required roles to determine whether host is capable of supporting the role, or combination of roles. The analysis uses various factors including; cpu, ram, disks and network.  

All validity logic is held within a ```Checker``` class. This class takes as input the summary data from ansible_facts, and executes all methods prefixed by "_check". So to add more checks, you just need to add another _check method!  

Here's a breakdown of the checks performed;  
- hosts with an osd role, **must** have free disks.
- rgw roles warn if the network is not based on 10g
- calculating cpu and ram for osd hosts factors in the osd drive count. If cpu/ram is low, a warning is issued
- for osd hosts, the number of disks is compared to NIC bandwidth. If the network bandwidth is low, a warning is issued
- each role has a predefined cpu profile, so these are summed and compared to the host. Shortages result in warnings
- each role has a predefined ram profile, so these are summed and compared to the host. Shortages result in warnings
- role collocation is checked. In rpm mode only osd and rgw roles are flagged as valid. For a container deployment, no collocation restrictions are enforced
- overall status is returned to the caller as OK or NOTOK, together with specific error messages for diagnostics
- in prod mode, 'warnings' become 'errors' which result in an overall NOTOK status
- for monitor hosts the freespace under /var/lib is checked  

## Example Output
Here's an example of the kind of output you can expect. You can see the result of the checks in the ```status``` and ```status_msgs``` variables.  
<pre>
ok: [eric] => {
    "result": {
        "changed": false, 
        "data": {
            "deployment_type": "rpm", 
            "mode": "prod", 
            "role": "osds,rgws,mons", 
            <b>"status": "NOTOK", </b>
            <b>"status_msgs": [
                "critical:OSD role without any free disks", 
                "error:too many roles for RPM deployment mode", 
                "warning:network bandwidth low for rgw role"
            ], </b>
            "summary_facts": {
                "cpu_core_count": 8, 
                "cpu_type": [
                    "AMD FX(tm)-8320 Eight-Core Processor"
                ], 
                "hdd": {}, 
                "hdd_count": 0, 
                "network": {
                    "subnet_details": {
                        "10.90.90.0/24": {
                            "count": 2, 
                            "desc": "10.90.90.0/24 (2x1g)", 
                            "devices": [
                                "bond0"
                            ], 
                            "speed": 2000
                        }, 
                        "192.168.1.0/24": {
                            "count": 1, 
                            "desc": "192.168.1.0/24 (1x1g)", 
                            "devices": [
                                "enp5s0"
                            ], 
                            "speed": 1000
                        }, 
                        "192.168.100.0/24": {
                            "count": 1, 
                            "desc": "192.168.100.0/24", 
                            "devices": [
                                "virbr0_nic"
                            ], 
                            "speed": 0
                        }
                    }, 
                    "subnets": [
                        "10.90.90.0/24", 
                        "192.168.1.0/24", 
                        "192.168.100.0/24"
                    ]
                }, 
                "ram_mb": 32132, 
                "ssd": {}, 
                "ssd_count": 0
            }
        }, 
        "failed": false
    }
}

</pre>

