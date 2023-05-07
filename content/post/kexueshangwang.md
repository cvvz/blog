---
title: "科学上网--k8s版"
date: 2023-04-28T23:14:10+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: []
tags: []
---

> 昨天折腾了一下，在自建的aks集群上搭好了出国的机场，用cert-manager自动签发和续期证书、域名动态的解析到service public ip，所以不用担心ip被封禁。。公司每个月150刀的羊毛薅的太爽了🤣
> 
> 原文地址：[https://github.com/cvvz/k8s-playground/tree/master/gost#科学上网-k8s版](https://github.com/cvvz/k8s-playground/tree/master/gost#科学上网-k8s版)


## pre-requisite
1. 购买域名
2. 准备好以下命令行工具: `envsubst`, `cmctl`, `kubectl`, `az`, `helm`

## step 1: 配置DNS

### step 1.1: 创建 azure dns zone

```shell
export AZURE_DEFAULTS_GROUP=your-resource-group
export DOMAIN_NAME=your-domain-name 
az network dns zone create --name $DOMAIN_NAME
```

### step 1.2: 在域名提供商的控制台中设置域名DNS的`NS records`为`Azure authoritative DNS servers`

执行以下命令获取 azure authoritative DNS servers 列表：

```shell
az network dns zone show --name $DOMAIN_NAME --query nameServers -o tsv
```

### step 1.3: 等待ns record 传播完成，可能需要几个小时。

通过以下命令验证是否能成功解析到ns record：

```shell
dig $DOMAIN_NAME ns +trace +nodnssec
```

## step 2: Enable workload identity feature

> cert-manager 需要调用azure api，使用[workload identity](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview)进行鉴权

```shell
az extension add --name aks-preview

az feature register --namespace "Microsoft.ContainerService" --name "EnableWorkloadIdentityPreview"

# 执行以下命令，直到状态变为 Registered：
az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/EnableWorkloadIdentityPreview')].{Name:name,State:properties.state}"

az provider register --namespace Microsoft.ContainerService
```

## step 3： 创建aks集群

```shell
export CLUSTER=your-aks-cluster-name
# region建议选择east asia，香港机房，网络延迟相对更小
export AZURE_DEFAULTS_LOCATION=eastasia

# 创建集群
az aks create -n ${CLUSTER} \
--enable-oidc-issuer \
--enable-workload-identity 

az aks get-credentials -n ${CLUSTER}
```

## step 4: 部署cert-manager

```shell
cat <<EOF > /tmp/values.yaml
podLabels:
  azure.workload.identity/use: "true"
serviceAccount:
  labels:
    azure.workload.identity/use: "true"
EOF

helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade cert-manager jetstack/cert-manager \
    --install \
    --create-namespace \
    --wait \
    --namespace cert-manager \
    --set installCRDs=true \
    --reuse-values \
    --values /tmp/values.yaml
```

## step 5: 为cert-manager配置federated workload identity

> eastasia region不支持 workload identity，参考：[https://learn.microsoft.com/en-us/azure/active-directory/workload-identities/workload-identity-federation-considerations#unsupported-regions-user-assigned-managed-identities](https://learn.microsoft.com/en-us/azure/active-directory/workload-identities/workload-identity-federation-considerations#unsupported-regions-user-assigned-managed-identities)
> 
> 选择一个支持的region（比如japaneast）中创建workload identity

```shell
export USER_ASSIGNED_IDENTITY_NAME=your-cert-manager-identity-name
export IDENTITY_RG=your-identity-group
export IDENTITY_RG_LOCATION=your-identity-group-location

az group create -n ${IDENTITY_RG} -l ${IDENTITY_RG_LOCATION}
az identity create --name "${USER_ASSIGNED_IDENTITY_NAME}" -g ${IDENTITY_RG} -l ${IDENTITY_RG_LOCATION}

export USER_ASSIGNED_IDENTITY_CLIENT_ID=$(az identity show --name "${USER_ASSIGNED_IDENTITY_NAME}" --query 'clientId' -o tsv -g ${IDENTITY_RG})
az role assignment create \
    --role "DNS Zone Contributor" \
    --assignee $USER_ASSIGNED_IDENTITY_CLIENT_ID \
    --scope $(az network dns zone show --name $DOMAIN_NAME -o tsv --query id)

# cert-manager的service account和namespace
export SERVICE_ACCOUNT_NAME=cert-manager 
export SERVICE_ACCOUNT_NAMESPACE=cert-manager 

export SERVICE_ACCOUNT_ISSUER=$(az aks show --name $CLUSTER --query "oidcIssuerProfile.issuerUrl" -o tsv)
az identity federated-credential create \
  --name "cert-manager" \
  --identity-name "${USER_ASSIGNED_IDENTITY_NAME}" \
  --issuer "${SERVICE_ACCOUNT_ISSUER}" \
  --subject "system:serviceaccount:${SERVICE_ACCOUNT_NAMESPACE}:${SERVICE_ACCOUNT_NAME}" \
  -g ${IDENTITY_RG}
```

## step 6: 生成证书

```shell
export GOST_NS=gost
kubectl create ns $GOST_NS

# 创建issuer
export EMAIL_ADDRESS=<email-address> 
export AZURE_SUBSCRIPTION_ID=<your-subscription-id>  
wget https://raw.githubusercontent.com/cvvz/k8s-playground/master/gost/clusterissuer-lets-encrypt.yaml 
envsubst < clusterissuer-lets-encrypt.yaml | kubectl apply -f  -
kubectl describe clusterissuer letsencrypt-production

# 创建Certificate
wget https://raw.githubusercontent.com/cvvz/k8s-playground/master/gost/certificate.yaml 
envsubst < certificate.yaml | kubectl apply -f -

# 验证证书状态
cmctl status certificate www -n $GOST_NS
cmctl inspect secret www-tls -n $GOST_NS
```

## step 7: 部署gost服务

```shell
export AZURE_LOADBALANCER_DNS_LABEL_NAME=lb-$(uuidgen) 
export USER=your-user-name
export PASSWORD=your-password

# deployment
wget https://raw.githubusercontent.com/cvvz/k8s-playground/master/gost/deployment.yaml
envsubst < deployment.yaml | kubectl apply -f -
# service
wget https://raw.githubusercontent.com/cvvz/k8s-playground/master/gost/service.yaml
envsubst < service.yaml | kubectl apply -f -
```

## step 8: 设置dns record

```shell
# 设置www A record
az network dns record-set cname set-record \
    --zone-name $DOMAIN_NAME \
    --cname $AZURE_LOADBALANCER_DNS_LABEL_NAME.$AZURE_DEFAULTS_LOCATION.cloudapp.azure.com \
    --record-set-name www

# 验证可以解析到service external ip
dig www.$DOMAIN_NAME A
```
> 不管pod和service ip怎么变化，只要`$AZURE_LOADBALANCER_DNS_LABEL_NAME`不变，域名始终会解析到service的public ip。
> 
> 所以就算ip被封禁，重新创建一个service生成新的public ip就行了。

## step 9：验证

```shell
curl -v "https://www.google.com" --proxy "https://www.$DOMAIN_NAME" --proxy-user $USER:$PASSWORD
```