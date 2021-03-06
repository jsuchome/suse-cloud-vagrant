# How to automatically deploy and test a highly available OpenStack cloud

First deploy SUSE Cloud and 3 client nodes as described in the
[HOWTO guide](HOWTO.md).

## Deployment of a highly available OpenStack cloud

### Identify available nodes

*   Open the Crowbar UI at [http://192.168.124.10:3000](http://192.168.124.10:3000)
    (login: `crowbar`, password: `crowbar`)
*   You should have four nodes active in tab *Nodes* → *Dashboard*
    (they should be in the green state)
*   If using Vagrant on Linux, run `./vagrant/scripts/list-MACs.sh` to see the MAC
    addresses (which also forms the hostname) for each node.
    Otherwise, just use the VirtualBox GUI to determine the MAC
    address of the second network interface on each node.
*   In the Crowbar UI, go to tab *Nodes* → *Bulk Edit* and assign
    aliases `controller1`, `controller2` and `compute1` to the
    machines that have the MAC addresses found above.

**N.B.!** Leave the *Public Name* field blank! (since this would
require extra upstream DNS records.)

### Deploy a Pacemaker cluster

*   Go to tab *Barclamps* → *OpenStack *and click on *Edit* for *Pacemaker*
*   Change `proposal_1` to `cluster1` and click *Create*
*   Scroll down to *Deployment*, and drag the `controller1` and `controller2`
    nodes to both roles (**pacemaker-cluster-member** and **hawk-server**)
*   Scroll back up and change the following options:
    *   *STONITH* section:
        *   Change *Configuration mode for STONITH* to
            **STONITH - Configured with STONITH Block Devices (SBD)**
        *   Enter `/dev/sdc` for both nodes under **Block devices for node**
    *   *DRBD* section:
        *   Change *Prepare cluster for DRBD* to `true`
            (`controller1` and `controller2` should have free disk to claim)
    *   *Pacemaker GUI* section:
        *   Change *Setup non-web GUI (hb_gui)* to `true`
*   Click on *Apply* to deploy Pacemaker

**Hint:** You can follow the deployment by typing
`tail -f /var/log/crowbar/chef_client/*` on the admin node.

### Deploy Barclamps / OpenStack / Database

*   Go to tab *Barclamps* → *OpenStack* and click on *Create* for Database
*   Under *Deployment*, remove the node that is assigned to the
    **database-server** role, and drag `cluster1` to the **database-server** role
*   Change the following options:
    * *High Availability* section:
        * Change *Storage mode* to **DRBD**
        * Change *Size to Allocate for DRBD Device* to **1**
*   Click on *Apply* to deploy the database

### Deploy Barclamps / OpenStack / RabbitMQ

*   Go to tab *Barclamps* → *OpenStack* and click on *Create* for RabbitMQ
*   Under *Deployment*, remove the node that is assigned to the
    **rabbitmq-server** role, and drag `cluster1` to the
    **rabbitmq-server** role
*   Change the following options:
    * *High Availability* section:
        * Change *Storage mode* to **DRBD**
        * Change *Size to Allocate for DRBD Device* to **1**
*   Click on *Apply* to deploy RabbitMQ

### Deploy Barclamps / OpenStack / Keystone

*   Go to tab *Barclamps* → *OpenStack* and click on *Create* for Keystone
*   Do not change any option
*   Under *Deployment*, remove the node that is assigned to the
    **keystone-server** role, and drag `cluster1` to the **keystone-server** role
*   Click on *Apply* to deploy Keystone

### Deploy Barclamps / OpenStack / Glance

**N.B.!** To simplify the HA setup of Glance for the workshop, a NFS
export has been automatically setup on the admin node, and mounted on
/var/lib/glance on both controller nodes. Reliable shared storage is
highly recommended for production; also note that alternatives exist
(for instance, using the swift or ceph backends).

*   Go to tab *Barclamps* → *OpenStack* and click on *Create* for Glance
*   Do not change any option
*   Under *Deployment*, remove the node that is assigned to the **glance-server** role, and drag `cluster1` to the **glance-server** role
*   Click on *Apply* to deploy Glance

### Deploy Barclamps / OpenStack / Cinder

**N.B.!** The cinder volumes will be stored on the compute node. The
controller nodes are not used to allow easy testing of failover. On a
real setup, using a SAN to store the volumes would be recommended.

*   Go to tab *Barclamps* → *OpenStack* and click on *Create* for Cinder
*   Change the following options:
    * Change *Type of Volume* to **Local file**
*   Under *Deployment*, remove the node that is assigned to the **cinder-controller** role, and drag `cluster1` to the **cinder-controller** role
*   Under *Deployment*, remove the node that is assigned to the **cinder-volume** role, and drag **compute1** to the **cinder-volume** role
*   Click on *Apply* to deploy Cinder

### Deploy Barclamps / OpenStack / Neutron

*   Go to tab *Barclamps* → *OpenStack* and click on *Create* for Neutron
*   Do not change any option
*   Under *Deployment*, remove the node that is assigned to the **neutron-server** role, and drag `cluster1` to the **neutron-server **role
*   remove the node that is assigned to the **neutron-l3** role, and drag and drop `cluster1` to **neutron-l3 **role
*   Click on *Apply* to deploy Neutron

### Deploy Barclamps / OpenStack / Nova

*   Go to tab *Barclamps* → *OpenStack* and click on *Create* for Nova
*   Do not change any option
*   Under *Deployment*:
    * remove all nodes which are assigned to roles such as **nova-multi-controller** and **nova-multi-compute-xen**
    * drag `cluster1` to the **nova-multi-controller **role
    * drag **compute1** to the **nova-multi-compute-qemu **role
*   Click on *Apply* to deploy Nova

### Deploy Barclamps / OpenStack / Horizon

*   Go to tab *Barclamps* → *OpenStack* and click on *Create* for Horizon
*   Do not change any option
*   Under *Deployment*, remove the node that is assigned to the **nova_dashboard-server** role, and drag `cluster1` to the **nova_dashboard-server **role
*   Click on *Apply* to deploy Horizon

## Playing with Cloud

### Introduction

*   To log into the OpenStack Dashboard (Horizon):
    * In the Crowbar web UI, click on *Nodes*
    * Click on `controller1`
    * Click on the **OpenStack Dashboard (admin)** link
    * login: `admin`, password: `crowbar`
*   Choose the `Project` tab and for *Current Project* select `openstack`

### Upload image

*   Go to Images & Snapshots from Manage Compute section

*   Click on **Create Image** button and provide the following data:
    * *Name* - image name
    * *Image Source* - Image File
    * *Image File* - click on Browse button to choose image file
    * Format - QCOW2 - QEMU Emulator
    * Minimum Disk GB - 0 (no minimum)
    * Minimum RAM MB - 0 (no minimum)
    * Public - check this option
    * Protected - leave unchecked
*   Click on **Create Image** button to proceed and upload image

### Launching VM instance

*   Go to Instances from Manage Compute section
*   Click on Launch Instance button
*   In the *Details* tab set up
    * Availability Zone - nova
    * Instance Name - name of new VM
    * Flavor - `m1.tiny`
    * Instance Count - 1
    * Instance Boot Source - Boot from image
    * Image Name - choose uploaded image file
*   in Networking tab set up
    * drag and drop fixed network from Available Networks to Selected Networks
*   click on Launch button and wait until new VM instance will be ready

## Playing with High-Availability

### Introduction

*   open console to `controller1`, `controller2` and compute1 nodes and login there
*   on `controller1` and `controller2` run `crm_mon` command
*   on `controller1` run:

        . .openrc
        nova service-list
        nova list

### Failover scenarios for services

*   on `controller1` or `controller2` try to kill OpenStack services
    using commands like:

        pkill openstack-keystone
        pkill openstack-glance
        pkill openstack-nova

*   watch on consoles with `crm_mon` how all services are bringing up by pacemaker

### Failover scenarios for nodes

*   on `controller1` run `crm_mon` command
*   kill `controller2` node via `halt` or `shutdown -h now`
*   watch on consoles with `crm_mon` how all services are bringing up by pacemaker
