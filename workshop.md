# 1. Kubernetes Workshop

This workshop/tutorial contains a number of different sections, each addressing a specific aspect of running workloads (containers) in Kubernetes in Azure.

You will go through the following steps to complete the workshop:

* Use Azure Portal and Azure Cloud Shell
* Setup Azure Container Registry to build and store docker images
* Create Kubernetes Cluster using AKS (Azure Kubernetes Service)
* Deploy and expose applications
## TODO:  Cluster upgrade. 
* Storage mounting options in AKS ​
* Secret Management in AKS



# 2. Prerequisites

## 2.1 Subscription
You need a valid Azure subscription. To use a specific subscription, use the ````account```` command like this (with your subscription id):
````
az account set --subscription <subscription-id>
````

## 2.2. Azure Portal

To make sure you are correctly setup with a working subscription, make sure you can log in to the Azure portal. Go to <https://portal.azure.com> Once logged in, feel free to browse around a little bit to get to know the surroundings!

It might be a good idea to keep a tab with the Azure Portal open during the workshop, to keep track of the Azure resources you create. We will almost exclusively use CLI based tools during the workshop, but everything we do will be visible in the portal, and all the resources we create could also be created using the portal.

## 2.3. Azure Cloud Shell

We will use the Azure Cloud Shell (ACS) throughout the workshop for all our command line needs. This is a web based shell that has all the necessary tools (like kubectl, az cli, helm, etc) pre-installed.

Start cloud shell by typing the address ````shell.azure.com```` into a web browser. If you have not used cloud shell before, you will be asked to create a storage location for cloud shell. Accept that and make sure that you run bash as your shell (not powershell).

**Protip: You can use ctrl-c to copy text in cloud shell. To paste you have to use shift-insert, or use the right mouse button -> paste. If you are on a Mac, you can use the "normal" Cmd+C/Cmd+V.**

**Protip II: Cloud Shell will time out after 20 minutes of inactivity. When you log back in, you will end up in your home directory, so be sure to ````cd```` into where you are supposed to be.**

# 3. Deploy and expose applications

## 3.1. Get the code

The code for this workshop is located in the same repository that you are looking at now. To *clone* the repository to your cloud shell, do this:

```bash
git clone https://github.com/pelithne/k8s-short.git
```

Then cd into the repository directory:

````bash
cd k8s-short
````

## 3.2. View the code

Azure Cloud Shell has a built in code editor, which is based on the popular VS Code editor. To view/edit all the files in the repository, run code like this:

````bash
code .
````

You can navigate the files in the repo in the left hand menu, and edit the files in the right hand window. Use the *right mouse button* to access the various commands (e.g. ````Save```` and ````Quit```` etc).

For instance, you may want to have a look in the ````application/azure-vote-app```` directory. This is where the code for the application is located. Here you can also find the *Dockerfile* which will be used to build your docker image, in a later step.

## 3.3. Create Resource Group

All resources in Azure exists in a *Resource Group*. The resource group is a "placeholder" for all the resources you create. 

All the resources you create in this workshop will use the same Resource Group. Use the commnd below to create the resource group.

If you are working in a shared subscription, make sure that you create a uniqely named resource group, eg by using your corporate signum.

````bash
az group create -n <resource-group-name> -l westeurope
````

## 3.4. Create an Azure Container Registry (ACR)

You will use a private Azure Container Registry to *build* and *store* the docker images that you will deploy to Kubernetes. The name of the the ACR needs to be globally unique, and should consist of only lower case letters. You could for instance use your corporate signum.

The reason it needs to be unique, is that your ACR will get a Fully Qualified Domain Name (FQDN), on the form ````<Your unique ACR name>.azurecr.io````

The command below will create the container registry and place it in the Resource Group you created previously.

````bash
az acr create --name <your unique ACR name> --resource-group <resource-group-name> --sku basic
````

## 3.5. Build images using ACR

Docker images can be built in a number of different ways, for instance by using the docker CLI. Another (and sometimes easier) way is to use *Azure Container Registry Tasks*, which is the approach we will use in this workshop.

The docker image is built using a so called *Dockerfile*. The Dockerfile contains instructions for how to build the image. Feel free to have a look at the Dockerfile in the repository (once again using *code*):

````bash
code application/azure-vote-app/Dockerfile
````

As you can see, this very basic Dockerfile will use a *base image* from ````tiangolo/uwsgi-nginx-flask:python3.6-alpine3.8````.

On top of that base image, it will install [redis-py](https://pypi.org/project/redis/) and then take the contents of the directory ````./azure-vote```` and copy it into the container in the path ````/app````.

To build the docker container image, cd into the right directory, and use the ````az acr build```` command:

````bash
cd application/azure-vote-app
az acr build --image azure-vote-front:v1 --registry <your unique ACR name> --file Dockerfile .
````

## 3.6. Azure Kubernetes Service (AKS)

AKS is the hosted Kubernetes service on Azure.

Kubernetes provides a distributed platform for containerized applications. You build and deploy your own applications and services into a Kubernetes cluster, and let the cluster manage the availability and connectivity. In this step a sample application will be deployed into your own Kubernetes cluster. You will learn how to:

* Create an AKS Kubernetes Cluster
* Connect/validate towards the AKS Cluster
* Update Kubernetes manifest files
* Run an application in Kubernetes
* Test the application

### 3.6.1. Create AKS Cluster

Create an AKS cluster using ````az aks create````. Give the cluster the name  ````k8s````, and run the command:

```azurecli
az aks create --resource-group <resource-group-name> --name k8s --node-count 2 --node-vm-size Standard_D2s_v4 --attach-acr <your unique ACR name>
```

#### note: in the command above, we attach the ACR created previously. This is to allow the AKS cluster to download images from the container registry.

The creation time for the cluster should be around 5 minutes, so this might be a good time for a leg stretcher and/or cup of coffee!

### 3.6.2. Validate towards Kubernetes Cluster

In order to use `kubectl` you need to connect to the Kubernetes cluster, using the following command (which assumes that you have used the naming proposals above):

```azurecli
az aks get-credentials --resource-group <resource-group-name> --name k8s
```

To verify that your cluster is up and running you can try a kubectl command, like ````kubectl get nodes```` which  will show you the nodes (virtual machines) that are active in your cluster. If you followed the instructions, you should see two nodes.

````bash
kubectl get nodes
````


### 3.6.3. Update a Kubernetes manifest file

You have built a docker image with the sample application, in the Azure Container Registry (ACR). To deploy the application to Kubernetes, you must update the image name in the Kubernetes manifest file to include the ACR login server name. Currently the manifest "points" to a container located in the microsoft repository in *docker hub*.

The manifest file to modify is the one that was downloaded when cloning the repository in a previous step. The location of the manifest file is in the ````./k8s/application/azure-vote-app```` directory.

The sample manifest file from the git repo cloned in the first tutorial uses the login server name of *microsoft*. Open this manifest file with a text editor, such as `code`:

```bash
code azure-vote-all-in-one-redis.yaml
```

Replace *microsoft* with your ACR login server name. The following example shows the original content and where you need to replace the **image**.

Original:

```yaml
containers:
- name: azure-vote-front
  image: mcr.microsoft.com/azuredocs/azure-vote-front:v2
```

Provide the ACR login server you created before, so that your manifest file looks like the following example:

```yaml
containers:
- name: azure-vote-front
  image: <your unique ACR name>.azurecr.io/azure-vote-front:v1
```

Please also take some time to study the manifest file, to get a better understanding of what it contains.

Right click Save and then right click Quit.

### 3.6.4. Deploy the application

To deploy your application, use the ```kubectl apply``` command. This command parses the manifest file and creates the needed Kubernetes objects. Specify the sample manifest file, as shown in the following example:

```console
kubectl apply -f azure-vote-all-in-one-redis.yaml
```

When the manifest is applied, a pod and a service is created. The pod contains the "business logic" of your application and the service exposes the application to the internet. This process can take a few minutes, in part because the container image needs to be downloaded from ACR to the Kubernetes Cluster. 

To monitor the progress of the download, you can use ``kubectl get pods`` and ``kubectl describe pod``, like this:

First use ``kubectl get pods`` to find the name of your pod:

```bash
kubectl get pods
```

Then use ``kubectl describe pod`` with the name of your pod:

```bash
kubectl describe pod <pod name>
```

You can also use ``kubectl describe`` to trouble shoot any problems you might have with the deployment (for instance, a common problem is **Error: ErrImagePull**, which can be caused by incorrect credentials or incorrect address/path to the container in ACR. It can also happen if the Kubernetes Cluster does not have read permission in the Azure Container Registry.

Once your container has been pulled and started, showing state **READY**, you can instead start monitoring the **service** to see when a public IP address has been created.

To monitor progress, use the `kubectl get service`. You will probably have to repeats a few times, as it can take a while to get the public IP address.

```console
kubectl get service azure-vote-front
```

The *EXTERNAL-IP* for the *azure-vote-front* service initially appears as *pending*, as shown in the following example:

```bash
azure-vote-front   10.0.34.242   <pending>     80:30676/TCP   7s
```

When the *EXTERNAL-IP* address changes from *pending* to an actual public IP address, the creation of the service is finished. The following example shows a public IP address is now assigned:

```bash
azure-vote-front   10.0.34.242   52.179.23.131   80:30676/TCP   2m
```

To see the application in action, open a web browser to the external IP address.

![Image of Kubernetes cluster on Azure](./media/azure-vote.png)


## 3.6.1 If you have the time - cluster update
Upgrading an AKS cluster is pretty straight forward, in its basic form. If you want to learn more details, feel free to have a look here:

https://learn.microsoft.com/en-us/azure/aks/upgrade-cluster

To show the current kubernetes version in your cluster:

````
az aks show --resource-group <resource-group-name> --name k8s |grep kubernetesVersion
````
### note: there is actually a little bit more to it, because nodepools can have different versions, but for our current case this is good enough.


To updade your cluster to version ````1.24.3```` just run the following command:
````
az aks upgrade --resource-group <resource-group-name> --name k8s --kubernetes-version 1.24.3
````



# 3.7 Secret management in AKS
The exercise below is using standard Kubernetes secrets. This is not recommended from a security point of view, but the idea here is to introduce the concept. There are things you can/should do to increase security later on.

## 3.7.1 create secret

Create files that will be used as input to the secret, and echo some text strings into the files. 

```bash
$ echo -n "user" > ./user.txt
$ echo -n "verysecretpassword" > ./pass.txt
```

Use kubectl create secret to generate a secret using the files just created

```bash
kubectl create secret generic credentials --from-file=./user.txt --from-file=./pass.txt
```

Check to see that your secret has been created

```bash
kubectl get secrets
```


## 3.7.2 Use the secret
For this example, we will insert the secret into an environmentvariable in a pod. For convenience we will continue to use the azure-vote container.

To use the secret in the pod, you need to edit the manifest once again. At the end of the ````Deployment```` section for ````azure-vote-front```` Change the following:
````
      env:
        - name: REDIS
          value: "azure-vote-back"
````

To look like this
````    
      env:
        - name: REDIS
          value: "azure-vote-back"
        - name: USERNAME
          valueFrom:
            secretKeyRef:
              name: credentials
              key: user.txt
        - name: PASSWORD
          valueFrom:
            secretKeyRef:
              name: credentials
              key: pass.txt
````

After this, just re-apply the configuration 

````
kubectl apply -f azure-vote-all-in-one-redis.yaml
````

After a few moments, you should be able to log into the pod and check the environment variables. Use the following commands to ````exec```` into the container.

First get the name of the pod
````
kubectl get pods
````

You should see two pods. One of them will be named something like ````azure-vote-front-d94895c88-p52sr````. This is the pod you want to access.

Use ````kubectl exec <name of the pod> -- sh```` to access the pod:

````
kubectl exec -it azure-vote-front-d94895c88-p52sr -- sh
````

Now, list the environment variables in the pod with the ````printenv```` command

````
printenv
````

This will give you a long list of environment varibles. You should be able to find the username and password variables you created.

To exit the container, just type:
````
exit
````




# 3.8 Storage options in AKS


## 3.8.2  Create volume
Create storage account. Give the storage account a unique name (e.g. use your signum)
````
az storage account create -n <unique name for storage account> -g <resource-group-name> -l westeurope  --sku Standard_LRS
````


Export the connection string as an environment variable, this will be used when creating the Azure file share

````
export AZURE_STORAGE_CONNECTION_STRING=$(az storage account show-connection-string -n <unique name for storage account> -g <resource-group-name> -o tsv)
````

Create the file share

Give the fileshare a useful name, e.g. "aksshare"

````
az storage share create -n <file share name> --connection-string $AZURE_STORAGE_CONNECTION_STRING
````

Get storage account key

````
STORAGE_KEY=$(az storage account keys list --resource-group <resource-group-name> --account-name <unique name for storage account> --query "[0].value" -o tsv)
````

Validate that the environment variable was correctly populated, by echoing the content. 

````
echo  $STORAGE_KEY
````

Create a kubernetes secret to hold the storage account name and key

````
kubectl create secret generic azure-secret --from-literal=azurestorageaccountname=<unique name for storage account> --from-literal=azurestorageaccountkey=$STORAGE_KEY
````
Now you need to edit the manifest once again. You need to add two things, a ````volume```` definition and a ````volumeMount````

For the volumeMount, change the following 

````
      containers:
      - name: azure-vote-front
        image: mcr.microsoft.com/azuredocs/azure-vote-front:v2
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 250m
          limits:
            cpu: 500m

````

to look like below (in other words, add the volumeMount at the end of the ````containers```` section)

````
      containers:
      - name: azure-vote-front
        image: mcr.microsoft.com/azuredocs/azure-vote-front:v2
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 250m
          limits:
            cpu: 500m
        volumeMounts:
        - name: azure
          mountPath: /mnt/azure
````

For the ````volume```` defintion, add the following at the very end of the ````deployment```` section for ````azure-vote-front````. 

## Note: Make sure intentation is correct. YAML is really picky when it comes to that. The ````volume```` statement should be on the same level as ````containers````

````
  volumes:
  - name: azure
    csi:
      driver: file.csi.azure.com
      readOnly: false
      volumeAttributes:
        secretName: azure-secret  # required
        shareName: aksshare  # required
        mountOptions: "dir_mode=0777,file_mode=0777,cache=strict,actimeo=30,nosharesock"  # optional
````

Now its time to apply the manifest again:

````
kubectl apply -f azure-vote-all-in-one-redis.yaml
````

After some time, you can once again ````exec```` into the container to make sure that the volume is where it should be

````
exec -it <pod name> -- sh
ls -l /mnt/azure
````

Another way of checking the mount is to use ````kubectl describe````

````
kubectl describe pod <pod name>
````

In the output you will find (among other things) that you have a mount at /mnt/azure.



## 3.8.3 Bonus exercise
https://learn.microsoft.com/en-us/azure/aks/use-kms-etcd-encryption


# 3.9. Clean-up

Make sure the application is deleted from the cluster (otherwise a later step, which is using Helm, might have issues...)

````bash
kubectl delete -f azure-vote-all-in-one-redis.yaml
````
