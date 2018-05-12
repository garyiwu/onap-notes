# onap-demo

## Terminology
* FW: Firewall
* SNK: Sink
* PKG: Packet Generator

## Env Setup
* Fresh Integration-Jenkins installation
* Make sure ONAP passes healthcheck (30/30)
* Make sure Robot VM config is initialized: `./demo.sh init_robot`


* Change local /etc/hosts to change the following hostname/IP mappings to match the new IPs assigned by the deployment.
```
10.12.5.19 portal.api.simpledemo.onap.org
10.12.5.32 sdc.api.simpledemo.onap.org
10.12.5.7 vid.api.simpledemo.onap.org
10.12.5.33 policy.api.simpledemo.onap.org
10.12.5.70 aai.ui.simpledemo.onap.org
10.12.5.11 robot.simpledemo.onap.org
```

Open browser to http://portal.api.simpledemo.onap.org:8989/ONAPPORTAL/login.htm

Login as cs0008/demo123456!

Open SDC app, show that on the SDC home page there is no existing VNF or service.  Now we will add some.

## Initialize Customers and Models
From robot VM, run
```sh
root@onap-robot:~$ cd /opt/
root@onap-robot:/opt$ ./demo.sh init
```
This will create initial customer and models.

Go back to the portal SDC page, refresh, and show that some services and VFs have been created.

Download the CSAR file from the TOSCA Artifacts screen for Democfwcl.  You will need information from this CSAR to identify the resource names for the Firewall/Sink and Packet Generator VNFs later.

## Instantiate Service
* Log in to portal as demo/demo123456!
* Open VID (may need to try twice)
* Browse SDC Service Models
* Deploy demoVFWCL
  * Instance Name: vFWCL-1
  * Subscriber Name: Demonstration
  * Service Type: vFWCL
  * Note the service instance ID for later use.
* Add VNFs to vFWCL-1
  * Add VNF for Packet Generator
    * Open up the CSAR to identify the resource name for the packet generator
    * Add VNF to service instance vFWCL-1 using the resource name above
      * Instance Name: vPKG-1
      * Product Family: vFWCL
      * LCP Region: RegionOne
      * Tenant: Integration-Jenkins
  * Add VNF for Firewall/Sink
    * Open up the CSAR to identify the resource name for the vFWSNK
    * Add VNF to service instance vFWCL-1 using the resource name above
      * Instance Name: vFWSNK-1
      * Product Family: vFWCL
      * LCP Region: RegionOne
      * Tenant: Integration-Jenkins
* Preload SDNC data
  * ```./demo.sh preload vFWSNK-1 vFWSNK-Module-1 vFWSNK```
  * ```./demo.sh preload vPKG-1 vPKG-Module-1 vPKG```
* Add VNF Modules
  * Add VNF Module for vFWSNK-1
    * Instance Name: vFWSNK-Module-1
    * LCP Region: RegionOne
    * Tenant: Integration-Jenkins
    * SDN-C Pre-Load: Yes
  * Add VNF Module for vPKG-1
    * Instance Name: vPKG-Module-1
    * LCP Region: RegionOne
    * Tenant: Integration-Jenkins
    * SDN-C Pre-Load: Yes
* Run heatbridge
  * ```./demo.sh heatbridge vFWSNK-Module-1 <service-instance-id> vFWSNK```
  * ```./demo.sh heatbridge vPKG-Module-1 <service-instance-id> vPKG```
* Try to ping the FW (vFWSNK-1), SNK (demofwl01snkdemo), and PKG (demofwl01pgndemo) IPs.  If any is unreachable, soft reboot the VM using OpenStack.
* Open browser to http://[IP of demofwl01snkdemo]:667/ for the packet capture graphs.  Turn on the automatic reload at the bottom.  You should see a steady stream of packets being received by the SNK VM.

## Configure Closed Loop
* Add APPC mount point for PKG
  * ```./demo.sh appc vPKG-Module-1```
* Push vFWCL policy
  * Within the CSAR downloaded earlier, look inside `Definitions/service-Demovfwcl-template.yml`, and look for the invariantUUID of the PKG (search for vTrafficPNG).  This PKG-invariantUUID will be used in the command below.
  * Clone the demo repo on your local machine (amsterdam branch) to push policy updates.
```
cd /tmp/
git clone -b amsterdam ssh://gerrit.onap.org:29418/demo
cd demo/vnfs/vFW/scripts/
./update-vfw-op-policy.sh policy.api.simpledemo.onap.org <PKG-invariantUUID> ~/.ssh/onap_key
```
  * When the command completes, it will restart PDP-Drools and in the end show 4 controlLoopYaml JSON definitions.  At this point the policy update has completed.
  
## Observe Closed Loop Results
* Note that it could take 15 minutes after the policy is updated for the traffic pattern to change, so just let the system run for a while.
* Observe the change in the SNK packet capture graphs.
  * Before the Closed Loop policy, the traffic originially starts with alternating high and low plateaus (like a square wave). 
  * After the Closed Loop policy, the traffic will settle in the middle of the original highs and lows.  Any occasional spikes are quickly adjusted back toward the middle.
* Check http://[IP of onap-message-router]:3904/events/unauthenticated.DCAE_CL_OUTPUT/group1/C1?timeout=50000 for the list of closed-loop events.
