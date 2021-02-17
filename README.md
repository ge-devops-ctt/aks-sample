# AKS setup

## Define variables

```bash
LOCATION=eastus2
RG=aks-rg
VNET=aks-vnet
SUBNET=aks-subnet
AKS=aks-cluster
PIP=PIP-VNG-Azure-VNet
VNG=VNG-Azure-VNet
```

$RG="aks-rg"
$VNG="VNG-Azure-VNet"

## Create network resources

### Create VNet and subnets

```bash
az group create -l $LOCATION -n $RG

az network vnet create \
    --resource-group $RG \
    --name $VNET \
    --address-prefix 10.20.0.0/16 \
    --location $LOCATION

az network vnet subnet create \
    --resource-group $RG \
    --vnet-name $VNET \
    --name $SUBNET \
    --address-prefixes 10.20.10.0/24
```

### Create Azure VPN Gateway

```bash
SUBNET_ID=$(az network vnet subnet show --vnet-name=$VNET --name $SUBNET -g=$RG --query id -o tsv)

az network public-ip create \
    --resource-group $RG \
    --name $PIP \
    --allocation-method Dynamic

az network vnet subnet create \
    --resource-group $RG \
    --vnet-name $VNET \
    --name GatewaySubnet \
    --address-prefixes 10.20.255.0/27

az network vnet-gateway create \
    --resource-group $RG \
    --name $VNG \
    --public-ip-address $PIP \
    --vnet $VNET \
    --gateway-type Vpn \
    --vpn-type RouteBased \
    --sku VpnGw1 \
    --no-wait

az network vnet-gateway update \
    --resource-group $RG \
    --name $VNG \
    --address-prefixes 172.16.201.0/24
```

## Configure certificates

### Generate certificates with PowerShell

```powershell
$cert = New-SelfSignedCertificate -Type Custom -KeySpec Signature `
-Subject "CN=P2SRootCert" -KeyExportPolicy Exportable `
-HashAlgorithm sha256 -KeyLength 2048 `
-CertStoreLocation "Cert:\CurrentUser\My" -KeyUsageProperty Sign -KeyUsage CertSign

New-SelfSignedCertificate -Type Custom -DnsName P2SChildCert -KeySpec Signature `
-Subject "CN=P2SChildCert" -KeyExportPolicy Exportable `
-HashAlgorithm sha256 -KeyLength 2048 `
-CertStoreLocation "Cert:\CurrentUser\My" `
-Signer $cert -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.2")
```

### Import CA public key in Azure

- [Export the CA public key with the Windows certificates manager](https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-certificates-point-to-site#cer)
- [Import the CA public key in Azure](https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-howto-point-to-site-resource-manager-portal#uploadfile)

### Client setup

- [Installation the VPN client](https://docs.microsoft.com/en-us/azure/vpn-gateway/point-to-site-vpn-client-configuration-azure-cert)
- [Connect to the Azure VNET](https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-howto-point-to-site-resource-manager-portal#connect)

## Create the AKS

If error  "--vnet-subnet-id is not a valid Azure resource ID.", try to update the CLI running the "az upgrade" command or run the command in the cloud shell.

```bash
az aks create \
    --resource-group $RG \
    --name $AKS \
    --load-balancer-sku standard \
    --network-plugin kubenet \
    --vnet-subnet-id $SUBNET_ID \
    --location $LOCATION \
    --node-count 2
   #  --enable-private-cluster \
   # --docker-bridge-address 172.17.0.1/16 \
   # --dns-service-ip 10.2.0.10 \
   # --service-cidr 10.2.0.0/24
```

## Create a VM (optional)

```bash
az vm create \
    -n MyVm \
    -g $RG \
    --image UbuntuLTS \
    --vnet-name $VNET \
    --subnet $SUBNET \
    --admin-username azureuser \
    --admin-password SuperPassword123! \
    --size Standard_DS1_v2 \
    --public-ip-address "" \
    --no-wait
```

az group delete -n $RG --no-wait

## Connect to the cluster

```bash
az aks get-credentials --resource-group $RG --name $AKS
```

## Install the CodeFresh Runner

- Check the [Codefresh documentation](https://codefresh.io/docs/docs/administration/codefresh-runner/)
- Install the [Codefresh CLI](https://codefresh-io.github.io/cli/installation/download/)
- Create a [Codefresh API key](https://g.codefresh.io/user/settings)

```bash
API_KEY="602ba948ab07033e079b8fc5.f23f44c7c20fccc7ef1ad73174558c9e"
codefresh auth create-context --api-key ${API_KEY}
codefresh runner init --insecure
```
## Create a private load balancer

```bash
helm install nginx-ingress ingress-nginx/ingress-nginx \
    --version 3.23.0 \
    --namespace private-ingress \
    -f ingress/internal-ingress.yaml \
    --create-namespace \
```

## Create test apps

```bash
kubectl apply -f ingress/app1.yaml
kubectl apply -f ingress/app2.yaml
kubectl apply -f ingress/ingress.yaml
```

## Clean resources

```bash
helm uninstall nginx-ingress --namespace private-ingress
kubectl delete ns private-ingress
kubectl delete -f ingress/
```

## Delete AKS

```bash
az aks delete \
    --resource-group $RG \
    --name $AKS \
    --no-wait
```
