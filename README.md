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

ID         AUTOSCALING  REPLICAS  INSTANCE TYPE  LABELS    TAINTS    AVAILABILITY ZONE  SUBNET                    DISK SIZE  VERSION  AUTOREPAIR
odf-nodes  No           3/3       m5.4xlarge                         us-east-2a         subnet-01b781d7fdf9230ab  300 GiB    4.17.30  Yes
vm-nodes   No           2/2       m5.metal                           us-east-2a         subnet-01b781d7fdf9230ab  300 GiB    4.17.30  Yes
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

![Browser](https://github.com/SimonDelord/Virtualization-on-ROSA/blob/main/images/rosa-virt-odf-1.png)


![Browser](https://github.com/SimonDelord/Virtualization-on-ROSA/blob/main/images/rosa-virt-odf-2.png)


Once the ODF operator is deployed, you can now create a "Storage System" (either via the Storage -> ODF view on the console or via the Deployed-Operators -> ODF).
As part of the various screens select:
 - ceph-rbd as the new default storage class
 - select the gp3-csi and the 3 m5.4x instances
 - select the default for everything else


![Browser](https://github.com/SimonDelord/Virtualization-on-ROSA/blob/main/images/rosa-virt-odf-3.png)

![Browser](https://github.com/SimonDelord/Virtualization-on-ROSA/blob/main/images/rosa-virt-odf-4.png)

![Browser](https://github.com/SimonDelord/Virtualization-on-ROSA/blob/main/images/rosa-virt-odf-5.png)


Once you go and check on the StorageClasses, there will 2 defaults. Remove the gp3-csi by changing the label to false

```
 annotations:
    storageclass.kubernetes.io/is-default-class: 'false'
```

![Browser](https://github.com/SimonDelord/Virtualization-on-ROSA/blob/main/images/rosa-virt-odf-6.png)



## Deploy OCP-virt
Once ODF has been deployed, you can then configure the OCP-virt operator.
Select in Operator-hub the OpenShift virtualiszation and do the standard deployment.

![Browser](https://github.com/SimonDelord/Virtualization-on-ROSA/blob/main/images/rosa-virt-virt-1.png)


You then need to create a HyperConverged CR by using the defaults.

You should then have a "virtualization tab" appearing on the OpenShift console tab on the left.

![Browser](https://github.com/SimonDelord/Virtualization-on-ROSA/blob/main/images/ocp-virt-virt-2.png)


## Deploy the VMs.

Once OCP-virt is up and running. You go to the Virtualization tab, create VMs from Templates and select the defaults (you can change the name and cpu/mem).

You can start deploying VMs.
it shoudl come up and you can play with the live migration feature or any other console aspects.

![Browser](https://github.com/SimonDelord/Virtualization-on-ROSA/blob/main/images/rosa-virt-linux-1.png)

![Browser](https://github.com/SimonDelord/Virtualization-on-ROSA/blob/main/images/rosa-virt-linux-2.png)


### Linux VM
For the linux one just go to Template, select fedora, select quick deploy and the VM should deploy without any "warning". You should be able to demo live-migration straight away.


### Windows VM
The windows one is a bit more tricky.
Go and select the Windows2019 Server template and follow the instructions below.


![Browser](https://github.com/SimonDelord/Virtualization-on-ROSA/blob/main/images/ocp-virt-windows-vm-1.png)



![Browser](https://github.com/SimonDelord/Virtualization-on-ROSA/blob/main/images/ocp-virt-windows-vm-2.png)



![Browser](https://github.com/SimonDelord/Virtualization-on-ROSA/blob/main/images/ocp-virt-windows-3.png)


Select the boot mode as BIOS.

Copy and paste the following in the "script" "autounattend.xml" section

```
<?xml version="1.0" encoding="utf-8"?>
<unattend xmlns="urn:schemas-microsoft-com:unattend" xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="urn:schemas-microsoft-com:unattend">
  <settings pass="windowsPE">
    <component name="Microsoft-Windows-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
      <DiskConfiguration>
        <Disk wcm:action="add">
          <CreatePartitions>
            <CreatePartition wcm:action="add">
              <Order>1</Order>
              <Extend>true</Extend>
              <Type>Primary</Type>
            </CreatePartition>
          </CreatePartitions>
          <ModifyPartitions>
            <ModifyPartition wcm:action="add">
              <Active>true</Active>
              <Format>NTFS</Format>
              <Label>System</Label>
              <Order>1</Order>
              <PartitionID>1</PartitionID>
            </ModifyPartition>
          </ModifyPartitions>
          <DiskID>0</DiskID>
          <WillWipeDisk>true</WillWipeDisk>
        </Disk>
      </DiskConfiguration>
      <ImageInstall>
        <OSImage>
          <InstallFrom>
            <MetaData wcm:action="add">
              <Key>/IMAGE/NAME</Key>
              <Value>Windows Server 2019 SERVERSTANDARD</Value>
            </MetaData>
          </InstallFrom>
          <InstallTo>
            <DiskID>0</DiskID>
            <PartitionID>1</PartitionID>
          </InstallTo>
        </OSImage>
      </ImageInstall>
      <UserData>
        <AcceptEula>true</AcceptEula>
        <FullName>Administrator</FullName>
        <Organization>My Organization</Organization>
      </UserData>
      <EnableFirewall>false</EnableFirewall>
    </component>
    <component name="Microsoft-Windows-International-Core-WinPE" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
      <SetupUILanguage>
        <UILanguage>en-US</UILanguage>
      </SetupUILanguage>
      <InputLocale>en-US</InputLocale>
      <SystemLocale>en-US</SystemLocale>
      <UILanguage>en-US</UILanguage>
      <UserLocale>en-US</UserLocale>
    </component>
  </settings>
  <settings pass="offlineServicing">
    <component name="Microsoft-Windows-LUA-Settings" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
      <EnableLUA>false</EnableLUA>
    </component>
  </settings>
  <settings pass="specialize">
    <component name="Microsoft-Windows-Shell-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
      <AutoLogon>
        <Password>
          <Value>R3dh4t1!</Value>
          <PlainText>true</PlainText>
        </Password>
        <Enabled>true</Enabled>
        <LogonCount>999</LogonCount>
        <Username>Administrator</Username>
      </AutoLogon>
      <OOBE>
        <HideEULAPage>true</HideEULAPage>
        <HideLocalAccountScreen>true</HideLocalAccountScreen>
        <HideOnlineAccountScreens>true</HideOnlineAccountScreens>
        <HideWirelessSetupInOOBE>true</HideWirelessSetupInOOBE>
        <NetworkLocation>Work</NetworkLocation>
        <ProtectYourPC>3</ProtectYourPC>
        <SkipMachineOOBE>true</SkipMachineOOBE>
      </OOBE>
      <UserAccounts>
        <LocalAccounts>
          <LocalAccount wcm:action="add">
            <Description>Local Administrator Account</Description>
            <DisplayName>Administrator</DisplayName>
            <Group>Administrators</Group>
            <Name>Administrator</Name>
          </LocalAccount>
        </LocalAccounts>
      </UserAccounts>
      <TimeZone>Eastern Standard Time</TimeZone>
    </component>
  </settings>
  <settings pass="oobeSystem">
    <component name="Microsoft-Windows-International-Core" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
      <InputLocale>en-US</InputLocale>
      <SystemLocale>en-US</SystemLocale>
      <UILanguage>en-US</UILanguage>
      <UserLocale>en-US</UserLocale>
    </component>
    <component name="Microsoft-Windows-Shell-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
      <AutoLogon>
        <Password>
          <Value>R3dh4t1!</Value>
          <PlainText>true</PlainText>
        </Password>
        <Enabled>true</Enabled>
        <LogonCount>999</LogonCount>
        <Username>Administrator</Username>
      </AutoLogon>
      <OOBE>
        <HideEULAPage>true</HideEULAPage>
        <HideLocalAccountScreen>true</HideLocalAccountScreen>
        <HideOnlineAccountScreens>true</HideOnlineAccountScreens>
        <HideWirelessSetupInOOBE>true</HideWirelessSetupInOOBE>
        <NetworkLocation>Work</NetworkLocation>
        <ProtectYourPC>3</ProtectYourPC>
        <SkipMachineOOBE>true</SkipMachineOOBE>
      </OOBE>
      <UserAccounts>
        <LocalAccounts>
          <LocalAccount wcm:action="add">
            <Description>Local Administrator Account</Description>
            <DisplayName>Administrator</DisplayName>
            <Group>Administrators</Group>
            <Name>Administrator</Name>
          </LocalAccount>
        </LocalAccounts>
      </UserAccounts>
      <TimeZone>Eastern Standard Time</TimeZone>
    </component>
  </settings>
</unattend>
```
