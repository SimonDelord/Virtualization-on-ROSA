# Virtualization-on-ROSA
This describes the set of steps to configure OCP-virt on ROSA and a build of a RHEL and Windows VMs



## Setup of the environment

I personnaly use the internal Red Hat system for it (RHDP).

These are the steps for it:
 - request a ROSA environment
 - configure 2 machine pools on the ROSA cluster (I normally remove the default one)
   - one of 2 bare-metal nodes (to deploy the VMs)
   - one of 3 m5.4xlarge nodes (to deploy ODF to allow for RWX for VM live-migration)

- install the ODF operator
- install the OCP-virt operator
- Deploy VMs
- you can then start playing around with the networking setup on the ROSA cluster (ovn-secondary, udn, etc..).


## Setup up the new machine pools

You need to have the rosa-cli installed.
Assuming this is done, you would typically do

```
rosa create machinepool -c cluster-name --name odf-pool --type 
rosa create machinepool -c cluster-name --name odf-pool --type 
```
