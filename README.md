# IBM API Connect Multi-site HA Deployment (Active/Passive)
To achieve high availability in your API Connect deployment, a minimum of three data centers are required. This configuration creates a quorum of data centers, allowing automated failover in any direction, and enabling all three data centers to be active. The quorum majority voting algorithm, allows for a single data center to be offline, and yet still maintain data consistency and availability across the remaining two data centers as they continue to represent a majority in the deployment (avoiding split-brain syndrome). However, when having three data centers is not possible, a two data center deployment provides a disaster recovery solution that has both a low Recovery Time Objective (RTO), and a low Recovery Point Objective (RPO). The following information provides an overview of the concepts and architecture required for configuring a two data center deployment in API Connect.

## High-level notes about the two data center disaster recovery (DR) solution

* Two data center DR is strictly  
    - an Active/Passive deployment for the API Manager and Developer Portal services, 
    - manual failover.
* Two DataPower® gateway subsystems must be deployed to provide high availability for the gateway service. However, this scenario doesn't provide high availability for the analytics service.
* The deployment in each data center is effectively an instance of the same API Connect deployment - therefore, the endpoints, certificates, and Kubernetes secrets must all be the same.
* Each data center must be set up within its own Kubernetes cluster (OpenShift and VMWare are hosted on Kubernetes).
* If high availability is required for the analytics service, two analytics subsystems must be configured, one per gateway subsystem, but with this configuration Developer Portal analytics isn't possible.
* Data consistency is prioritized over data availability.
* Latency between the two data centers must be less than 80ms.
* Replication of the
    - API Manager is asynchronous so, depending on the amount of latency between the two data centers, there is a small possibility that recent updates are not transferred in the event of a failure.
    - Developer Portal is synchronous, and therefore the latency is limited to 80ms or less.
* The API Manager and the Developer Portal services in the two data centers must either all be highly available, or none of them highly available, and the number of nodes on each data center must match.

## Step-By-Step instructions
This article would help you to with step by step installation and configuration instructions to achieve IBM APIConnect multi-site HA Active/Passive also how to test the deployment in case of failure.

### Pre-Requsite

1. Two RedHat OpenShift Cluster v4.9 or later with Storage Provisioned, IngressDomain configured. Pls ensure your OpenShift cluster configured with 16Core x 32GB RAM worker node. Minimum 3 worker nodes in each cluster.
2. HAProxy node which would act as load balancer for APIConnect in both the cluster. 
3. OpenShift Command line tool. `oc`


#### Data Center A - OpenShift Cluster Readiness
Installation and Configurations of OpenShift Cluster is not in the scope of this demo. For this demo I have used IBM TechZone Platform to provision OCP v4.10 with RWO storage provisioned along with Ingress-Domain. Please watch this video if you want to make use of IBM TechZone `Unleash IBM TechZone Platform https://www.youtube.com/watch?v=Ip348en661E` to provision OpenShift Cluster for Demo purposes.
From the existing cluster ensure you have the following details

`Ingress Domain` You can make use of the below command to get the ingress
```
oc get ingresses.config/cluster -o jsonpath={.spec.domain}
```
Provided an example of an output of the above command
`apps.itzocp-110000rrwb-u5n2.selfservice.aws.techzone.ibm.com`

Get the IP address of the IngressDomain of the OCP Cluster in Data Center A. Prepend some dummy name say `"dummy.<ingress domain of the OCP in Data Center A>"`
```
ping dummy.apps.itzocp-110000rrwb-u5n2.selfservice.aws.techzone.ibm.com
```
Have the IP address handy which would be used in HAProxy Configuration.

`StorageClasses` You can make use of the below command to get the StorageClass. Pls check with your OCP admin to get the right classname for `ReadWriteOnce (RWO)` storageclass
```
oc get sc
```



#### Data Center B - OpenShift Cluster Readiness
For checking the rediness, pls make use of the instructions provided in `Data Center A - OpenShift Cluster Readiness` section.


#### Install and Configure HAProxy service
For this demo I have used Ubuntu 20.04 LTS
```
apt-get update
sudo apt show haproxy
sudo apt install -y haproxy=2.4.\*
sudo systemctl status haproxy
sudo systemctl enable haproxy
```

##### Update haproxy.cfg
Edit and Add the detail in `/etc/haproxy/haproxy.cfg` at the EOF. Below provided the sample or eample configuration. 
***Pls read the #INFO in the below example and add the details that is applicable to your environment.***
```
frontend apic-https
 bind *:443
 mode tcp
 option tcplog
 use_backend apic-https

 backend apic-https
 mode tcp
 balance roundrobin
 option ssl-hello-chk
 
 #INFO: Ensure you comment the Passive Site 
 #INFO: Below on is pointing to Data Center A. This IP address (53.53.89.75) is IP address of Ingress Domain in DataCenter A
 server site-a-ingress-endpoint         53.53.89.75:443 check
 
 #INFO: Below on is pointing to Data Center B. This IP address (142.126.108.171) is IP address of Ingress Domain in DataCenter B
 #server site-b-ingress-endpoint         142.126.108.171:443 check
```
##### Update /etc/hosts
Add the entries in /etc/hosts for Ingress Domain of the both the datacenters A & B
```
site-a-ingress-endpoint        53.53.89.75
site-b-ingress-endpoint        142.126.108.171
```
`***NOTE***`
In the Active/Passive deployment, the `HAProxy should contain an entry that should always points to Active Site.` In this example, DataCenter A is an Active Site now. We also have the details of DataCenter B which is commented for future use while testing failover scenario. While we do a failover scenario testing, we need to swap the entry. i.e. Active becomes Passive and vice-versa. 

#### Install and Configure APIConnect in Data Center A

Clone this repository 
```
git clone https://github.com/gowdhamanj/apic-multi-site-HA.git
cd apic-multi-site-HA
```

Using `oc` cli, Login to OpenShift cluster
```
oc login --token=sha256~iZrxDieN3jkEZfzmCwT_2KXBr8hnE3EUBGxV8c35E7U  --server=https://api.itzocp-110000rrwb-u5n2.selfservice.aws.techzone.ibm.com:6443
```
Create CatalogSource & Subscription for APIConnect Operator.
```
oc apply -f CatalogSource.yaml
oc apply -f Subscription.yaml
```
Create a namespace for APIConnect
```
oc new-project apic
```
Have Common Services Operand Request created
```
oc apply -f common-services-operandrequest.yaml
```
###### Obtaining your IBM entitlement API key
You are required to have an entitlement key that provides access to the software components. After the necessary entitlements have been granted, use the following steps to download the entitlement key and apply it to the automation:
    Visit the Container Software Library site - https://myibm.ibm.com/products-services/containerlibrary
    Log in with your IBMId credentials
    Assuming the entitlements are in place, you will be presented with an entitlement key. Click "Copy key".
    
```
export APIC_NAMESPACE=apic
export IBM_ENTITLEMENT_KEY=<paste the entitlement-key copied from previous step>

oc create secret docker-registry ibm-entitlement-key \
    --docker-username=cp \
    --docker-password=$IBM_ENTITLEMENT_KEY \
    --docker-server=cp.icr.io \
    --namespace=$APIC_NAMESPACE
```
Create CertManager for APIC instance
```
oc apply -f Cert-Manager-for-api-instance-in-Datacenter-A.yaml
```
We should create the common Encryption Key for Management & Portal component between both the Site A & B
```
 cat /dev/urandom | head -c63 | base64  > mgmt-enc-key.txt
 oc create secret generic mgmt-encryption-key --from-file=encryption_secret.bin=mgmt-enc-key.txt -n $APIC_NAMESPACE
 cat /dev/urandom | head -c63 | base64 - > ptl-enc-key.txt
 oc create secret generic ptl-encryption-key --from-file=encryption_secret=ptl-enc-key.txt -n $APIC_NAMESPACE
```
While we are working on Data Center A, we should provide IngressDomain of Data Center B. I assume you have IngressDomain name handy which you would have obtained in section `Data Center B - OpenShift Cluster Readiness`

```
export DATA_CENTRE_A_INGRESS_DOMAIN=<pasted the Ingress domain of the Data Center A>
export DATA_CENTER_A_RWO_STORAGECLASS=<paste the RWO storage class in Data Center A>
export DATA_CENTRE_B_INGRESS_DOMAIN=<pasted the Ingress domain of the Data Center B>
#Prove the Ip address of HA Proxy server
export HA_PROXY_IP=140.229.168.60
export HA_MODE=active
```

Update the ingress domain & storgeclass in *apic-instance-in-data-center-A.yaml* 

Execute the below commands
```
export HA_PROXY_IP=`echo $HA_PROXY_IP | sed 's/\./-/g'`
cat apic-instance-in-data-center-A.yaml | sed "s/TO_BE_REPLACED_BY_DATA_CENTER_A_INGRESS_DOMAIN/$DATA_CENTRE_A_INGRESS_DOMAIN/" > A1.yaml
cat A1.yaml | sed "s/TO_BE_REPLACED_BY_DATA_CENTER_B_INGRESS_DOMAIN/$DATA_CENTRE_B_INGRESS_DOMAIN/" > A2.yaml
cat A2.yaml | sed "s/TO_BE_REPLACED_BY_DATA_CENTER_A_RWO_STORAGECLASS/$DATA_CENTER_A_RWO_STORAGECLASS/" > A3.yaml
cat A3.yaml | sed "s/TO_BE_REPLACED_HA_PROXY_IP/$HA_PROXY_IP/" > A4.yaml
cat A4.yaml | sed "s/TO_BE_REPLACED_HA_MODE/$HA_MODE/" > apic-instance-in-data-center-A.yaml
rm -rf A1.yaml , A2.yaml, A3.yaml,  A4.yaml

```

###### Create an instance of apiconnect in Data Center A by executing the below command
```
oc apply -f apic-instance-in-data-center-A.yaml
```
Get the Certificate Authority Issuer Secret from Data Center A.
```
oc get secret apis-dev-ingress-ca -o yaml > ca-issuer-secret.yaml
```
***NOTE:-***
Open ca-issuer-secret.yaml file, remove creationTimestamp, resourceVersion, uid, selfLink and managedFields property and save the file.

#### Install and Configure APIConnect in Data Center B
Using `oc` cli, Login to OpenShift cluster
```
oc login --token=sha256~a8lvYB14bIO7HhjJ7KIb4JgP2eT_G2Pg2mgDT3kEkZs --server=https://c100-e.eu-gb.containers.cloud.ibm.com:32280
```
Create CatalogSource & Subscription for APIConnect Operator.
```
oc apply -f CatalogSource.yaml
oc apply -f Subscription.yaml
```
Create a namespace for APIConnect
```
oc new-project apic
```
Have Common Services Operand Request created
```
oc apply -f common-services-operandrequest.yaml
```

```
export APIC_NAMESPACE=apic
export IBM_ENTITLEMENT_KEY=<paste the entitlement-key copied from  Obtaining your IBM entitlement API key step>

oc create secret docker-registry ibm-entitlement-key \
    --docker-username=cp \
    --docker-password=$IBM_ENTITLEMENT_KEY \
    --docker-server=cp.icr.io \
    --namespace=$APIC_NAMESPACE
```
Create CertManager for APIC instance in DataCenter B
```
oc apply -f Cert-Manager-for-api-instance-in-Datacenter-B.yaml
```
```
oc apply -f ca-issuer-secret.yaml
```
Set the env variable
```
export DATA_CENTRE_B_INGRESS_DOMAIN=<pasted the Ingress domain of the Data Center B>
export DATA_CENTER_B_RWO_STORAGECLASS=<paste the RWO storage class in Data Center B>
export DATA_CENTRE_A_INGRESS_DOMAIN=<pasted the Ingress domain of the Data Center A>
#Prove the Ip address of HA Proxy server
export HA_PROXY_IP=140.229.168.60
export HA_MODE=passive
```
Execute the below commands
```
export HA_PROXY_IP=`echo $HA_PROXY_IP | sed 's/\./-/g'`
cat apic-instance-in-data-center-B.yaml | sed "s/TO_BE_REPLACED_BY_DATA_CENTER_B_INGRESS_DOMAIN/$DATA_CENTRE_B_INGRESS_DOMAIN/" > B1.yaml
cat B1.yaml | sed "s/TO_BE_REPLACED_BY_DATA_CENTER_A_INGRESS_DOMAIN/$DATA_CENTRE_A_INGRESS_DOMAIN/" > B2.yaml
cat B2.yaml | sed "s/TO_BE_REPLACED_BY_DATA_CENTER_B_RWO_STORAGECLASS/$DATA_CENTER_B_RWO_STORAGECLASS/" > B3.yaml
cat B3.yaml | sed "s/TO_BE_REPLACED_HA_PROXY_IP/$HA_PROXY_IP/" > B4.yaml
cat B4.yaml | sed "s/TO_BE_REPLACED_HA_MODE/$HA_MODE/" > apic-instance-in-data-center-B.yaml
rm -rf B1.yaml , B2.yaml, B3.yaml, B4.yaml

```
Create the Management & Portal Encryption Key. The same key that is used in Data Center A
```
 oc create secret generic mgmt-encryption-key --from-file=encryption_secret.bin=mgmt-enc-key.txt -n $APIC_NAMESPACE
 oc create secret generic ptl-encryption-key --from-file=encryption_secret=ptl-enc-key.txt -n $APIC_NAMESPACE
```
###### Create an instance of apiconnect in Data Center B by executing the below command
```
oc apply -f apic-instance-in-data-center-B.yaml
```