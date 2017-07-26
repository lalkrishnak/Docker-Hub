# Docker-Hub
Repo that contains all docker related files

Keep in mind that Docker is not the first and not the only containerization platform. However, at the moment Docker is the biggest and the most powerful player on the market.

## The list of benefits of Docker are the following:
* Faster development process. There is no need to install 3rd parties like PostgreSQL, Redis, Elasticsearch. Those can be run in containers.
* Handy application encapsulation (you can deliver your application in one piece).
* Same behaviour on local machine / dev / stage / production servers.
* Easy and clear monitoring.
* Easy to scale (if you’ve done your application right it will be ready to scaling not only in Docker).

## Installation
Docker can be easily install via packages provided in all major Linux distributions. 
For Ubuntu-14.04 we run:
*$ sudo sh -c "wget -qO- https://get.docker.io/gpg | apt-key add -"*
*$ sudo sh -c "echo deb http://get.docker.io/ubuntu docker main > /etc/apt/sources.list.d/docker.list"*
*$ sudo aptitude update*
*$ sudo aptitude install lxc-docker apparmor*

To confirm successful installation:

$ sudo docker info
Containers: 0
Images: 0
Storage Driver: aufs
 Root Dir: /var/lib/docker/aufs
 Dirs: 0
Execution Driver: native-0.2
Kernel Version: 3.13.0-39-generic
Operating System: Ubuntu precise (12.04.5 LTS)
WARNING: No swap limit support
 
$ sudo docker version
Client version: 1.3.1
Client API version: 1.15
Go version (client): go1.3.3
Git commit (client): 4e9bbfa
OS/Arch (client): linux/amd64
Server version: 1.3.1
Server API version: 1.15
Go version (server): go1.3.3
Git commit (server): 4e9bbfa

Download and run the check-config script from the docker repository https://github.com/docker/docker.git to check if all dependencies are satisfied:

$ sudo ./check-config.sh
warning: /proc/config.gz does not exist, searching other paths for kernel config...
info: reading kernel config from /boot/config-3.13.0-39-generic ...
Generally Necessary:
- cgroup hierarchy: properly mounted [/sys/fs/cgroup]
- apparmor: enabled and tools installed
- CONFIG_NAMESPACES: enabled
- CONFIG_NET_NS: enabled
- CONFIG_PID_NS: enabled
- CONFIG_IPC_NS: enabled
- CONFIG_UTS_NS: enabled
- CONFIG_DEVPTS_MULTIPLE_INSTANCES: enabled
- CONFIG_CGROUPS: enabled
- CONFIG_CGROUP_CPUACCT: enabled
- CONFIG_CGROUP_DEVICE: enabled
- CONFIG_CGROUP_FREEZER: enabled
- CONFIG_CGROUP_SCHED: enabled
- CONFIG_MACVLAN: enabled
- CONFIG_VETH: enabled
- CONFIG_BRIDGE: enabled
- CONFIG_NF_NAT_IPV4: enabled
- CONFIG_IP_NF_FILTER: enabled
- CONFIG_IP_NF_TARGET_MASQUERADE: enabled
- CONFIG_NETFILTER_XT_MATCH_ADDRTYPE: enabled
- CONFIG_NETFILTER_XT_MATCH_CONNTRACK: enabled
- CONFIG_NF_NAT: enabled
- CONFIG_NF_NAT_NEEDED: enabled
Optional Features:
- CONFIG_MEMCG_SWAP: enabled
- CONFIG_RESOURCE_COUNTERS: enabled
- CONFIG_CGROUP_PERF: enabled
- Storage Drivers:
  - "aufs":
    - CONFIG_AUFS_FS: enabled
    - CONFIG_EXT4_FS_POSIX_ACL: enabled
    - CONFIG_EXT4_FS_SECURITY: enabled
  - "btrfs":
    - CONFIG_BTRFS_FS: enabled
  - "devicemapper":
    - CONFIG_BLK_DEV_DM: enabled
    - CONFIG_DM_THIN_PROVISIONING: enabled
    - CONFIG_EXT4_FS: enabled
    - CONFIG_EXT4_FS_POSIX_ACL: enabled
    - CONFIG_EXT4_FS_SECURITY: enabled
  - "overlayfs":
    - CONFIG_OVERLAYFS_FS: enabled

If not running we can start the service and set it for auto start:

$ sudo service docker start
$ sudo update-rc.d docker enable
After that we should see the default network bridge docker0 and its network 172.17.0.0/16 created by Docker.

## Docker Images

The building base for Docker images is a file so called Dockerfile. Here we specify the building blocks of the image using specific Docker command language. Check the Docker documentation for more details.

We will create separate Docker image for each of our services like Tomcat, MongoDB, ElasticSearch etc. This means that to run a local Devtest environment on Docker one will need to run three containers and setup appropriate networking between them.

### Elastic Search
This is going to be our ElasticSearch Dockerfile:

Here we specify everything related to the image starting from the Linux distribution and version, in this case we download and build on the official public Ubuntu-14.04 Docker image, Ansible integration, services to start when the image gets launched in a Docker container and the ports it will expose to the clients. Then we create two Ansible playbooks in the current working directory that will do all the installation and configuration job for us. 

The main playbook elastic_search.yml:


and the tasks playbook attached elastic_search_main_tasks.yml. We gonna drop in our cloned repository in the image and run them during creation. To start the build we simply run:

$ sudo docker build --rm -t lalkrishna/elastic_search .

The --rm switch removes the intermediate containers upon successful build. Docker works in layers thus the containers build on top of the previous one (meaning for each instruction in the Dockerfile it starts a new container, executes the command, shuts down that container and removes it after committing it into a image that will be used for a new container in the next step) and we don’t want those left lying around. When finished running we will have a new Docker image tagged latest in the local encompass/elastic_search repository:

$ sudo docker images
REPOSITORY                 TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
lalkrishna/elastic_search  latest              68fac36afdb9        24 hours ago        1.735 GB
ubuntu                     14.04               5506de2b643b        3 weeks ago         197.8 MB

Apart our new image we also see the downloaded official Ubuntu image we based our build upon. We want to keep it so we don’t have to download it over and over again in the next builds. This is one nice feature that Docker caching provides.

### MongoDB
This is going to be our MongoDB Dockerfile:

FROM ubuntu:14.04

and the main playbook mongodb.yml:


and the attached mongodb_main_tasks.yml playbook for the setup. Similar to the previous example we build our image:

$ sudo docker build --rm -t lalkrishna/mongodb .
But in this case we have some additional job to do. After starting a container with this initial image:

$ sudo docker run --name="MongoDB" --rm -t -i lalkrishna/mongodb
which will also log us in, we need to run the following inside the container:

$ /usr/bin/mongo /tmp/create-lalkrishna-db-and-users.js
$ /usr/bin/mongo /tmp/create-audit-db-and-users.js
$ for col in User Site SubscriptionType Account; do /usr/bin/mongorestore --port 27017 --db lalkrishna --username lalkrishna --password password --collection $col --drop /tmp/${col}.collection/lalkrishna/${col}.bson; done

to create our databases, users and collections. The files needed have been already placed under /tmp during our image creation by Ansible but couldn’t be ran since of course no services can be started inside the image it self at that point. Now we need to commit this container in order to create our final image:

$ sudo docker commit 8111c11050c5 lalkrishna/mongodb:latest
where 8111c11050c5 is the container id we find by running:

$ sudo docker ps
and finding the one named MongoDB. After we exit from this container, it will be automatically deleted since we have started it with the --rm switch.

### Tomcat
We follow the same procedure in this case too. We change the Dockerfile for tomcat configuration:


the main playbook tomcat7.yml:


and the attached tomcat7_main_tasks.yml file. We run the build command with only these three files in the current working directory to build the Tomcat image:

$ sudo docker build --rm -t lalkrishna/tomcat7 .
At the end we have our three images built and ready to be used:

$ sudo docker images
REPOSITORY                 TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
lalkrishna/tomcat7         latest              ad640bbc8e1e        About an hour ago   2.012 GB
lalkrishna/mongodb         latest              60c7f5c41c5e        5 hours ago         2.985 GB
lalkrishna/elastic_search  latest              68fac36afdb9        25 hours ago        1.735 GB
ubuntu                     14.04               5506de2b643b        3 weeks ago         197.8 MB
The next step is to create some containers.

## Docker Containers

Creating containers from our images is fairly simple. This is how we do it, the -d switch send the process in the background:

$ sudo docker run -d --name="ElasticSearch" -t -i lalkrishna/elastic_search:latest
$ sudo docker run -d --name="MongoDB" -t -i lalkrishna/mongodb:latest
$ sudo docker run -d --name="Tomcat" -t -i lalkrishna/tomcat7:latest
and there we have our containers up and running:

$ sudo docker ps
CONTAINER ID        IMAGE                             COMMAND             CREATED             STATUS              PORTS                                                NAMES
82b1f9d4d872        lalkrishna/tomcat7:latest         "/bin/bash"         4 hours ago         Up 4 hours          443/tcp, 80/tcp, 8998/tcp, 8999/tcp, 22/tcp          Tomcat             
78aeae57d011        lalkrishna/mongodb:latest         "/bin/bash"         7 hours ago         Up 7 hours          28017/tcp, 28018/tcp, 22/tcp, 27017/tcp, 27018/tcp   MongoDB            
e4e2bca92bce        lalkrishna/elastic_search:latest  "/bin/bash"         28 hours ago        Up 7 hours          22/tcp, 9200/tcp, 9300/tcp                           ElasticSearch

We can stop and start them:

$ sudo docker stop 82b1f9d4d872
$ sudo docker start 82b1f9d4d872
or attach them in case we want to do some work inside:

$ sudo docker attach 82b1f9d4d872
To detach from it we use the Ctrl+P followed by Ctrl+Q keyboard sequence. If we leave the container with exit command it will shut the container down as well. To completely remove a container we need to stop it first and then wipe it off using its id or name as target:

$ sudo docker rm [container-name|container-id]
To remove an image, first we need to make sure it is not in use by any container and then run:

$ sudo docker rmi [image-name|image-id]
We can always remove it forcefully if needed by using the --f switch in the above command.

Another feature I like about Docker is the way we can dynamically dedicate resources for the containers on start up. For example:

$ sudo docker run --rm -c 512 --cpu 2 -m 512m -t -i lalkrishna/elastic_search:latest
will start the container and limit its resources to 2 CPU’s with 50% of their processing time and 512MB of RAM. We can even tell the container on which CPU’s we want it to run via --cpuset option, for example --cpuset=0,2.

If we want our container to be started together with the docker service we can add --restart always option to the docker run command. That way we can make sure our container is constantly running even after docker or host restart.

Since we have exposed some service ports inside our custom built images, now we can connect to them from the host after we find the IP address of our containers. For Tomcat one for example:

$ sudo docker inspect --format '{{ .NetworkSettings.IPAddress }}' 82b1f9d4d872
172.17.0.138
 
$ telnet 172.17.0.138 443
Trying 172.17.0.138...
Connected to 172.17.0.138.
Escape character is '^]'.
^]
telnet> quit
 
## Docker Repository

Now that we have our images ready we need to make them available for the rest of the Encompass users. The simplest and fastest way is using the Docker Hub. I have created an account and a private repository for our images. Since the repository name is <my-user>/<my-repository> I need to tag my images accordingly to be able to push them:

$ sudo docker tag lalkrishna/elastic_search:latest <my-user>/<my-repository>:elastic_search
$ sudo docker tag lalkrishna/mongodb:latest <my-user>/<my-repository>:mongodb
$ sudo docker tag lalkrishna/tomcat7:latest <my-user>/<my-repository>:tomcat7
Now just need to run:

$ sudo docker login
$ sudo docker push <my-user>/<my-repository>:elastic_search
$ sudo docker push <my-user>/<my-repository>:mongodb
$ sudo docker push <my-user>/<my-repository>:tomcat7
and the images will appear in the Hub repository with tags of elastic_search, mongodb and tomcat7.

Converting Docker Container into Image

Lets say we have a new version of our app we have deployed into our local Tomcat container Deploying Encompass In Docker Containers. Or we have made some important configuration changes to our local instance that we want to make official and propagate to the rest of the users. After we have finished with our changes, all we need to do is commit the container into new image tagging it as appropriate and push that image to our private Docker repository. For example if our Tomcat container has an id of e4e2bca92bce:

$ sudo docker stop e4e2bca92bce
$ sudo docker commit e4e2bca92bce <my-user>/<my-repository>:AugustRelease
$ sudo docker push <my-user>/<my-repository>:AugustRelease
$ sudo docker start e4e2bca92bce
Now the rest of the users can pull this new version of the image, shutdown their old Tomcat container (optional) and start a new one using this new image:

$ sudo docker pull <my-user>/<my-repository>:AugustRelease
$ sudo docker run -d --name="Tomcat" --link MongoDB:db --link ElasticSearch:es -p 443:443 -v /opt/lalkrishna/deploy:/opt/lalkrishna/deploy -t -i <my-user>/<my-repository>:AugustRelease

In this way we can have more than one version of our application running locally. We can keep the initial version running and start a new one in parallel and both will share same MongoDB and ElasticSearch resources. Of course, if we want we can produce separate images for these services as well similar to what we have done with Tomcat and run two completely separate stacks in parallel.
