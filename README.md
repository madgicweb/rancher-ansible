# Project ansible to manage Rancher

## Foreword

If you don't know yet rancher please read the official documentation http://docs.rancher.com/rancher/latest/en/.

Rancher will be used to create segregated environments for projects.

Each project/environments should exists as "environments" in rancher (example: my-uat-project my-production-project ).
Each rancher environment is segregated at network level via Docker Overlay and IPSec tunnels (managed by rancher itself).

Only trusted ops can connect to the machines, unix accounts are managed by ansible.

End users will have tools in their project to see their application logs & metrics (elk, prometheus).

### Machine requirements

At the moment this set of playbooks is designed to be run on **ubuntu**. We recommend using latest LTS (16.04 at times of writing).

Each host (including the master) must have at least the following requirements :
* 2 vcpu
* 4 Go ram
* 2 disks with one for the system with 12Go (8Go for / + 4Go for swap) and one for /var/lib/docker with 8Go.

*Note that by default we will only install docker on the machine, so the root filesystem doesn't needs to be big (just system logs and the base packages).
Docker will have it's own partition and everything will run inside containers, so if at some point you need more space you need to extends docker partition (see above)*

### Versions
#### Defined into configuration
* rancher/server:v1.6.5
* rancher/agent:v1.2.5
* rancher-compose-v0.12.5
* docker: 1.12.6
* dockerpy: 1.8.1


### Exemple for test environments

2 hosts per rancher environment :
* one for tools (elk, prometheus, janitor) and the load balancer
* one for project

The machine for tools will be identified with specific label on it (role=tools).

### Vagrant

If you use proxy, you need to install vagrant plugin :
```
vagrant plugin install vagrant-proxyconf
```
To create all node  
```
vagrant up
```
To create specific node  (Ex: Master)
```
vagrant up master
```

## Playbooks documentation

### Requirements

* Ansible >= 2.1.X

### Workflow to setup rancher from scratch

#### Prerequisites
* Duplicate "vagrant" directory to init your environment and define your inventory file and others configuration files
* Config "conf/ssh.config" to connect into your target servers (User and IdentityFile )
* Define docker_disks="/dev/sdb,/dev/sdc" and docker_disks_options_enabled: false|true in docker role

#### Principal Playbooks
* run configure_host playbook
* run create_master playbook
* run create_master_tools playbook
* run create_project playbook (everytime you need to setup a new env)


### Configure the Machine
Launch the playbook

You can bootstrap the machine for the first time with the following command :

* If ssh key is already defined :

```
ansible-playbook configure_host.yml -i path/to/inventory
```

* If you want to access by generic account / Password already create in template :

```
ansible-playbook_wrapper configure_host.yml -u genericaccount -Kk
```

This ansible install

* Python
* Docker
* Create local user with ssh_key, put into the group ADM,SUDO and set the password
* *Set machine hostname* Disabled
* *(Configure ssh (disable root login, enforce auth by key)* Disabled
* Remove the former ops account

*Docker will be installed in /var/lib/docker folder with a lvm partition by default it will create it on /dev/sdb but you can configure it in you inventory file by overriding docker_disks variable.
```
mymachine docker_disks="/dev/sdb,/dev/sdc"
```
Note that the playbook will auto extends to 100% of the volume group the /var/lib/docker. So if in future you're out of space on docker partition just override that property and rerun the playbook.*


###  Create a Master

Launch the playbook
```
./ansible-playbook create_master.yml -i path/to/inventory
```

This playbook will create a file apiKey in {{ inventory_dir }}/group_vars/all/apikey. This file contains the RANCHER_API_KEY_ACCOUNT_TOKEN and RANCHER_API_KEY_ACCOUNT_SECRET

### Create a master tools

The host in tools group will become the "tools" host for monitoring the rancher cluster (running elk, prométheus, grafana)

Launch the playbook
```
./ansible-playbook create_master_tools.yml -i path/to/inventory
```

### Create a project

The first host of the tag will become the "tools" host for the project
You can install elk, prometheus, garfana, janitor if you tag "common_stack_enabled: true" in group_vars/vars

Launch the playbook
```
./ansible-playbook create_project.yml -i path/to/inventory -e "NAME_PROJECT=MyFirstProject" -e "TYPE_PROJECT=cattle"
```

Tree type of project is possible :
* cattle (By Rancher)
* mesos
* kubernetes


This ansible create :

* A project into RANCHER
* Create "API KEY ENVIRONMENT" into rancher and write into group_vars/{{NAME_PROJECT}}
* Add Host into the project
* Install common stacks : Elk, Prometheus, Janitor and load balancer configuration if you have tag "common_stack_enabled: true"


### Add Host to a project

Launch the playbook utils_add_host_to_project.yml with the good project and playbook install all of node defined into the group_vars in inventory.
```
./ansible-playbook_wrapper utils_add_host_to_project.yml -e "NAME_PROJECT=myproject"
```

The playbook will launch on each host the agent Rancher with the good configuration.

### Utils clean project

To delete all hosts in a project run the playbook utils_add_registry.yml
```
./ansible-playbook_wrapper utils_clean_host.yml -K -e "NAME_PROJECT=myproject"
```

To delete one host in a project run the following command
```
./ansible-playbook_wrapper utils_clean_host.yml -K -e "NAME_PROJECT=myproject" -e "NAME_HOST=myHost
```

## Deployments

Deployments will be performed to developers via jenkins.
We will provide to dev teams a jenkins instance for their projects with a sample of deployment pipeline.
Our jenkins instance will have rancher-compose installed and a few utils scripts to simplify deployment.

rancher-compose will require an apikey per project.

Pipeline sample
```
withEnv(['RANCHER_COMPOSE_HOME=/usr/lib/rancher-compose', 'RANCHER_URL=http://rancher-master:8080/','RANCHER_SECRET_KEY=ABCD','RANCHER_ACCESS_KEY=ABCD']) {
    node {
       stage 'Prepare deploy'
       git credentialsId: 'projectname', url: 'git@your-git-repo'
       stage 'Deploy to UAT'
       sh '${RANCHER_COMPOSE_HOME}/deploy-stack.sh your-repo-git/deploy'
    }
    stage 'Validation'
    try {
        input message: 'UAT valid ?', ok: 'Yes please deploy to prod !'
    } catch (InterruptedException e) {
        node {
            sh '${RANCHER_COMPOSE_HOME}/rollback-stack.sh your-repo-git/deploy'
        }
        throw e
    }
    node {
       stage 'Deploy to Prod'
       sh '${RANCHER_COMPOSE_HOME}/confirm-deploy-stack.sh your-repo-git/deploy'
    }
    stage 'Validation in production'
    input message: 'Production OK ?', ok: 'Yes'

}
```


## Troubleshooting

### On master side

Check host global heatlh (freespace, memory, cpu).

Check docker state
```
sudo service docker status
```

Check docker daemon logs
```
journalctl -u docker
```

Check system logs
```
dmesg
```

Check the logs of rancherServer container.
```
sudo docker logs --tail=1000 -f rancherServer
```

### On environments hosts

Check container logs, if it doesn't work check the log of network agent container.

Check host global heatlh (freespace, memory, cpu).

Check docker state
```
sudo service docker status
```

Check docker daemon logs
```
journalctl -u docker
```

Check system logs
```
dmesg
```

## Disaster recovery

If for some reason the host running rancher-master dies, application will remain up and running, so there is no impact.
If after running troubleshooting steps you don't have any clue we recommend you to wipe the master machine spawn a new master with "create_master.yml" playbook and run mysql restore script (see above).


### Quick cleanup procedure (optional)
This step can be skipped if you spawn a new master.

Run the following commands on your rancher-master
```
sudo service docker stop
rm -rf /var/lib/docker/*
rm -rf /var/lib/mysql/*
sudo service docker start
```

### Restore mysql

On the master enter into the container itself and run the restore script
```
sudo docker exec -ti mysql-backup-s3 bash
./restore.sh
```

This will restore the latest backup find on s3.

If you want to restore a specific backup you can do
```
ID_BUCKET_RESTORE=2016-07-21T133544Z.dump.sql.gz ./restore.sh
```

## Upgrading rancher

To upgrade rancher you just need to change the rancher_version version in group vars.
Exemple: in production/group_vars/all/rancher-config
```
rancher_version: v1.1.1
```

We highly recommend to stick to a specific version for production rancher environment to make sure everything is repeatable.

Then just run again the create_master.yml playbook.
It will upgrade rancher-master smoothly.
When rancher-master is up again it will contact environments host agents and update it if necessary, this operation is done by rancher itself.

## Upgrading docker

To update docker you just need to set the docker_version variable.
If you want to set this version to all rancher platforms, just update it in roles/docker/defaults/main.yml.
If you want to set this version only for a particular rancher platform, add this variable in your group_vars/all/vars file.

Then run again the configure_host.yml playbook. Note that you will get downtime if your application containers are not resilient as docker will be restarted.
