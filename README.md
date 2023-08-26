# Ansible roles for installing and configuring Vitess

### What is vitess?
Vitess is a scaleable databases built on top of MySQL. One key feature of vitess is that it supports sharding.
Checkout [Vitess website](https://vitess.io/) for more information

### What this role does?
This playbook sets up VTAdmin, VTGate, VTTablets and Zookeeper with the sample commerce database.

We have used this playbook to setup Vitess multiple times in our staging and production environments successfully.

This repository contains multiple roles:
* vitess-base - Downloads / copies vitess on machines
* zookeeper - Sets up zookeeper which acts as topology server for vitess
* vtAdmin - Sets up vtadmin 
* vttablet - Sets up vttablets according to number of shards. This repo supports setting upto three shards. It can be easily extended to even multiple shards
* vtgate - Sets up vtgate

### Supported Versions

This ansible role has been tested on Vitess v16. The plan is to keep updating this repo as and when newer versions of vitess are released.

### Supported Platforms
Currently, I have only tested this playbook on CentOS 7 and RHEL 7 platforms.

Feel free to raise any [issue](https://github.com/madhur/ansible-vitess/issues) you have encountered if any while setting it up on another platform.

## Quick setup

I have currently not uploaded this role to [Ansible Galaxy](https://galaxy.ansible.com/)

Just clone the repo and copy the folders under `roles` directory into your own `roles` directory of your ansible repo. 

Run below playbooks to setup vitess

### Topology server (Zookeeper)
```yaml
---
- hosts: zookeeper-nodes
  become: True
  roles:
    - role: ansible-role-java
    - role: ansible-zookeeper
```

### vtadmin

```yaml
---
- hosts: vtadmin
  roles:
    - role: common
    - role: vtadmin
```
### vttablets

```yaml
- hosts: vttablet
  roles:
    - role: common
    - role: vttablet
```

### vtgate

```yaml
- hosts: vtgate
  roles:
    - role: common
    - role: vtgate
```

## Roles Configuration & Operations

### Global Variables

The setup requires few variables to function properly which are given below are specified in the inventory file.

See the file `vagrant_1shard/hosts` file for the sample

```properties
vitess_user = vitess 
cell_name = cell1
topo_global_root = /vitess/global
vitess_dir = /usr/local/vitess
VTDATAROOT = /data/vt
vtadmin_ip = 192.168.56.9
vtadmin_grpc_port = 15999
zk_host = 192.168.56.8
```

* `vitess_user`: is the user which will be created under which all vitess processes will run  
* `cell_name`: is the name of cell  
* `topo_global_root`:  is the vitess topo_global_root for vitess. See [Vitess docs](https://vitess.io/docs/16.0/user-guides/configuration-basic/global-topo/)  
* `vitess_dir` : the directory where vitess will be installed
* `VTDATAROOT`: See [vitess docs](https://vitess.io/docs/16.0/user-guides/configuration-basic/planning/) on VTDATAROOT
* `vtadmin_ip`: IP Address where vtadmin will be installed  
* `vtadmin_grpc_port`: grpc port of vtadmin  
* `zk_host`: IP Address of topology server


## vitess-base

This role downloads / copies vitess to hosts.

### Role Variables

```yaml
use_local_vitess_binary: false  # ensure that vitess binary is placed in files/folder and its name is mentioned in vitess_file variable

## Use the below properties if you are using `use_local_vitess_binary: true` . The vitess tar.gz must be placed in files/ folder
vitess_file: vitess-16.0.2-6076fed.tar.gz
vitess_folder: vitess-16.0.2-6076fed

## Vitess binary url
vitess_binary_url: 'https://github.com/vitessio/vitess/releases/download/v16.0.2/vitess-16.0.2-6076fed.tar.gz'
```
If you want to download vitess from the vitess repo provide the url in variable `vitess_binary_url`.

Incase you want the vitess to be copied from the local folder via ssh:
* Set `use_local_vitess_binary` to `true`
* Place the `vitess-xx.x.x-xxx.tar.gz` in `vitess-base/files/` folder 
* Provide the name of file in `vitess_file` and `vitess_folder` variables


## vtadmin Role

This role sets up the VTAdmin on the designated hosts


### Role Variables

```yaml
vtadmin_port: 15000
vtorc_port: 16000
log_dir:  /tmp
```
These variables define standard ports [vtctld](https://vitess.io/docs/16.0/reference/programs/vtctld/) and [vtorc](https://vitess.io/docs/17.0/user-guides/configuration-basic/vtorc/). 

By default, we do not enable vtorc service. If required, it can be enabled using `systemctl enable vtorc`

`vtadmin_grpc_port` is global variable defined in inventory file, such as `vagrant_1shard/hosts` for single shard.

Restarting vtadmin or vtorc
```sh
systemctl restart vtctld
systemctl restart vtorc
```

## vttablet Role

This role sets up the VTtablet on the designated hosts.

### Role variables

```yaml
update_mycnf: false
install_mysql: true
```
Incase, you want to provide the additional variables in `my.cnf`, set `update_mycnf` to `true` and provide the variables in `vttablet/templates/append_my_cnf.sh.j2` template. If its set to `false`, the default `my.cnf` provided by vitess will be used.

Incase you have mysql already installed on machines, `install_mysql` can be set to `false`. In that case, the role won't install mysql

Setting up vttablet also requires host level variables such as cell id , shard name, vttablet port etc for sharded and unsharded tablets.

The variables are specified in inventory file as follows:

```properties
tablet3 ansible_host=192.168.56.12       tablet_id_sharded=102   shard_name=0        tablet_id_unsharded=1002    shard_name_unsharded=0   tablet_port=15100   tablet_grpc_port=16100     tablet_port_unsharded=15101   tablet_grpc_port_unsharded=16101     mysql_port_sharded=3306  mysql_port_unsharded=3307   init=true  apply_schema=true
```

* `tablet_id_sharded` : vitess tablet id, for example, if tablet_path is cell1-101, `tablet_id_sharded` will be 101
* `tablet_id_unsharded` : Same as above, but for unsharded keyspace
* `shard_name` : 0 for single sharded keyspace, can be `-80` or `80-` for double sharded and so on
* `shard_name_unsharded`: shard name for unsharded keyspace, usually 0
* `tablet_port` : vttablet port for sharded keyspace
* `tablet_grpc_port` : vttablet grpc port for sharded keyspace
* `tablet_port_unsharded` : vttablelt port for unsharded keyspace
* `mysql_port_sharded` : mysql port for sharded keyspace
* `mysql_port_unsharded` : mysql port for unsharded keyspace
* `init` : If true, this shard will be initialized with [initshardprimary](https://vitess.io/docs/16.0/reference/programs/vtctldclient/vtctldclient_initshardprimary/)
* `apply_schema` : If true, this tablet will execute [`ApplyVSchema`](https://vitess.io/docs/16.0/reference/programs/vtctldclient/vtctldclient_applyvschema/) and [`SetKeyspaceDurabilityPolicy`](https://vitess.io/docs/17.0/user-guides/configuration-basic/durability_policy/) commands for the keyspace. This needs to be executed once only per keyspace. Hence we execute this when we have finished setting up the last shard.


Restarting sharded tablet
```sh
systemctl restart commerce
```

Restarting unsharded tablet
```sh
systemctl restart commerce_unsharded
```
Below scripts can be used to init / start / stop / teardown mysql
```sh
/home/vitess/start_commerce.sh
/home/vitess/start_commerce_unsharded.sh

/home/vitess/stop_commerce.sh
/home/vitess/stop_commerce_unsharded.sh

/home/vitess/teardown_commerce.sh
/home/vitess/teardown_commerce_unsharded.sh
```

### vtgate Role

This role sets up the VTGate on the designated hosts

### Role variables

```yaml
## vtgate specific variables
vtgate_mysql_port: 15306
vtgate_port: 15001
vtgate_grpc_port: 15991
```
We expose the vtgate mysql port on 15306. Hence to connect to vtgate post setup, the command will be

```sh
mysql -h 192.168.56.13 -P 15306 -u tester -p
```

Password is not configured for the `tester` user.

The authentication is configured statically as of now and can be configured in `vtgate/files/auth.json` as defined by the [Vitess docs](https://vitess.io/docs/16.0/user-guides/configuration-advanced/static-auth/)

Restarting vtgate
```sh
systemctl restart vtgate
```

## Usage on Vagrant

Apart from using it on staging / production machines. This ansible playbook can also be executed on [Vagrant](https://www.vagrantup.com/)

There are sample configuration already provided for upto 3 shards:

* `vagrant_1shard/` : inventory and Vagrantfile for 1 shard
* `vagrant_2shard/` : inventory and Vagrantfile for 2 shards
* `vagrant_3shard/` : inventory and Vagrantfile for 3 shards

Just cd into the any of the above folders and run `vagrant up` to start the provisioning process.

In the sample setup, we provision each shard with one primary tablet and 2 replicas

### Future Roadmap
* Add [Vitess ACL's](https://vitess.io/docs/17.0/user-guides/configuration-advanced/authorization/)
* Add [etcd](https://etcd.io/) as topology server
* Enable vtadmin UI interface


## Credits
* [sleighzy/ansible-zookeeper](https://github.com/sleighzy/ansible-zookeeper) - zookeeper role is utilized in this role
* [geerlingguy/ansible-role-java](https://github.com/geerlingguy/ansible-role-java) - Java is installed using this role (Required by zookeeper)
