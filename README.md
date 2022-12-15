Ansible Role: ha-kubernetes
=========

Ansible role that configures a kubernetes cluster with high availability options using Kubeadm, Keepalived and HAProxy.


<img src="images/ha-diagram.png" alt="ha-diagram" width="800"/>


This role installs keepalived and HAproxy before forming a highly available kubernetes cluster. The cluster is initiated on a single controle plane node (reffered to as the lead_controller, see Inventory configuration). All remaining nodes are joined to the lead_controller after initiation.

Dependencies
------------

I am using the excellent work of others to achieve 

Required roles:
~~~
- src: geerlingguy.containerd
- src: geerlingguy.kubernetes
- name: ansible-keepalived
  src: https://github.com/evrardjp/ansible-keepalived.git
~~~

Can be installed using:
~~~
ansible-galaxy install -r requirements.yml
~~~

Python packages:
- dnspython (only when using FQDNs for ansible_host, see inventory configuration below)

~~~
pip install dnspython
~~~

lead_controller
---------------

One control plane node is elected as leader, this node will initialise the cluster and all other nodes will join. This should be set explicitly as a global var in the Inventory or assigned as a role var (See Inventory Configuration and Role Variables below). If lead_controller is not set, the role will chose the alphanumerically lowest inventory name to assign as the lead_controller. The lead_controller has no operational significance to the cluster, it is simply the node on which the cluster is intialized.


Inventory configuration
-----------------------

This role requires that control plane nodes are a part of the "controllers" group and worker nodes are a part of the "workers" group.  


The following example inventory file shows the node named "barry" as the lead_controller.

~~~
all:
  vars:
    ansible_user: vagrant
    lead_controller: barry
  children:
    controllers:
      hosts:
        barry:
          ansible_host: 192.168.60.101
          ansible_ssh_private_key_file: .vagrant/machines/control-1/virtualbox/private_key
        robin:
          ansible_host: 192.168.60.102
          ansible_ssh_private_key_file: .vagrant/machines/control-2/virtualbox/private_key 
        maurice:
          ansible_host: 192.168.60.103
          ansible_ssh_private_key_file: .vagrant/machines/control-3/virtualbox/private_key
    workers:
      hosts:
        diana:
          ansible_host: 192.168.60.111
          ansible_ssh_private_key_file: .vagrant/machines/worker-1/virtualbox/private_key
        florence:
          ansible_host: 192.168.60.112
          ansible_ssh_private_key_file: .vagrant/machines/worker-2/virtualbox/private_key
        mary:
          ansible_host: 192.168.60.113
          ansible_ssh_private_key_file: .vagrant/machines/worker-3/virtualbox/private_key
~~~

*Note:* when using FQDNs for ansible_host, dnspython package is required to resolve the IP address. This role explicitly sets a bind interface for VRRP which needs the cluster_interface to be set. 

Role Variables
--------------
~~~
lead_controller: "{{ lead_controller_inventory_hostname }}"
~~~
Set the lead_controller of the cluster. This can also be set in the inventory or not declared at all. It should be set to an inventory hostname. 

~~~
node_name: "{{ ansible_hostname }}"
~~~
Set the node name for each node. By default, it uses the nodes hostname. node_name: "{{ inventory_name }}" will set the node name to the alias name defined in the inventory file.

~~~
vrrp_virtual_ip: "192.168.60.100"
~~~
Virtual IP Address used by VRRP and managed by keepalived. 

~~~
haproxy_listen_port: 8443
~~~
The port that HAProxy listens on

~~~
pod_subnet: 10.252.0.0/16
~~~
The subnet that is used to for the pod network

~~~
cni: calico
~~~
The CNI to use. Can be "calico", "flannel" or "weave"

~~~
taint_controllers: true
~~~
Sets if control plane nodes can host pods. 
- true: no pods can be scheduled to the control plane
- false: pods can be scheduled to the control plane

When transitioning betwen a taint and not taint, and pods that are already scheduled will not be removed. 

~~~
set_user_kubeconfig: true
~~~
Sets the kubeconfig for the ansible_user. The root user always has its kubeconfig set.

~~~
set_host_kubeconfig: false
~~~
When set to true, the kubeconfig file is downloaded to the ansible controller host and placed in the directory set by host_kubeconfig_dest

~~~
host_kubeconfig_dest: ~/.kube/config
~~~
The directory that the kubeconfig is downloaded to when set_host_kuneconfig is true.

Example Playbook
----------------
~~~
- name: Provision Kubernetes Cluster
  hosts: all
  become: true
  gather_facts: true
  roles:
    - role: gadgieOps.ha-kubernetes
      vars:
      - taint_controllers: true
      - set_host_kubeconfig: true
      - vrrp_virtual_ip: "10.252.1.1"
      - haproxy_listen_port: "38080"
~~~

Collaboration
-------------
Please feel free to raise issues and submit pull requests.

Notes
-----
Tested using:
- Ubuntu 20.04
- CentOS 7.9

This method of deploying a non production kubernetes environment is admitidly a heavy process. Various tools such as micro-k8s, kind, k3s may be more appropriate for lighter weight solutions.

License
-------
MIT

Author Information
------------------
Authored by gadgieOps.
