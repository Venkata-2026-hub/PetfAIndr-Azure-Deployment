# PetfAIndr Azure Deployment Commands

This file provides an end-to-end command guide for deploying the **PetfAIndr** assignment to Microsoft Azure, based on the assignment specification and presentation.

## Assumptions

- You already have the assignment source code locally (including `config.bicep`, `app.bicep`, `app/backend.bicep`, `app/frontend.bicep`, and the `DAPR/` YAML files).
- You are using a **Windows VM** with **Docker Desktop**, **Azure CLI**, **kubectl**, and optionally **Helm** installed.
- Replace every placeholder wrapped in `<...>` before running commands.
- The assignment document recommends:
  - **Sweden Central** for the AKS/application environment
  - a **different region** for the Windows developer VM
  - **2 AKS nodes** of **Standard_B2s** for the student subscription deployment
  - `--attach-acr` when creating AKS
  - Cosmos DB (`petfaindr` / `pets` / `/partitionKey`)
  - Storage container `images`
  - Service Bus topics `foundpet` and `lostpet`
  - Dapr components `pets`, `images`, and `pubsub`

---

## 0) Set variables

### PowerShell
```powershell
$SUBSCRIPTION_ID = "<your-subscription-id>"
$RG = "rg-petfaindr"
$LOCATION = "swedencentral"
$DEV_VM_LOCATION = "switzerlandnorth"

$AKS_NAME = "aks-petfaindr"
$ACR_NAME = "<globally-unique-acr-name>"
$COSMOS_ACCOUNT = "<globally-unique-cosmos-name>"
$STORAGE_ACCOUNT = "<globally-unique-storage-name>"
$SB_NAMESPACE = "<globally-unique-sb-name>"
$CUSTOMVISION_TRAIN = "<globally-unique-cv-train-name>"
$CUSTOMVISION_PRED = "<globally-unique-cv-pred-name>"

$VNET_NAME = "vnet-petfaindr"
$SUBNET_NAME = "snet-petfaindr"
$VM_NAME = "vm-petfaindr-dev"
$ADMIN_USER = "azureuser"

$FRONTEND_IMAGE = "frontend"
$BACKEND_IMAGE = "backend"
$IMAGE_TAG = "v1"
```

---

## 1) Login and select subscription

```powershell
az login
az account set --subscription $SUBSCRIPTION_ID
az account show --output table
```

---

## 2) Create resource group

```powershell
az group create `
  --name $RG `
  --location $LOCATION
```

---

## 3) Optional: create a Windows developer VM in another region

> The assignment says the Windows machine used for Docker should be in a region **other than Sweden Central**.

```powershell
az group create `
  --name "$RG-dev" `
  --location $DEV_VM_LOCATION

az network vnet create `
  --resource-group "$RG-dev" `
  --name "vnet-petfaindr-dev" `
  --address-prefix 10.10.0.0/16 `
  --subnet-name "default" `
  --subnet-prefix 10.10.0.0/24

az vm create `
  --resource-group "$RG-dev" `
  --name $VM_NAME `
  --image "MicrosoftWindowsDesktop:windows-11:win11-23h2-pro:latest" `
  --size "Standard_D4s_v5" `
  --admin-username $ADMIN_USER `
  --admin-password "<StrongPasswordHere!>" `
  --vnet-name "vnet-petfaindr-dev" `
  --subnet "default" `
  --public-ip-sku Standard
```

Open RDP:
```powershell
az vm show -d `
  --resource-group "$RG-dev" `
  --name $VM_NAME `
  --query publicIps `
  --output tsv
```

---

## 4) Create Azure Container Registry (ACR)

```powershell
az acr create `
  --resource-group $RG `
  --name $ACR_NAME `
  --sku Basic `
  --admin-enabled true
```

Check login server:
```powershell
az acr show `
  --resource-group $RG `
  --name $ACR_NAME `
  --query loginServer `
  --output tsv
```

Login to ACR:
```powershell
az acr login --name $ACR_NAME
```

---

## 5) Create Azure Cosmos DB (NoSQL)

```powershell
az cosmosdb create `
  --name $COSMOS_ACCOUNT `
  --resource-group $RG `
  --locations regionName=$LOCATION failoverPriority=0 isZoneRedundant=False
```

Create database:
```powershell
az cosmosdb sql database create `
  --account-name $COSMOS_ACCOUNT `
  --resource-group $RG `
  --name "petfaindr"
```

Create container:
```powershell
az cosmosdb sql container create `
  --account-name $COSMOS_ACCOUNT `
  --resource-group $RG `
  --database-name "petfaindr" `
  --name "pets" `
  --partition-key-path "/partitionKey" `
  --throughput 400
```

Get Cosmos connection values:
```powershell
az cosmosdb keys list `
  --name $COSMOS_ACCOUNT `
  --resource-group $RG `
  --type keys
```

---

## 6) Create Storage Account and blob container

Create storage account with blob public access enabled:
```powershell
az storage account create `
  --name $STORAGE_ACCOUNT `
  --resource-group $RG `
  --location $LOCATION `
  --sku Standard_LRS `
  --kind StorageV2 `
  --allow-blob-public-access true
```

Get storage key:
```powershell
$STORAGE_KEY = az storage account keys list `
  --resource-group $RG `
  --account-name $STORAGE_ACCOUNT `
  --query "[0].value" `
  --output tsv
```

Create blob container `images` with anonymous access:
```powershell
az storage container create `
  --name "images" `
  --account-name $STORAGE_ACCOUNT `
  --account-key $STORAGE_KEY `
  --public-access blob
```

---

## 7) Create Azure Service Bus namespace and topics

```powershell
az servicebus namespace create `
  --resource-group $RG `
  --name $SB_NAMESPACE `
  --location $LOCATION `
  --sku Standard
```

Create topics exactly as required:
```powershell
az servicebus topic create `
  --resource-group $RG `
  --namespace-name $SB_NAMESPACE `
  --name "foundpet"

az servicebus topic create `
  --resource-group $RG `
  --namespace-name $SB_NAMESPACE `
  --name "lostpet"
```

Create shared access policy `Dapr` with Manage rights:
```powershell
az servicebus namespace authorization-rule create `
  --resource-group $RG `
  --namespace-name $SB_NAMESPACE `
  --name "Dapr" `
  --rights Listen Send Manage
```

Get Service Bus connection string:
```powershell
az servicebus namespace authorization-rule keys list `
  --resource-group $RG `
  --namespace-name $SB_NAMESPACE `
  --name "Dapr"
```

---

## 8) Create Azure Custom Vision resources

Training resource:
```powershell
az cognitiveservices account create `
  --name $CUSTOMVISION_TRAIN `
  --resource-group $RG `
  --kind CustomVision.Training `
  --sku F0 `
  --location $LOCATION `
  --yes
```

Prediction resource:
```powershell
az cognitiveservices account create `
  --name $CUSTOMVISION_PRED `
  --resource-group $RG `
  --kind CustomVision.Prediction `
  --sku F0 `
  --location $LOCATION `
  --yes
```

Get keys and endpoints:
```powershell
az cognitiveservices account keys list `
  --name $CUSTOMVISION_TRAIN `
  --resource-group $RG

az cognitiveservices account show `
  --name $CUSTOMVISION_TRAIN `
  --resource-group $RG `
  --query properties.endpoint `
  --output tsv

az cognitiveservices account keys list `
  --name $CUSTOMVISION_PRED `
  --resource-group $RG

az cognitiveservices account show `
  --name $CUSTOMVISION_PRED `
  --resource-group $RG `
  --query properties.endpoint `
  --output tsv
```

> Then create the Custom Vision project in the **Custom Vision portal**, upload the images from the `Dummy-Tag-Images` folder, and tag them as `Dummy-Tag`. You will also need the **project ID** from the portal.

---

## 9) Build and push container images to ACR

Get ACR login server:
```powershell
$ACR_LOGIN_SERVER = az acr show `
  --resource-group $RG `
  --name $ACR_NAME `
  --query loginServer `
  --output tsv
```

Build frontend image:
```powershell
docker build -t "$ACR_LOGIN_SERVER/$FRONTEND_IMAGE:$IMAGE_TAG" .\frontend
```

Build backend image:
```powershell
docker build -t "$ACR_LOGIN_SERVER/$BACKEND_IMAGE:$IMAGE_TAG" .\backend
```

Push images:
```powershell
docker push "$ACR_LOGIN_SERVER/$FRONTEND_IMAGE:$IMAGE_TAG"
docker push "$ACR_LOGIN_SERVER/$BACKEND_IMAGE:$IMAGE_TAG"
```

List repositories:
```powershell
az acr repository list `
  --name $ACR_NAME `
  --output table
```

List tags:
```powershell
az acr repository show-tags `
  --name $ACR_NAME `
  --repository $FRONTEND_IMAGE `
  --output table

az acr repository show-tags `
  --name $ACR_NAME `
  --repository $BACKEND_IMAGE `
  --output table
```

---

## 10) Create AKS cluster and attach ACR

> The assignment requires `--attach-acr <yourACRName>`.

```powershell
az aks create `
  --resource-group $RG `
  --name $AKS_NAME `
  --location $LOCATION `
  --node-count 2 `
  --node-vm-size Standard_B2s `
  --generate-ssh-keys `
  --attach-acr $ACR_NAME
```

Get AKS credentials:
```powershell
az aks get-credentials `
  --resource-group $RG `
  --name $AKS_NAME `
  --overwrite-existing
```

Verify cluster:
```powershell
kubectl get nodes -o wide
```

---

## 11) Install Dapr on AKS

### Option A - AKS Dapr extension (recommended current Azure approach)

Register extension features if needed:
```powershell
az extension add --name k8s-extension
az extension add --name aks-preview
az provider register --namespace Microsoft.KubernetesConfiguration
az provider register --namespace Microsoft.ContainerService
```

Install Dapr extension:
```powershell
az k8s-extension create `
  --cluster-type managedClusters `
  --cluster-name $AKS_NAME `
  --resource-group $RG `
  --name dapr `
  --extension-type Microsoft.Dapr `
  --scope cluster
```

Check extension:
```powershell
az k8s-extension show `
  --cluster-type managedClusters `
  --cluster-name $AKS_NAME `
  --resource-group $RG `
  --name dapr
```

### Option B - Dapr CLI (only use if your course materials explicitly require manual runtime install)

Install Dapr CLI:
```powershell
winget install Dapr.CLI
```

Initialize Dapr on the connected Kubernetes cluster:
```powershell
dapr init -k
```

Check Dapr status:
```powershell
dapr status -k
```

> Do **not** mix both approaches in the same cluster unless you are intentionally managing that complexity.

---

## 12) Apply the Dapr component YAML files

> The assignment says the three YAML files in the `DAPR` folder must be applied **before** application deployment.

```powershell
kubectl apply -f .\DAPR\pets.yaml
kubectl apply -f .\DAPR\images.yaml
kubectl apply -f .\DAPR\pubsub.yaml
```

Verify:
```powershell
kubectl get components
```

---

## 13) Prepare Kubernetes secrets using `config.bicep`

Edit `config.bicep` first and fill in:
- Cosmos DB endpoint/key
- Storage account name/key
- Service Bus connection string
- Custom Vision training endpoint/key
- Custom Vision prediction endpoint/key
- Custom Vision project ID
- any other values marked as placeholders in the file

Deploy the secret template:
```powershell
az deployment group create `
  --resource-group $RG `
  --template-file .\config.bicep `
  --parameters aksName=$AKS_NAME `
               cosmosAccountName=$COSMOS_ACCOUNT `
               storageAccountName=$STORAGE_ACCOUNT `
               serviceBusNamespaceName=$SB_NAMESPACE
```

Check created secrets:
```powershell
kubectl get secrets
```

---

## 14) Deploy the application

> The assignment mentions using `app.bicep`, and that image names in ACR must match those referenced in `./app/backend.bicep` and `./app/frontend.bicep`.

Deploy:
```powershell
az deployment group create `
  --resource-group $RG `
  --template-file .\app.bicep `
  --parameters aksName=$AKS_NAME `
               acrName=$ACR_NAME
```

Check deployments, pods, and services:
```powershell
kubectl get deployments
kubectl get pods -o wide
kubectl get svc
kubectl get ingress
```

Watch pod startup:
```powershell
kubectl get pods -w
```

If you see `ImagePullBackOff`, confirm:
1. the image names in ACR are exactly the same as in the Bicep files  
2. AKS was created with `--attach-acr`  
3. the tags you pushed are the tags referenced in the manifests/Bicep

---

## 15) Find the public URL / public IP

If using ingress:
```powershell
kubectl get ingress
```

If using a LoadBalancer service:
```powershell
kubectl get svc
```

Open the app:
```powershell
start "http://<external-ip-or-hostname>"
```

---

## 16) Validate the application

Useful checks:
```powershell
kubectl get pods
kubectl get svc
kubectl get ingress
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>
```

Backend logs:
```powershell
kubectl get pods
kubectl logs <backend-pod-name>
```

Restart a deployment:
```powershell
kubectl rollout restart deployment <deployment-name>
```

Check rollout:
```powershell
kubectl rollout status deployment <deployment-name>
```

---

## 17) Train the AI model

This part is mostly done in the running app and Custom Vision portal:

1. Open the PetfAIndr app  
2. Submit **one dog at a time**  
3. Upload the **5 images** for that dog  
4. Wait about **12 minutes** for training  
5. Confirm that a **published iteration** exists in Custom Vision  
6. Validate through the **found pet** page

---

## 18) Scale the application

Scale deployments:
```powershell
kubectl scale deployment frontend --replicas=2
kubectl scale deployment backend --replicas=2
```

Confirm:
```powershell
kubectl get pods
```

---

## 19) Stop / start AKS to save cost

Stop:
```powershell
az aks stop `
  --name $AKS_NAME `
  --resource-group $RG
```

Start:
```powershell
az aks start `
  --name $AKS_NAME `
  --resource-group $RG
```

---

## 20) Delete everything when finished

Delete app resource group:
```powershell
az group delete `
  --name $RG `
  --yes `
  --no-wait
```

Delete dev VM resource group:
```powershell
az group delete `
  --name "$RG-dev" `
  --yes `
  --no-wait
```

---

## 21) Quick troubleshooting checklist

### Quota / region issues
```powershell
az vm list-usage `
  --location $LOCATION `
  --output table
```

### ACR connection
```powershell
az aks check-acr `
  --resource-group $RG `
  --name $AKS_NAME `
  --acr $ACR_NAME.azurecr.io
```

### Pod details
```powershell
kubectl describe pod <pod-name>
```

### Events
```powershell
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Dapr components
```powershell
kubectl get components
kubectl get pods -A
```

---

## 22) Suggested repository structure

```text
PetfAIndr/
├─ README.md
├─ COMMANDS.md
├─ config.bicep
├─ app.bicep
├─ app/
│  ├─ frontend.bicep
│  └─ backend.bicep
├─ DAPR/
│  ├─ pets.yaml
│  ├─ images.yaml
│  └─ pubsub.yaml
├─ frontend/
│  └─ Dockerfile
└─ backend/
   └─ Dockerfile
```

---

## Notes

- Replace all placeholder values before execution.
- If your course package uses different parameter names in `config.bicep` or `app.bicep`, adapt the `az deployment group create` commands to those exact parameter names.
- The assignment document says the **student deployment** should use **2 × Standard_B2s** for AKS because of quota constraints, but this sizing is **not** intended for the final production cost calculation.
