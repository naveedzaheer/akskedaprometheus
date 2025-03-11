# Steps to setup a demo for autoscaling Kubernetes workloads with KEDA using Azure Monitor Prometheus metrics

## Setup environment variables
```
SUBSCRIPTION_ID={Your-Azure-Subscription-Id}
RESOURCE_GROUP={Resource-Group-Name}
AKS_NAME={AKS-CLuster-Name}
LOCATION={Azure-Region}
```

## Create the Resource Group
'''
az group create -l $LOCATION -n $RESOURCE_GROUP
'''

## Create the AKS cluster

