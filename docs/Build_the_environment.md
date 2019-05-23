# Build the environment

Based on Percona XtraDB Cluster 5.7.25 running on CentOS 7, the environment consist on a 3-node cluster plus an additional VM that will serve as the app server (sysbench) and the PMM host. 

## Requirements

- Vagrant 2.2.0
- VirtualBox 5.2.20

## Get the environment up and running

- Clone the repo: `git clone https://github.com/nethalo/pxc-break-fix-lab.git`
- Go to the directory: `cd pxc-break-fix-lab`
- Install the hostmanager vagrant plugin: `vagrant plugin install vagrant-hostmanager`
- Start the build: `vagrant up; vagrant hostmanager;`

## Login into the VMs

For every machine you want to login into, execute vagrant ssh <name>. For example, to ssh to the first node:

```
vagrant ssh pxc1
```

## 