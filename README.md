# Go app published in Azure Container Register and managed into Azure Kubernetes Services

This repo contains a sample web service created in GO LANG and show the steps to create and publish into Azure Container Register and managed by Azure Kubernetes Services

### Requirements

1. Azure Subscription
2. Run Powershell Console Admin Mode
3. Docker installed
4. Be an Subscription Admin or with privilegies to interact with resources
5. Azure CLI (Lastest Version)


## Part I - Setup ACR and AKS

You can Skip this part if  you already have ACR and AKS configured in your subscription

### **Setup Resource Group**

First step of all create a blank RG. It requires the AZ CLI already logged in your subscription

```
az group create -l eastus -n MyAKSRG 
```

### **Create a Azure Container Register**

ACR is the private repository where you can host your docker images. In our case, the Hello World Go Web Service

I´m going to use *MyAKSRG* as RG name

``` Powershell
#Create the ACR
az acr create --resource-group MyAKSRG --name MyACR66 --sku Basic --admin-enabled true

#Run this to check if it was created or not
az acr login --name MyACR
```

### **Create the Service Principal Account**

AKS needs permissions to interact with other azure resource. So you will need to create a SP account to accomplish that

***Be aware*** on this step because you will need to save some results after run the command

*- appID is a GUID Format*

*- acrId is a long string with Slashes*


``` Powershell
# 1st. Create the Service Principal - Save the results (appID)
az ad sp create-for-rbac --skip-assignment

#get the ACR ResourceID (save the result to use in acrID)
az acr show --resource-group myResourceGroup --name <acrName> --query "id" --output tsv

#Use the ID Above to grant access to the SP Account
az role assignment create --assignee <appId> --scope <acrId> --role acrpull
```

### **Create the Azure Kubernetes Cluster**

Important STEP, time to create the AKS Cluster. Take a look that you will need some previous data already created before.

If everything ran well, you will see a *Running* message on screen and them a large Response Json

*appID* - The one that you saved from Service Account

*password* - From the Step that you created the Service Account


``` Powershell 
#Create AKS Cluster
az aks create  --resource-group myAKSRG  --name MyAKSCluster  --node-count 1  --service-principal <appId>  --client-secret <password>  --generate-ssh-keys

```

### **Install Kubectl and log into Cluster**

Very quick step, you need to install KubeCtl to manage your cluster and verify the installation

``` Powershell
#Install Kubeclt
az aks install-cli

# Log Into Cluster
az aks get-credentials --resource-group myAKSRG --name MyAKSCluster 
```

With the ACR and AKS ready to receive new containers, now it´s time to deal with application.

---

## Part II - Publish the application

### **Publish the application to ACR**

1. Build and create the image. Don´t forget the POINT at the end of command. Pay attention that you should use your ACR Login Server Name. In this case, I used the one created before *myacr66*


```
docker build -t myacr66.azurecr.io/hellowebservice:latest . 

```
2. If you want to verify, execute the command below and access http://localhost:6060 on your web browser. You should see *Welcome to my website!*

```
docker run -it --rm -p 6060:6060 myacr66.azurecr.io/hellowebservice
```

2. Now, time to push the image to ACR. Execute the command below and verify to switch *myacr66* by your Login Server Name. First you need to TAG image and then PUSH

```powershell
# Switch the ID after tag by using docker images and retrieve the current id of your image
docker tag 02db30eec7b3 myacr66.azurecr.io/app/hellowebservice    
```

Push to ACR
```powershell
#First, Authenticate on ACR
az acr login --name myacr66  

#Push to ACR the image
docker push myacr66.azurecr.io/app/hellowebservice
```

if everything is ok, you can check your image deployed on your private repository. Which is very commom on Enterprise Scenarios where the company does not want its repository on GitHub.

### **Publish on Azure Kubernete Services**

For the next step, AKS needs to retrieve the image from ACR and added it to PODS.

1. Get the Credentials to publish on AKS. Remember to use the Resource Group name and your AKS Cluster name create before. You will receive a message saying that it was merged in a config file.

``` powershell
#get the Credentials
az aks get-credentials -g MyAKSRG -n MyAKSCluster
```

2. Apply using the YAML file on this repository. Execute the following command.
Take a time to understand the yaml file. There are some comments that you should know

``` powershell
#Deploy using YAML
kubectl apply -f HelloWebService.yaml      
```

There are many ways to verify your installation:

a) This will list the pods from your application

```powershell
#List your current pods. If it´s a blank installation, there will be some
kubectl get pods 
```

You can access from your browser. First, discover the IP from your Load balancer (EXTERNAL-IP)

```powershell
#Get the external-ip from your LoadBalancer
kubectl get services
```

Then with this IP, access your application from browser http://<IPFromBalancer>:6060

I hope you liked this article,

Cheers,


### References

https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest

https://docs.microsoft.com/en-us/azure/container-instances/container-instances-tutorial-prepare-acr

https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-cluster

https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-docker-cli


















