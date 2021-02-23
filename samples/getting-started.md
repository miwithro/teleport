# AKS & Teleport - Getting Started

## Comparing With and Without Teleport

To get a sense of the performance benefits of Teleport two deployments will be made.

Teleport is node specific. If an image is pulled to a node, with teleport enabled, the expanded layers are mounted.
If a second copy of the same image is pulled to the same node, even if pulled from a non-teleport expanded repository, the node will identify the image is the same and mount the layers from the previously pulled and teleport expanded image.

To test the same image with and without Teleport enabled, two nodepools will be created.

## Set environment variables

Configure variables unique to your environment. 

```azurecli-interactive
AKS_RG=teleport-rg
AKS=teleport
RG_LOCATION=westus2
REGISTRY=teleport
REGISTRY_URL=${REGISTRY}.azurecr.io
```

## Create an AKS Cluster

At the current moment, Teleport does not yet support managed identity access to teleport expanded layers. Until managed identity is supported, configure the cluster with a service principal.

```azurecli-interactive
SP_PWD=$(az ad sp create-for-rbac --skip-assignment --name ${AKS}-sp --query password -o tsv)
az aks create \
    -g ${AKS_RG} \
    -n ${AKS} \
    --attach-acr $REGISTRY \
    --kubernetes-version 1.19.7 \
    -l $RG_LOCATION \
    --service-principal $(az ad sp show --id http://${AKS}-sp --query appId -o tsv) \
    --client-secret $SP_PWD \
    --aks-custom-headers EnableACRTeleport=true
```

## Import an image for teleportation

You may use any image. For completeness of this walkthrough, the `mcr.microsoft.com/azuredocs/azure-vote-front:v1` image is used.

https://mcr.microsoft.com/v2/azuredocs/azure-vote-front/tags/list

```azurecli-interactive
az acr import \
  --source mcr.microsoft.com/azuredocs/azure-vote-front:v1 \
  --name $REGISTRY \
  --image azure-vote-front:v1
```

## Confirm import and teleport expansion

```azurecli
az acr repository show \
  --repository azure-vote-front \
  -o jsonc
```

Look for `"teleportEnabled": true,` in the output

```json
{
  "changeableAttributes": {
    "deleteEnabled": true,
    "listEnabled": true,
    "readEnabled": true,
    "teleportEnabled": true,
    "writeEnabled": true
  }
```

## Check layer expansion

Although the repository is configured for teleport expansion, each image upload will take time to be expanded on push. The length of time is based on the number and size of layers.

To confirm layers have been expanded, at this point in the Teleport preview, the `check-expansion.sh` script may be used.
As the script uses a `/mount` api, basic auth is required. An [ACR Token](https://aka.ms/acr/tokens) is created and saved as environment variable.

```azurecli-interactive
export ACR_USER=teleport-token
export ACR_PWD=$(az acr token create \
  --name teleport-token \
  --registry $REGISTRY \
  --repository azure-vote-front \
  content/read \
  --query credentials.passwords[0].value -o tsv)

../check-expansion.sh teleport azure-vote-front v1
```

## Deploy to AKS

Update the `wordpress.yaml` file to reference your registry name:

```yml
    spec:
      nodeSelector:
        "beta.kubernetes.io/os": linux
      containers:
      - name: wordpress
        image: <registryName>.azurecr.io/azure-vote-front:v1
```

Deploy the kube manifest:

```azurecli-interactive
kubectl apply -f vote.yaml
```

Get the list of pods to find the wordpress pod. The shorthand version can be used if only one pod is named `wordpress`:

```azurecli-interactive
kubectl get pods
kubectl describe pod azure-vote-front
```

Under the `events` list, an entry for `Successfully pulled image...` provides the pull time, which would include expansion for non-teleported images. For teleport, this time should be considerably faster.

```bash
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  5s    default-scheduler  Successfully assigned default/wordpress-5fb5767ddc-lz6t2 to aks-nodepool1-10583637-vmss000001
  Normal  Pulling    4s    kubelet            Pulling image "teleport.azurecr.io/wordpress:5"
  Normal  Pulled     0s    kubelet            Successfully pulled image "teleport.azurecr.io/wordpress:5" in 4.460727264s
```

## Create a non-teleport enabled pod for comparison

```bash
kubectl apply -f https://raw.githubusercontent.com/andyzhangx/demo/master/dev/docker-image-cleanup.yaml
```


- [Create and manage multiple node pools for a cluster in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools)
- [Provide dedicated nodes using taints and tolerations](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-advanced-scheduler#provide-dedicated-nodes-using-taints-and-tolerations)

**Note:** https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/

```azurecli
az aks nodepool add \
    --resource-group $AKS_RG \
    --cluster-name $AKS \
    --kubernetes-version 1.19.7 \
    --name nonteleport \
    --node-count 1
```

```azurecli
az aks nodepool add \
    --resource-group $AKS_RG \
    --cluster-name $AKS \
    --kubernetes-version 1.19.7 \
    --name nonteleport \
    --node-count 1 \
    --aks-custom-headers EnableACRTeleport=true
```

```azurecli
az aks nodepool list \
    --resource-group $AKS_RG \
    --cluster-name $AKS \
    -o jsonc
```

Note a few properties to assure like configurations

The following should only be available on the teleport enabled nodepools. The second non-teleport nodepool should not have this label.
```azurecli
 "nodeLabels": {
      "kubernetes.azure.com/enable-acr-teleport-plugin": "true"
    },
```

az aks create -g teleport-rg -n teleport --attach-acr teleport --kubernetes-version 1.19.3 -l westus2 --aks-custom-headers EnableACRTeleport=true

az aks get-credentials -g teleport-rg -n teleport
kubectl get nodes

kubectl describe node aks-nodepool1-10583637-vmss000000

### Scratch/Notes

Both nodepools should have the same `vmSize`

```azurecli
    "vmSize": "Standard_DS2_v2",
```

Taints

```azurecli-interactive
az aks nodepool add \
    --resource-group myResourceGroup \
    --cluster-name myAKSCluster \
    --name taintnp \
    --node-count 1 \
    --node-taints sku=gpu:NoSchedule \
    --no-wait
```
