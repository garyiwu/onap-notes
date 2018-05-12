# onap-oom-deploy
This is a step-by-step installation guide for deploying ONAP Amsterdam using OOM, including any workarounds I encountered.

## Prerequisites
This installation guide assumes the following:
* Installation of Ubuntu OpenStack Ocata per https://docs.openstack.org/ocata/install-guide-ubuntu/.
  * All the API endpoints should be specified with numeric IP addresses to avoid issues with DNS resolution of "controller" as the host name.
* Installation of Heat per https://docs.openstack.org/project-install-guide/orchestration/ocata/install-ubuntu.html
* Installation of Designate per https://docs.openstack.org/project-install-guide/dns/ocata/install-ubuntu.html
  * There is a bug in the installation guide.  The target type in `/etc/designate/pools.yaml` needs to be "bind9" instead of "bind".
* A separate VM or system that has Rancher server 1.6.10 installed per https://wiki.onap.org/display/DW/OOM+Infrastructure+setup+-+Rancher.  

## Install Kubernetes
These are the steps to create a Kubernetes VM with 64 GB RAM running on Ubuntu 16.04.3 LTS, docker 1.12.6, rancher 1.6.10, kubernetes 1.8.6, and helm 2.3.

Launch k8s instance, Ubuntu 16.04 LTS with 64 GB RAM, selfservice network.  Associate floating IP.

Run everything as root.

Add hostname to `/etc/hosts`:
```sh
echo `hostname -I` `hostname` >> /etc/hosts
```

Add apt proxy definition (we have a local apt proxy):
```sh
cat > /etc/apt/apt.conf.d/90curtin-aptproxy <<EOF
Acquire::http::Proxy "http://apt-proxy.neo.futurewei.com:3142";
Acquire::https::Proxy "DIRECT";
EOF
apt update
```

Install docker 1.12.x:
```sh
curl https://releases.rancher.com/install-docker/1.12.sh | sh
```
Add docker insecure registry (we have a local docker proxy):
```sh
cat > /etc/docker/daemon.json <<EOF
{
    "insecure-registries" : [ "docker-proxy.neo.futurewei.com:5000" ]
}
EOF
service docker restart
```
Install kubernetes 1.8.6:
```sh
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.8.6/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
mkdir ~/.kube
```
Install helm 2.3:
```sh
wget http://storage.googleapis.com/kubernetes-helm/helm-v2.3.0-linux-amd64.tar.gz
tar -zxvf helm-v2.3.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
```
Fix virtual memory allocation for onap-log:elasticsearch:
```sh
sysctl -w vm.max_map_count=262144
```

## Deploy OOM
Clone OOM:
```sh
cd ~
git clone -b amsterdam http://gerrit.onap.org/r/oom
```
Update `values.yaml` to point to docker-proxy instead of nexus3:
```sh
cd ~/oom/kubernetes
perl -p -i -e 's/nexus3.onap.org:10001/docker-proxy.neo.futurewei.com:5000/g' `find ./ -name values.yaml`
```
Pre-pull docker images:
```sh
cd ~
wget https://jira.onap.org/secure/attachment/10750/prepull_docker.sh
chmod +x prepull_docker.sh
sed -i -e 's/ &//g' prepull_docker.sh
./prepull_docker.sh
```
Run rancher agent by following the Add Host instructions from the Rancher server UI.

Wait for kubernetes to fully initialize (all the host containers show green in Rancher UI).

Copy and paste `~/.kube/config` from Rancher UI > Kubernetes > CLI.

Update `~/oom/kubernetes/kube2msb/values.yaml` kubeMasterAuthToken to use the token from `~/.kube/config`.

Put your onap_key ssh private key in `~/.ssh/onap_key`.

Import your keypair "onap_key" into OpenStack as the user "demo".

Create a DNS zone called "dcaeg2.onap.org" in the demo tenant.  From wherever you have openstack CLI installed:
```sh
source demo-openrc
openstack zone create --email "gary.i.wu@huawei.com" dcaeg2.onap.org.
```

Create or edit `~/oom/kubernetes/config/onap-parameters.yaml`:
```
OPENSTACK_UBUNTU_14_IMAGE: "trusty"
OPENSTACK_PUBLIC_NET_ID: "3a6247f1-fac6-4167-a49f-33cc8415ccf4"
OPENSTACK_OAM_NETWORK_ID: "fe04c13b-3221-4a63-b8b8-695e92a12056"
OPENSTACK_OAM_SUBNET_ID: "1cf4bd0a-eb1c-4cdc-a8e8-532ac62a4a33"
OPENSTACK_OAM_NETWORK_CIDR: "172.16.1.0/24"
OPENSTACK_USERNAME: "demo"
OPENSTACK_API_KEY: "demo"
OPENSTACK_TENANT_NAME: "demo"
OPENSTACK_TENANT_ID: "0662614435ce404293ed61cad6fe981c"
OPENSTACK_REGION: "RegionOne"
OPENSTACK_KEYSTONE_URL: "http://10.145.122.117:5000"
OPENSTACK_FLAVOUR_MEDIUM: "m1.medium"
OPENSTACK_SERVICE_TENANT_NAME: "service"
DMAAP_TOPIC: "AUTO"
DEMO_ARTIFACTS_VERSION: "1.1.0-SNAPSHOT"
```

Create or edit `~/oom/kubernetes/dcaegen2/dcae-parameters.yaml`:
```
# UUID of the OpenStack"s CentOS 7 cloud VM image
# If your Openstack does not have a CentOS 7 cloud image, you will have to add one.
centos7image_id: '3b9bcad0-633f-4250-993c-02275946fb9a'
# UUID of the OpenStack"s Ubuntu 16.04 VM image
# If your Openstack does not have a buntu 16.04 cloud image, you will have to add one.
ubuntu1604image_id: '3bfa78a3-d50e-48a7-89c5-a4464a839873'
# ID of the OpenStack"s VM flavor to be used by DCAEGEN2 VMs (m1.medium/m1.large)
flavor_id: '4'
# UUID of the OpenStack"s security group to be used for DCAEGEN2 VMs
security_group: '8c9e2beb-bb02-4a7b-97b7-9ce5cbbf6119'
# The name of the OpenStack network where public IP addresses and floating IPs are allocated from
# This must use the name and not the UUID.  The name must be unique otherwise the installer fails
public_net: 'provider'
# The name of the OpenStack network where private IP addresses are allocated from
private_net: 'selfservice'
# Group header for OpenStack Keystone parameters
openstack:
  # User name
  username: 'demo'
  # Password
  password: 'demo'
  # Name of the OpenStack tenant/project where DCAEGEN2 VMs are deployed
  tenant_name: 'demo'
  # Openstack authentication API URL, for example 'https://horizon.playground.onap.org:5000/v2.0'
  auth_url: 'http://10.145.122.117:5000/v2.0'
  # Name of the OpenStack region where DCAEGEN2 VMs are deployed, for example 'RegionOne'
  region: 'RegionOne'
# Name of the public key uploaded to OpenStack in the Prepration step
keypair: 'onap_key'
# Path to the private key within the conatiner (!! Do not change!!)
key_filename: '/opt/dcae/key'
# Prefix (location code) of all DCAEGEN2 VMs
location_prefix: 'onap-dcaeg2-'
# Domain name of the OpenStack tenant 'onapr1.playground.onap.org'
location_domain: 'dcaeg2.onap.org'
# Location of the raw artifact repo hosting additional boot scripts called by DCAEGEN2 VMs" cloud-init, for example: 'https://nexus.onap.org/service/local/repositories/raw/content'
codesource_url: 'https://nexus.onap.org/content/sites/raw'
# Path to the boot scripts within the raw artifact repo, for example: 'org.onap.dcaegen2.deployments.scripts/releases/'
codesource_version: 'org.onap.dcaegen2.deployments/releases/scripts/'
```

Do one-time configuration initialization:
```sh
# Update your setenv.env to point to your onap_key and dcae-parameters.yaml
OPENSTACK_PRIVATE_KEY_PATH=~/.ssh/onap_key
DCAEGEN2_CONFIG_INPUT_FILE_PATH=../dcaegen2/dcae-parameters.yaml

# Source the environment file:
cd ~/oom/kubernetes/oneclick/
source setenv.bash

# run the config pod creation
cd ~/oom/kubernetes/config
./createConfig.sh -n onap
```
Wait until the config container completes.
```sh
kubectl get pods --all-namespaces -a
```
Run ONAP:
```sh
cd ~/oom/kubernetes/oneclick/
./createAll.bash -n onap
```
Check ONAP status:
```sh
watch kubectl get pods --all-namespaces
```

## Health Check (excluding DCAE)

Add missing GLOBAL_INJECTED_SCRIPT_VERSION = "amsterdam" to `/dockerdata-nfs/onap/robot/eteshare/config/vm_properties.py`.  Note you need to do this in two places in that file.  If this is not done, the init_robot step below will fail.

If your OpenStack API Endpoints were defined using the "controller" host name instead of a numeric IP, add the controller IP address to /etc/hosts inside the Robot container.

Initialize Robot container:
```sh
cd ~/oom/kubernetes/robot
echo test | ./demo-k8s.sh init_robot
```
Run health check:
```sh
cd ~/oom/kubernetes/robot
./ete-k8s.sh health
```
At this point everything should pass except DCAE.

If ASDC fails health check, you'll need to redeploy the entire ONAP stack:
```sh
cd ~/oom/kubernetes/oneclick
./deleteAll.bash -n onap
./createAll.bash -n onap
```

## DCAE (Not Fully Working Yet)
DCAE may fail to initialize due to https://jira.onap.org/browse/DCAEGEN2-201.  In that case, delete the orcl00 VM, release the floating IPs, and re-install the dcaegen2 pod in OOM:
```sh
cd ~/oom/kubernetes/oneclick/
./deleteAll.bash -n onap -a dcaegen2
./createAll.bash -n onap -a dcaegen2
```

## Post Deployment Steps 

Check kubernetes services port mappings to access container API/UIs:
```sh
kubectl get services --all-namespaces -o wide
```
Check consul for service statuses.

Login to portal UI using vnc-portal.  Go to http://${kube-ip}:30211/.  Password is "password".

Open up Firefox to http://portal.api.simpledemo.onap.org:8989/ONAPPORTAL/login.htm.
