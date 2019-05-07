# Provisioning Compute Resources

```shell
rggroup=kuberneteshardfast 

az group create -n $rggroup -l westus2

az network vnet create -g $rggroup \
  -n kubernetes-vnet \
  --address-prefix 10.240.0.0/24 \
  --subnet-name kubernetes-subnet

az network nsg create -g $rggroup -n kubernetes-nsg

az network vnet subnet update -g $rggroup \
  -n kubernetes-subnet \
  --vnet-name kubernetes-vnet \
  --network-security-group kubernetes-nsg

az network nsg rule create -g $rggroup \
  -n kubernetes-allow-ssh \
  --access allow \
  --destination-address-prefix '*' \
  --destination-port-range 22 \
  --direction inbound \
  --nsg-name kubernetes-nsg \
  --protocol tcp \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --priority 1000

az network nsg rule create -g $rggroup \
  -n kubernetes-allow-api-server \
  --access allow \
  --destination-address-prefix '*' \
  --destination-port-range 6443 \
  --direction inbound \
  --nsg-name kubernetes-nsg \
  --protocol tcp \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --priority 1001


az network lb create -g $rggroup \
  -n kubernetes-lb \
  --backend-pool-name kubernetes-lb-pool \
  --public-ip-address kubernetes-pip \
  --public-ip-address-allocation static


az network nsg rule list -g $rggroup --nsg-name kubernetes-nsg --query "[].{Name:name, \
  Direction:direction, Priority:priority, Port:destinationPortRange}" -o table

az network public-ip  list --query="[?name=='kubernetes-pip'].{ResourceGroup:resourceGroup, \
  Region:location,Allocation:publicIpAllocationMethod,IP:ipAddress}" -o table


ubuVer=$(az vm image list --location westus2 --publisher Canonical --offer UbuntuServer --sku 18.04-LTS --all -o table | tail -n 1 | awk '{print $4}')

az vm availability-set create -g $rggroup -n controller-as
for i in 0 1 2; do
    echo "[Controller ${i}] Creating public IP..."
    az network public-ip create -n controller-${i}-pip -g $rggroup > /dev/null

    echo "[Controller ${i}] Creating NIC..."
    az network nic create -g $rggroup \
        -n controller-${i}-nic \
        --private-ip-address 10.240.0.1${i} \
        --public-ip-address controller-${i}-pip \
        --vnet kubernetes-vnet \
        --subnet kubernetes-subnet \
        --ip-forwarding \
        --lb-name kubernetes-lb \
        --lb-address-pools kubernetes-lb-pool > /dev/null

    echo "[Controller ${i}] Creating VM..."
    az vm create -g $rggroup \
        -n controller-${i} \
        --image $ubuVer \
        --generate-ssh-keys \
        --nics controller-${i}-nic \
        --availability-set controller-as \
        --nsg '' \
        --admin-username 'kuberoot' > /dev/null
done

az vm availability-set create -g $rggroup -n worker-as
for i in 0 1 2; do
    echo "[Worker ${i}] Creating public IP..."
    az network public-ip create -n worker-${i}-pip -g $rggroup > /dev/null

    echo "[Worker ${i}] Creating NIC..."
    az network nic create -g $rggroup \
        -n worker-${i}-nic \
        --private-ip-address 10.240.0.2${i} \
        --public-ip-address worker-${i}-pip \
        --vnet kubernetes-vnet \
        --subnet kubernetes-subnet \
        --ip-forwarding > /dev/null

    echo "[Worker ${i}] Creating VM..."
    az vm create -g $rggroup \
        -n worker-${i} \
        --image ${UBUNTULTS} \
        --nics worker-${i}-nic \
        --tags pod-cidr=10.200.${i}.0/24 \
        --availability-set worker-as \
        --nsg '' \
        --admin-username 'kuberoot' > /dev/null
done
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
