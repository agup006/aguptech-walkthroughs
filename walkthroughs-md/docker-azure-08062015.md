With Docker being the best thing since sliced bread, I felt it necessary to provide a simple how-to guide on Docker's more robust tool sets. There are a ton of guides out there that go over the basic Docker principles and show you how to type `docker run`; this guide will focus on using Docker Machine and Docker Swarm in conjunction with Microsoft Azure.

Let's start with some extremely basic background on the Docker tools:

###### Docker Machine
https://docs.docker.com/machine/ - With the phrase "zero to docker" as the tagline, Docker Machine was built for easing up deployments of Docker Hosts on whatever cloud provider, Hypervisor, and platform you are using.

###### Docker Swarm
https://docs.docker.com/swarm/ - Swarm is a native clustering tool for Docker that abstracts all your Docker Hosts into a large sentient Docker being.

###### Example
Or as I like to sum up these tools to my co-workers, Docker Machine is the guy who makes the party and Docker Swarm is the guy who knows the best party to go to.

###### Pre-requisites
1. A Microsoft Azure Subscription - The free trial for Azure is pretty good
2. A Docker Server - I am using an Ubuntu 14.04 LTS x64 VM running in Azure
3. A Keyboard, Mouse, and exactly one beer *for time purposes*

###### Installing Docker Machine
Installing Docker Machine is thankfull pretty trivial and only requires you to copy and paste the following commands from the Docker Machine documentation. These commands grab the lastest blessed release of Docker Machine, move the file to a local `/bin`, and modify the file to make it executable for your running pleasure. Lastly we issue a `docker-machine -v` to check version and correct installation.

1. `curl -L https://github.com/docker/machine/releases/download/v0.3.0/docker-machine_linux-amd64 > /usr/local/bin/docker-machine`
2. `chmod +x /usr/local/bin/docker-machine`
3.  `docker-machine -v`
```
# curl -L https://github.com/docker/machine/releases/download/v0.3.0/docker-machine_linux-amd64 > /usr/local/bin/docker-machine
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   402    0   402    0     0   1205      0 --:--:-- --:--:-- --:--:--  1203
100 11.8M  100 11.8M    0     0  6475k      0  0:00:01  0:00:01 --:--:-- 16.8M

# chmod +x !$
chmod +x /usr/local/bin/docker-machine

# docker-machine -v
docker-machine version 0.3.0 (0a251fe)
```
**Note:** if you get the following error, make sure you are downloading the correct OS Type 
`-bash: docker-machine: cannot execute binary file: Exec format error`

###### Preparing Azure for Docker Machine
Now that Docker Machine is up and running, lets take a step up to the cloud to prepare Azure for the fun times ahead. In order for Docker Machine to leverage your Azure subscription we need to grab our Azure Subscription ID as well as upload certificates for Docker Machine to authenticate itself with. 

Let's start with certificate creation with openssl. These certificates are used to confirm your identity every time you talk to Azure with docker-machine. 

Ralph Squilace on the Azure side of the world has already enumerated the openssl and management console steps [here](https://azure.microsoft.com/en-us/documentation/articles/virtual-machines-docker-machine/), and I have reiterated below. Run the following commands to generate certificate goodness; enter the necessary information.

- `openssl req -x509 -nodes -days 365 -newkey rsa:1024 -keyout mycert.pem -out mycert.pem`
- `openssl pkcs12 -export -out mycert.pfx -in mycert.pem -name "My Certificate"`
- `openssl x509 -inform pem -in mycert.pem -outform der -out mycert.cer`

Once these commands are run you will have the following `mycert.cer`,`mycert.pem`, and `mycert.pfx` all ready to go. 

Lets grab `mycert.cer` and add the certificate as part of our Management Certificates in the Azure Portal. 

Login to the Azure Portal and scroll down to the bottom section on the left hand side and click "SETTINGS"

![SETTINGS](/content/images/docker-azure-08062015/Settings.PNG?raw=true)
After clicking SETTINGS click "Management Certificates" at the top and "Upload a the bottom". Choose `mycert.cer` and begin the upload
![UPLOAD](/content/docker-azure-08062015/Upload.PNG?raw=true)

Sweet now that we have the cert squared away lets also grab our Azure Subscription ID. The easiest way to do this is to click the Subscriptions Tab in the top right, and copy and paste the GUID.
![SubscriptionID](/content/docker-azure-08062015/Subscription.PNG?raw=true)

###### Running Docker Machine to create Azure Docker Swarm
Now that Docker Machine and Azure are good to go, lets leverage the Azure driver integration to spin up a Docker Swarm running in the Microsoft cloud.

On the Docker Server lets run a basic `docker run` to generate the swarm token our Docker Swarm is going to use

```
# docker run swarm create
83da01c49f2859146737d797d8dc9e6e
```

With our swarm token ready lets take a look at all of the azure command line options by running `docker-machine create | grep azure`

```
docker-machine create | grep azure
   --azure-docker-port "2376"                                                                                           Azure Docker port
   --azure-image                                                                                                        Azure image name. Default is Ubuntu 14.04 LTS x64 [$AZURE_IMAGE]
   --azure-location "West US"                                                                                           Azure location [$AZURE_LOCATION]
   --azure-password                                                                                                     Azure user password
   --azure-publish-settings-file                                                                                        Azure publish settings file [$AZURE_PUBLISH_SETTINGS_FILE]
   --azure-size "Small"                                                                                                 Azure size [$AZURE_SIZE]
   --azure-ssh-port "22"                                                                                                Azure SSH port
   --azure-subscription-cert                                                                                            Azure subscription cert [$AZURE_SUBSCRIPTION_CERT]
   --azure-subscription-id                                                                                              Azure subscription ID [$AZURE_SUBSCRIPTION_ID]
   --azure-username "ubuntu"                                                                                            Azure username
```

Lets also take a look at the swarm commands by running `docker-machine create | grep swarm`

```
# docker-machine create | grep swarm
You must specify a machine name
   --swarm                                                                                                              Configure Machine with Swarm
   --swarm-image "swarm:latest"                                                                                         Specify Docker image to use for Swarm [$MACHINE_SWARM_IMAGE]
   --swarm-master                                                                                                       Configure Machine to be a Swarm master
   --swarm-discovery                                                                                                    Discovery service to use with Swarm
   --swarm-strategy "spread"                                                                                            Define a default scheduling strategy for Swarm
   --swarm-opt [--swarm-opt option --swarm-opt option]                                                                  Define arbitrary flags for swarm
   --swarm-host "tcp://0.0.0.0:3376"                                                                                    ip/socket to listen on for Swarm master
   --swarm-addr                                                                                                         addr to advertise for Swarm (default: detect and use the machine IP)
```

Lets start out by creating our docker machine swarm master with the following command. Replace AZURE\_SUBSCRIPTION\_ID, DOCKER\_SWARM\_TOKEN\_ID, and certificate path with the appropriate values and add any command line options you deem fit. The certificate should be the same certificate uploaded to the Azure Portal. 

**Additional Note:** Azure requires a unique service name, and errors out if a duplicate service name is found.

```
# docker-machine create -d azure \
--azure-subscription-id AZURE_SUBSCRIPTION_ID \
--azure-subscription-cert /root/path/to/my/cert.pem \
--swarm --swarm-discovery=token://DOCKER_SWARM_TOKEN_ID \
--swarm-master anugup-swarm-master
```

Docker Machine is now talking to Azure to create your beautiful new Swarm Master, this process usually takes a couple minutes. In the mean time if you navigate to the Azure Portal you can watch the Azure VM appear and start running after around 30 seconds
![SwarmMasterSpinningUp](/content/docker-azure-08062015/SwarmMaster.PNG?raw=true)

Once our docker swarm master is done spinning up, if we issue a `docker-machine ls` we should see our swarm master populated

```
# docker-machine ls
NAME                  ACTIVE   DRIVER   STATE     URL                                           SWARM
anugup-swarm-master            azure    Running   tcp://anugup-swarm-master.cloudapp.net:2376   anugup-swarm-master (master)
```

Our swarm master is most likely getting lonely. Lets create a couple docker host friends he can boss around

Following the same command structure used previously, remove the option `--swarm-master` and change the machine name.

```
# docker-machine create -d azure \
--azure-subscription-id AZURE_SUBSCRIPTION_ID \
--azure-subscription-cert /root/path/to/my/cert.pem \
--swarm --swarm-discovery=token://DOCKER_SWARM_TOKEN_ID \
  anugup-swarm-01
```
 Once completed, issue another `docker-machine ls` to see all the swarm nodes in their full glory

```
# docker-machine ls
NAME                  ACTIVE   DRIVER   STATE     URL                                           SWARM
anugup-swarm-01                azure    Running   tcp://anugup-swarm-01.cloudapp.net:2376       anugup-swarm-master (master)
anugup-swarm-02                azure    Running   tcp://anugup-swarm-02.cloudapp.net:2376       anugup-swarm-master (master)
anugup-swarm-master            azure    Running   tcp://anugup-swarm-master.cloudapp.net:2376   anugup-swarm-master (master)
```

Lets point docker to the swarm by issuing the following command `eval $(docker-machine env --swarm anugup-swarm-master)`. Now lets issue a `docker info` to get a layout of all the swarm resources in our docker swarm

```
# docker info
Containers: 4
Images: 3
Role: primary
Strategy: spread
Filters: affinity, health, constraint, port, dependency
Nodes: 3
 anugup-swarm-01: anugup-swarm-01.cloudapp.net:2376
  └ Containers: 1
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.721 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=3.13.0-36-generic, operatingsystem=Ubuntu 14.04.1 LTS, provider=azure, storagedriver=aufs
 anugup-swarm-02: anugup-swarm-02.cloudapp.net:2376
  └ Containers: 1
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.721 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=3.13.0-36-generic, operatingsystem=Ubuntu 14.04.1 LTS, provider=azure, storagedriver=aufs
 anugup-swarm-master: anugup-swarm-master.cloudapp.net:2376
  └ Containers: 2
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.721 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=3.13.0-36-generic, operatingsystem=Ubuntu 14.04.1 LTS, provider=azure, storagedriver=aufs
CPUs: 3
Total Memory: 5.163 GiB
```

And there we have it, a Docker Swarm deployed to Azure with Docker Machine. 

**Now you can drink that beer**
