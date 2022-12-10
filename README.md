Ansible Role: ha-kubernetes
=========

Ansible role that configures a kubernetes cluster with high availability options using keepalived and HAProxy.


<img src="images/ha-diagram.png" alt="ha-diagram" width="600"/>


This role installs keepalived and HAproxy before forming a highly available kubernetes cluster.

I'm using roles from @geerlingguy to:
- install containerd as the container runtime
- install kubernetes package
- initiate a control plane node (referred to as the 'lead_controller')

The remaining control plane and worker nodes are then joined to the cluster.

Dependencies
------------

Required roles:
- geerlingguy.containerd
- geerlingguy.kubernetes

Can be installed using:
~~~
ansible-galaxy install -r requirements.yml
~~~

Inventory configuration
-----------------------

This role requires that control plane nodes are a part of the "controllers" group and worker nodes are a part of the "workers" group.  


One control plane node is elected as leader, this node will initialise the cluster and all other nodes will join. This should be set using the "lead_controller: {{ inventory_name }}" global variable. The following example inventory file shows the node named "barry" has the lead controller.

~~~
all:
  vars:
    ansible_user: vagrant
    lead_controller: *barry*
  children:
    controllers:
      hosts:
        *barry:*
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

If no lead controller is set, the alpha-numerically lowest controller will be dynamically set as the lead controller. The lead controller has no operational significance to the cluster, it is simply the node on which the cluster is intialized.

Role Variables
--------------
~~~
api_endpoint: "192.168.60.100"
~~~
Virtual IP Address used by VRRP and managed by keepalived. 

~~~
api_port: 8443
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
      - api_endpoint: "10.252.1.1"
      - api_port: "38080"
~~~

Collaboration
-------------
Please feel free to raise issues and submit pull requests. I have been using this role along side Vagrant to spin up dev and test environments that are in line with my production environment. I am working on making this role more generic and accessible for other usecases.

Notes
-----
Tested using:
- Ubuntu 20.04
- CentOS 7.9

License
-------
MIT

Author Information
------------------
Authored by gadgieOps.
