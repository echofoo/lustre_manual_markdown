# Installation Overview

**Table of Contents**

- [Installation Overview](#installation-overview)
    * [Steps to Installing the Lustre Software](#steps-to-installing-the-lustre-software)

This chapter provides on overview of the procedures required to set up, install and configure a Lustre file system.

### Note

If the Lustre file system is new to you, you may find it helpful to refer to [Introducing the Lustre* File System](part.intro.html) for a description of the Lustre architecture, file system components and terminology before proceeding with the installation procedure.

## Steps to Installing the Lustre Software

To set up Lustre file system hardware and install and configure the Lustre software, refer the the chapters below in the order listed:

1. *(Required)* **Set up your Lustre file system hardware.**

   See [*Determining Hardware Configuration Requirements and Formatting Options*](settinguplustresystem.html) - Provides guidelines for configuring hardware for a Lustre file system including storage, memory, and networking requirements.

2. *(Optional - Highly Recommended)* **Configure storage on Lustre storage devices.**

   See [*Configuring Storage on a Lustre File System*](configuringstorage.html) - Provides instructions for setting up hardware RAID on Lustre storage devices.

3. *(Optional)* **Set up network interface bonding.**

   See [*Setting Up Network Interface Bonding*](settingupbonding.html) - Describes setting up network interface bonding to allow multiple network interfaces to be used in parallel to increase bandwidth or redundancy.

4. *(Required)* **Install Lustre software.**

   See [*Installing the Lustre Software*](installinglustre.html) - Describes preparation steps and a procedure for installing the Lustre software.

5. *(Optional)* **Configure Lustre Networking (LNet).**

   See [*Configuring Lustre Networking (LNet)*](configuringlnet.html) - Describes how to configure LNet if the default configuration is not sufficient. By default, LNet will use the first TCP/IP interface it discovers on a system. LNet configuration is required if you are using InfiniBand or multiple Ethernet interfaces.

6. *(Required)* **Configure the Lustre file system.**

   See [*Configuring a Lustre File System*](configuringlustre.html) - Provides an example of a simple Lustre configuration procedure and points to tools for completing more complex configurations.

7. *(Optional)* **Configure Lustre failover.**

   See [*Configuring Failover in a Lustre File System*](configuringfailover.html) - Describes how to configure Lustre failover.

