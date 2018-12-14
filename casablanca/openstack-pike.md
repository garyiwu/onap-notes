# openstack-pike
This is a step-by-step installation guide for deploying Ubuntu OpenStack Pike to run ONAP Casablanca, including any workarounds I encountered.

## Basic OpenStack Pike Installation
This installation guide assumes the following:
* Installation of Ubuntu OpenStack Pike per https://docs.openstack.org/install-guide/openstack-services.html#minimal-deployment-for-pike.
  * All the API endpoints should be specified with numeric IP addresses to avoid issues with DNS resolution of "controller" as the host name.
  * Make sure that the provider network name is "external".  This is currently hard-coded in the ONAP Casablanca Robot testsuite.
* Installation of Cinder per https://docs.openstack.org/cinder/pike/install/index-ubuntu.html
  * Need to also add cinder v1 service and endpoints
* Installation of Heat per https://docs.openstack.org/heat/pike/install/

