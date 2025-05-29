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
[rosa@bastion ~]$ rosa list clusters
ID                                NAME        STATE  TOPOLOGY
2j1ub9cdloflimkfjm00jpq21b27v4ij  rosa-5qppb  ready  Hosted CP

rosa create machinepool -c rosa-556nh --name=vm-nodes --replicas=2 --instance-type=m5.metal
rosa create machinepool -c rosa-556nh --name=odf-nodes --replicas=3 --instance-type=m5.4xlarge

rosa list machinepools --cluster rosa-556nh
```


## Setup ODF
once the previous step is finished.
On the Operator-hub console, look for ODF operator. Select it.
You need an arn to deploy since it's a ROSA cluster, I typically use my "own credentials".

```
arn:aws:iam::614894697993:user/sdelord@redhat.com-XXXXXXXX
```
You cut and paste those credentials into the ODF operator setup.
Just click on the create button.


Once the ODF operator is deployed, you can now create a "Storage System" (either via the Storage -> ODF view on the console or via the Deployed-Operators -> ODF).
As part of the various screens select:
 - ceph-rbd as the new default storage class
 - select the gp3-csi and the 3 m5.4x instances

## Deploy OCP-virt
Once ODF has been deployed, you can then configure the OCP-virt operator.
Select in Operator-hub the OpenShift virtualiszation and do the standard deployment.

You then need to create a ClusterSomething CR by using the defaults.

## Deploy the VMs.

Once OCP-virt is up and running.

You can start deploying VMs.

### Linux VM
For the linux one just go to Template, select fedora, select quick deploy and the VM should deploy without any "warning". You should be able to demo live-migration straight away.


### Windows VM
The windows one is a bit more tricky.
Go and select the 


![Browser](https://github.com/SimonDelord/Virtualization-on-ROSA/blob/main/images/ocp-virt-windows-vm-1.png)

![Browser](https://github.com/SimonDelord/Virtualization-on-ROSA/blob/main/images/ocp-virt-windows-vm-2.png)

![Browser](https://github.com/SimonDelord/Virtualization-on-ROSA/blob/main/images/ocp-virt-windows-3.png)

