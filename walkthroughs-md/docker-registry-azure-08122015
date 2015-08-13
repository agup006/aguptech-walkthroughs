The Docker Registry is an application you can run in your datacenter environment to store your organization's Docker images. The Registry application mimics the current Docker Hub registry that all Docker clients use by default. The main appeal of running a separate Docker registry has mostly been control around Docker image storage location and distribution, but there are additional benefits to running a separate Docker Registry that deal specifically with performance. This blog post is a walk through on setting up a Docker Registry on Microsoft Azure, and a comparison of speed savings the Docker Registry can offer.

### My Environment
1. I already have a Docker Swarm set up in Microsoft Azure thanks to Docker-Machine: [Walkthrough](http://email.docker.com/E0T3F0MSJCQ0I0JL3e06K01)
2. I have a local Ubuntu 14.04 LTS x64 server that has docker-machine installed.
3. I have a Microsoft Azure subscription with a storage account

### Setting up a Docker Registry
The easiest part of setting up a Docker registry is that the application runs in a container. First lets create a Docker server in Azure w/ docker-machine

```
# docker-machine create -d azure \
--azure-subscription-id AZURE_SUBSCRIPTION_ID \
--azure-subscription-cert /root/path/to/my/cert.pem \
anugup-dreg
```
Once that new Azure Docker host has loaded up lets use docker-machine to point our local docker client to the the new Azure Docker host. We can simply run the following command.

`eval $(docker-machine env anugup-dreg)`

Now that our Docker client is pointed to the Azure Docker host let's load up the Docker registry container. 

**Note:** There are additional steps to secure this registry and make it public facing that involves getting a certificate from a CA authority. For the purposes of this blog post I am going to set my Docker clients to ignore using an insecure registry.

Run the following commands to create a Docker registry container.

1. `docker run -d -p 5000:5000 --restart=always --name registry registry:2`

Once our Docker registry is running, lets run the following command to disjoin our Docker client from the Azure Docker host and set our Docker client to point back to our local environment: `eval $(docker-machine env -u)`. 

To ensure that the docker client is still not pointed to the server running the registry container run a `docker ps` to make sure the registry does not show up under images. As menitoned in the note above we are going to set our Docker daemon to allow insecure registries. In order to do this edit the `/etc/default/docker` file and modify the following commented line from

```
#DOCKER_OPTS="RANDOM THINGS"
```

to 

```
DOCKER_OPTS="RANDOM THINGS --insecure-registry anugup-dreg.cloudapp.net:5000"
```

Now that we are insecure :) , lets restart our Docker daemon and pull down the official `ubuntu:latest` image and reupload this image as `anugup-ubuntu` to our Docker registry.

```
docker pull ubuntu
docker tag ubuntu http://anugup-dreg.cloudapp.net:5000/anugup-ubuntu
docker push http://anugup-dreg.cloudapp.net:5000/anugup-ubuntu
```

#### Comparing Docker Swarm pull speed
Now that we have our fun insecure Docker Registry setup lets see how much faster our deployments are compared to pulling from the public Docker Hub. The following is a picture representation of the Docker Swarm I have running in Azure below. **Important:** I have configured each of my swarm Docker hosts to also accept images from *insecure-registries* for the purpose of this blog.

![Docker Swarm](/content/images/2015/08/DockerImage.PNG)

In addition to the Ubuntu image I have also upload CentOS, MySQL, Busybox, Redis, and Node images to my Docker registry running in Azure. *To add additional images follow steps from above detailing ubuntu, and replace 'ubuntu' image name as necessary*

First lets switch our Docker client to point to the swarm master. `eval $(docker-machine env --swarm anugup-swarm-master)`. Now when we run a `docker pull` our swarm pulls that image onto each and every node. For example running a `docker pull DOCKER_REGISTRY:PORT/IMAGE_NAME` brings up every node in the swarm, and an appendation of `...: download` once the download on that node is complete.

```
anugup-swarm-mast: Pulling anugup-dreg.cloudapp.net:5000/anugup-ubuntu:latest... : downloaded
anugup-swarm-node1: Pulling anugup-dreg.cloudapp.net:5000/anugup-ubuntu:latest... : downloaded
```

To get an accurate comparison of the pulls between my private registry running in Azure, and the public registry Docker hub has I am going to make use of the following script. This script is in bash, and does the following

1. Declare an array of docker image names
2. Measure the amount of time to pull that image from my private Azure Docker registry and output results to a text file
3. Remove the Docker Images from the Swarm
4. Measure the amount of time to pull the docker image from the public Docker Hub and output results to a text file
5. Remove the Docker Images from the Swarm

The steps 2 + 3, and 4 + 5 are repeated 100 times to get accurate results.

```bash
#!/bin/bash

#declar type array
declare -a arr=( "ubuntu" "centos" "mysql" "redis" "busybox" "node" )

for t in "${arr[@]}"
do
        echo "Starting Tests for $t"
        for i in {1..100}
        do
                echo "PrivateRegTesting$t$i: " >> priv_reg_$t;
                { time docker pull anugup-dreg.cloudapp.net:5000/anugup-$t; } 2>> priv_reg_$t
                docker rmi anugup-dreg.cloudapp.net:5000/anugup-$t;
        done
        for i in {1..100}
        do
                echo"PublicRegTesting$t$i: " >> pub_reg_$t;
                { time docker pull $t;} 2>> pub_reg_$t
                docker rmi $t;
        done
done
```

Here are the results after running this test over a couple hours

![DockerPull2Graph](/content/images/2015/08/DockerPullTime2Comparison.PNG)

When looking at the time saved from having a Docker Private Registry, we do not see any groundbreaking speeds come into the mix.

##### Adding Azure Storage as Docker Registry backend
In addition to storing the Docker images locally on the Docker host, the Docker Registry can also be configured to make use of Azure Storage, Amazon Storage, Google Storage, etc. In order to set this up we simply need to redeploy our Docker registry container with additional options.

Point the Docker client to the Docker Registry host `eval $(docker-machine env anugup-dreg)`. Stop the current Docker registry container: `docker stop REGISTRY_CONTAINER`. Once stopped run the following command to start a new Registry that is using Azure storage to house your enterprises images. The storage account name can be found in the panes of the Azure portal or through the Azure-cli. The storage key is the keys button in the Azure portal, or also found through Azure-cli as well. In case you forget -- [Stack Overflow is awesome](http://stackoverflow.com/questions/6985921/where-can-i-find-my-azure-account-name-and-account-key)

```
# docker run -d -p 5000:5000 \
-e REGISTRY_STORAGE=azure \
-e REGISTRY_STORAGE_AZURE_ACCOUNTNAME="<storage-account-name>" \
-e REGISTRY_STORAGE_AZURE_ACCOUNTKEY="<storage-key>" \
-e REGISTRY_STORAGE_AZURE_CONTAINER="registry" \
--name=registry \
registry:2
```

**Note**: the variable `REGISTRY_STORAGE_AZURE_CONTAINER` and `--name` must be the same.

Once this is up and running we can run through the same bash scripts from above, with no changes needed. Here are the new metrics for the following images: `ubuntu`, `centos`

![RegistryWithThree](/content/images/2015/08/Registry3.PNG)

This was interesting to me as I thought using the storage account would net me a much faster `docker pull`. To verify I even double checked my storage account metrics to make sure the storage account was being used  :)

![AzureProof](/content/images/2015/08/AzureFile.PNG)

Altogether using a private Docker Registry in Azure shaves off a couple seconds for pulling an image in your docker swarm. There are definitely optimizations that can be done here such as premium disk in Azure - but I'll leave that for others to tell.
