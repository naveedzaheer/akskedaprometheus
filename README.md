# Steps to setup a demo for autoscaling Kubernetes workloads with KEDA using Azure Monitor Prometheus metrics

## Setup environment variables
```
SUBSCRIPTION_ID={Your-Azure-Subscription-Id}
RESOURCE_GROUP={Resource-Group-Name}
AKS_NAME={AKS-CLuster-Name}
LOCATION={Azure-Region}
TENANT_ID={Your-AD-TenanatID}
```

## Create the Resource Group
```
az group create -l $LOCATION -n $RESOURCE_GROUP
```

## Create the AKS cluster with KEDA Addon, OIDC issuer, WorkLoad Identity support and App Routing Addon for demo 
```
az aks create --resource-group $RESOURCE_GROUP --name $AKS_NAME --location $LOCATION --enable-workload-identity --enable-oidc-issuer  --enable-keda --enable-app-routing --generate-ssh-keys
```
### Validate the deployment was successful and make sure the cluster has KEDA, workload identity, and OIDC issuer
``` 
az aks show --name $AKS_NAME --resource-group $RESOURCE_GROUP --query "[workloadAutoScalerProfile, securityProfile, oidcIssuerProfile]"
```
### Connect to the Cluster
``` 
az aks get-credentials --name $AKS_NAME --resource-group $RESOURCE_GROUP --overwrite-existing
```

### Once the cluster is created follow these steps to enable Managed Prometheus
Use this link to enable Managed Prometheus [Enable full monitoring with Azure portal](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-monitoring-enable?tabs=cli#enable-full-monitoring-with-azure-portal) 

### Get the link to the Prometheus Query Endpoint from the Azure Monitor Workspace and set environment variable
```
PROMETHEUS_SERVER={Prometheus-Query-Endpoint} 
```

## Deploy the sample application
### Create the AKS namespace
```
kubectl create namespace aks-store
```
### Deploy the sample app
```
kubectl apply -f https://raw.githubusercontent.com/Azure-Samples/aks-store-demo/main/sample-manifests/docs/app-routing/aks-store-deployments-and-services.yaml -n aks-store
```
### Create the Ingress object
Use the ingress.yaml to deploy the ingress object
```
kubectl apply -f ingress.yaml -n aks-store
```
### Verify the managed Ingress was created
```
kubectl get ingress -n aks-store
```
### Deploy the configmap to pod annotation based scraping
```
kubectl apply -f prom-configmap.yaml
```

## Create the Workload Identity Federation
### Create a User Managed Identity
```
MI_NAME={managed-identity-name}

MI_CLIENT_ID=$(az identity create --name $MI_NAME --resource-group $RESOURCE_GROUP --query "clientId" --output tsv)
```

### Get the OIDC issuer URL 
```
AKS_OIDC_ISSUER=$(az aks show --name $AKS_NAME --resource-group $RESOURCE_GROUP --query oidcIssuerProfile.issuerUrl --output tsv)
```

### Create a second federated credential between the managed identity and the namespace and service account used by the keda-operator 
```
FED_KEDA={federated-credential-keda-name}

az identity federated-credential create --name $FED_KEDA --identity-name $MI_NAME --resource-group $RESOURCE_GROUP --issuer $AKS_OIDC_ISSUER --subject system:serviceaccount:kube-system:keda-operator --audience api://AzureADTokenExchange
```

### Give the Managed Idenity 'Monitoring Data Reader' access in Azure Monitor Workspace attached to AKS
1. You can use Azure Portal to access the Azure Monitor Workspace that is attached to AKS cluster
2. Once you are there, you need to go to 'Access Control (IAM)
3. Add a role assignment to assign 'Monitoring Data Reader' role to the Managed Identity


### Enable Workload Identity on KEDA operator 
Restart keda-operator pods
```
kubectl rollout restart deploy keda-operator -n kube-system
```
Confirm keda-operator pods restart
```
kubectl get pod -n kube-system -lapp=keda-operator -w
```

## Create KEDA Scaled objects
### Deploy a KEDA TriggerAuthentication resource that includes the User-Assigned Managed Identity's Client ID
```
kubectl apply -f keda-trigger.yaml -n aks-store
```
### Deploy a KEDA ScaledObject Resource
```
kubectl apply -f keda-scaledobject.yaml -n aks-store
```
