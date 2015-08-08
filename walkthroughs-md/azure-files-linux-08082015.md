Azure has a great preview service that allows you to take advantage of numerous services before they hit the main stream found [here](http://azure.microsoft.com/en-us/services/preview/). One of the fun ones on the block is [Azure Files](https://account.windowsazure.com/PreviewFeatures?fid=xsmb). Azure Files allow Virtual Machines in Microsoft Azure to mount a shared file system using the SMB protocol. Additionally, numerous Virtual Machines can make use of the same file share concurrently, allowing persistent data to easily be shared among numerous instances. Lastly, there are some nice Windows file APIs along with REST API, *similar to the Azure Blob interfaces*, that allow for easy data accessibility.

Of course the Azure File service is a home run for Azure Windows customers, but I was curious as to how Linux handles the service - and what speeds I could derive from utilizing the service.

##### Environment
1. I am using two Ubuntu 14.04 LTS x64 Virtual Machine running on Azure
  * As a side note these are regular A0 standard size images, that do not have premium disk support.
2. I am using an Azure account with the Azure Files service turned on

##### Setting up the Azure File Service

In order to set up the Azure File Service I need to go the new Azure preview portal found at http://portal.azure.com. once you've made your way to the portal you want to go ahead and click on the New button in the top left, click "Data + Storage", and finally "Storage Account". 
![Portal Image](/content/images/2015/08/AzurePortal.PNG)

Go ahead and assign a name to your storage account, choose your Pricing Tier, Subscription, Location, and Diagnostic settings. For resource group I am going to assign this under the same resource group my Azure Virtual Machines are under, but I can not comment on the exact benefits -- Resource Groups documentation is found [here](https://azure.microsoft.com/en-us/documentation/articles/resource-group-overview/). Once done selecting all these options go ahead and click create. **WABHOOOSHH** because the cloud makes resourcing these objects so easy, I feel adding noises is necessary to add some life to an otherwise boring process.

Anyways, by the time you finished trying to comprehend my onomatopoeia your Storage Account is done loaded up and ready for some File Shares. Go ahead and click on "File Shares".

![File Shares](/content/images/2015/08/AzureFileShare.PNG)

If you receive a message notifying you of this "Preview Feature not available" you need to enable this feature and spin up a new storage account again.

![Preview](/content/images/2015/08/Preview.PNG)

Under the new "Files" tab click on Add, and give your new FileShare a meaningful name -- you are going to be using the name a bit. After the share comes up make note of the URL the File Share is given. Generally the URL follows this schema http://[StorageAccountName].file.core.windows.net/[file_name]. Lastly click the keys on the storage account in order to retrieve the primary access keys. **Note** I've blurred mine out because I do not trust you.

![Storage Keys](/content/images/2015/08/StorageKeys.PNG)

Lets go ahead and switch back to our Linux Server, and mount the Azure File. First create the directory where the Azure File is going to rest: ``mkdir -p ~/azure/drive``. Lets change into the `azure` directory right above the drive directory.

Once here lets issue the following command to mount this guy:

where username is the name of the storage account, password is the primary access key, and the piece following the ``//`` is going to be the URL from above.

```
mount.cifs -o forceuid,vers=2.1,username=anugupfile,password=[TOKEN FROM AZURE STORAGE ACCOUNT] //anugupfile.file.core.windows.net/agup drive/
``` 

**Note** If you receive the following error `` cannot access /sbin/mount.cifs: No such file or directory``, you need to install the ``cifs-utils`` package. This can be done on Ubuntu 14.04 LTS by running ``apt-get install cifs-utils``. 

After clicking enter we should be returned to the terminal and now have an Azure File mounted!!!

##### Speed tests

Now having an Azure File mounted in Linux is cool, but what about the speeds that this guy can handle. In order to test I am going to use the following methods.

1. Test write speeds
2. Test read speeds

###### Write Speeds
In order to test write speeds I am going to use the following script. This script takes data from `/dev/urandom` and writes it to a file in the Azure File. It uses `bs` to specify a byte size of 10K and does this 1000 times for a total of 10MB being written to the Azure File. lastly I run a sync in order to force changed blocks to disk.

```bash
#!/bin/bash

for i in {1..100}
do
        echo "speedtest$i";
        dd if=/dev/urandom of=/root/azure/drive/speedtest$i bs=10k count=1000;sync;
done
```

Here are the results: An average of **8.8488099 seconds** to write 10MB, with a standard deviation of **0.64239296 seconds**. Or a not so great speed of **1.130095 MB/s**
![AzureWrite](/content/images/2015/08/WriteAzueFileTest.PNG)


##### Read Speeds
In order to test Read speeds I am going to use the following script. This script takes the one hundred  speedtest files I generated from the write test and reads them with output to `/dev/null`.

```bash
#!/bin/bash

for i in {1..100}
do
        echo "speedtest$i";
        dd if=/dev/zero of=./drive/speedtest$i bs=10k count=1000;sync;
done
```

Drumroll please .... Here are the results: An Average of 1.499051 seconds** to read 10MB, with a standard deviation of **.292453 seconds**. Or nicer speed of **6.6708867 MB/s**

![AzureRead](/content/images/2015/08/ReadAzureFileTest.PNG)

Overall, hope you enjoyed this post and now are able to use the Azure Files Service with your Linux Servers. I might try these speedtests again when the Files service is out of preview, and with some of the premium disk options Azure offers, but until then..



