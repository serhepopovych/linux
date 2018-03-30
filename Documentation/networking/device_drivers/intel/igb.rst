.. SPDX-License-Identifier: GPL-2.0+

Linux* Base Driver for Intel(R) Ethernet Network Connection
===========================================================

Intel Gigabit Linux driver.
Copyright(c) 1999-2018 Intel Corporation.

Contents
========

- Identifying Your Adapter
- Command Line Parameters
- Additional Configurations
- Support


Identifying Your Adapter
========================
For information on how to identify your adapter, and for the latest Intel
network drivers, refer to the Intel Support website:
http://www.intel.com/support


Command Line Parameters
========================
If the driver is built as a module, the following optional parameters are used
by entering them on the command line with the modprobe command using this
syntax::

    modprobe igb [<option>=<VAL1>,<VAL2>,...]

There needs to be a <VAL#> for each network port in the system supported by
this driver. The values will be applied to each instance, in function order.
For example::

    modprobe igb max_vfs=2,4

In this case, there are two network ports supported by igb in the system.

NOTE: A descriptor describes a data buffer and attributes related to the data
buffer. This information is accessed by the hardware.

max_vfs
-------
:Valid Range: 0-7

This parameter adds support for SR-IOV. It causes the driver to spawn up to
max_vfs worth of virtual functions.  If the value is greater than 0 it will
also force the VMDq parameter to be 1 or more.

The parameters for the driver are referenced by position. Thus, if you have a
dual port adapter, or more than one adapter in your system, and want N virtual
functions per port, you must specify a number for each port with each parameter
separated by a comma. For example::

    modprobe igb max_vfs=4

This will spawn 4 VFs on the first port.

::

    modprobe igb max_vfs=2,4

This will spawn 2 VFs on the first port and 4 VFs on the second port.

NOTE: Caution must be used in loading the driver with these parameters.
Depending on your system configuration, number of slots, etc., it is impossible
to predict in all cases where the positions would be on the command line.

NOTE: Neither the device nor the driver control how VFs are mapped into config
space. Bus layout will vary by operating system. On operating systems that
support it, you can check sysfs to find the mapping.

NOTE: When either SR-IOV mode or VMDq mode is enabled, hardware VLAN filtering
and VLAN tag stripping/insertion will remain enabled. Please remove the old
VLAN filter before the new VLAN filter is added. For example::

    ip link set eth0 vf 0 vlan 100	// set vlan 100 for VF 0
    ip link set eth0 vf 0 vlan 0	// Delete vlan 100
    ip link set eth0 vf 0 vlan 200	// set a new vlan 200 for VF 0

Debug
-----
:Valid Range: 0-16 (0=none,...,16=all)
:Default Value: 0

This parameter adjusts the level debug messages displayed in the system logs.


Additional Features and Configurations
======================================

Jumbo Frames
------------
Jumbo Frames support is enabled by changing the Maximum Transmission Unit (MTU)
to a value larger than the default value of 1500.

Use the ifconfig command to increase the MTU size. For example, enter the
following where <x> is the interface number::

    ifconfig eth<x> mtu 9000 up

Alternatively, you can use the ip command as follows::

    ip link set mtu 9000 dev eth<x>
    ip link set up dev eth<x>

This setting is not saved across reboots. The setting change can be made
permanent by adding 'MTU=9000' to the file:

- For RHEL: /etc/sysconfig/network-scripts/ifcfg-eth<x>
- For SLES: /etc/sysconfig/network/<config_file>

NOTE: The maximum MTU setting for Jumbo Frames is 9216. This value coincides
with the maximum Jumbo Frames size of 9234 bytes.

NOTE: Using Jumbo frames at 10 or 100 Mbps is not supported and may result in
poor performance or loss of link.


ethtool
-------
The driver utilizes the ethtool interface for driver configuration and
diagnostics, as well as displaying statistical information. The latest ethtool
version is required for this functionality. Download it at:

https://www.kernel.org/pub/software/network/ethtool/


Enabling Wake on LAN* (WoL)
---------------------------
WoL is configured through the ethtool* utility.

WoL will be enabled on the system during the next shut down or reboot. For
this driver version, in order to enable WoL, the igb driver must be loaded
prior to shutting down or suspending the system.

NOTE: Wake on LAN is only supported on port A of multi-port devices.  Also
Wake On LAN is not supported for the following device:
- Intel(R) Gigabit VT Quad Port Server Adapter


Multiqueue
----------
In this mode, a separate MSI-X vector is allocated for each queue and one for
"other" interrupts such as link status change and errors. All interrupts are
throttled via interrupt moderation. Interrupt moderation must be used to avoid
interrupt storms while the driver is processing one interrupt. The moderation
value should be at least as large as the expected time for the driver to
process an interrupt. Multiqueue is off by default.

REQUIREMENTS: MSI-X support is required for Multiqueue. If MSI-X is not found,
the system will fallback to MSI or to Legacy interrupts. This driver supports
receive multiqueue on all kernels that support MSI-X.

NOTE: On some kernels a reboot is required to switch between single queue mode
and multiqueue mode or vice-versa.


MAC and VLAN anti-spoofing feature
----------------------------------
When a malicious driver attempts to send a spoofed packet, it is dropped by the
hardware and not transmitted.

An interrupt is sent to the PF driver notifying it of the spoof attempt. When a
spoofed packet is detected, the PF driver will send the following message to
the system log (displayed by the "dmesg" command):
Spoof event(s) detected on VF(n), where n = the VF that attempted to do the
spoofing


Setting MAC Address, VLAN and Rate Limit Using IProute2 Tool
------------------------------------------------------------
You can set a MAC address of a Virtual Function (VF), a default VLAN and the
rate limit using the IProute2 tool. Download the latest version of the
IProute2 tool from Sourceforge if your version does not have all the features
you require.

Credit Based Shaper (Qav Mode)
------------------------------
When enabling the CBS qdisc in the hardware offload mode, traffic shaping using
the CBS (described in the IEEE 802.1Q-2018 Section 8.6.8.2 and discussed in the
Annex L) algorithm will run in the i210 controller, so it's more accurate and
uses less CPU.

When using offloaded CBS, and the traffic rate obeys the configured rate
(doesn't go above it), CBS should have little to no effect in the latency.

The offloaded version of the algorithm has some limits, caused by how the idle
slope is expressed in the adapter's registers. It can only represent idle slopes
in 16.38431 kbps units, which means that if a idle slope of 2576kbps is
requested, the controller will be configured to use a idle slope of ~2589 kbps,
because the driver rounds the value up. For more details, see the comments on
:c:func:`igb_config_tx_modes()`.

NOTE: This feature is exclusive to i210 models.

Double VLAN support
--------------------

** Configuration

By default support for double vlans (i.e. Double VLAN mode in hardware) isn't
active. One might use following command to activate it:

  # ethtool -K eth2 rx-vlan-stag-hw-parse on
  Actual changes:
  tx-vlan-offload: off [requested on]
  rx-vlan-stag-hw-parse: on
  rx-vlan-stag-filter: on

To change Ethernet frame Type field id from default ETH_P_8021Q (0x8100) to
ETH_P_8021AD (0x88a8) as per IEEE 802.1ad specification one might use
following command:

  # ethtool --set-priv-flags eth2 stag-ethertype-802.1ad on

and checked with following:

  # ethtool --show-priv-flags eth2
  Private flags for eth2:
  stag-ethertype-802.1ad: on

** Implementation details

According to "Intel(R) 82575EB Gigabit Ethernet Controller Software
Developer’s Manual and EEPROM Guide" and future documentation hardware
supports Double VLAN (DV) mode for stacked vlan scenarious. However it has few
limitations:

  o It is assumed that in this mode frame contains at least one VLAN header:
    no hardware offloading will be provided for frames without it.

  o If only one header present in the frame it is assumed to be outer.

  o Hardware does not provide any additional offload except skipping outer
    header to parse inner and next protocols (e.g. IP, IPv6).

  o No support for outer header strip/insert and filtering based on vid in
    hardware.

  o Support for inner header strip/insert and filtering based on vid in
    hardware is available as in single vlan mode.

Note that Linux pushes VLAN headers for stacked vlans in native order: from
upper to lower. Later is put into skb->vlan_tci when output network device
declares support for hardware offload for header insertion on transmit.

Same applies to filtering: only vids from lowest vlan propagated to the
driver/hardware.

Also Linux has strict assumption about Ethernet frame Type (EtherType) field
and VLAN encapsulated protocol (EncapProto):

  o Outer header should have ETH_P_8021AD in EtherType.

  o Inner header should have ETH_P_8021Q in outer header EncapProto.

All above places following restrictions on implementation of double vlans
support in the driver:

  o Filtering based on inner vid must be turned off: VFTA (VLAN Filter Table
    Array) contains vids from lowest vlans (outer) but hardware applies
    filtering to vids from upper vlans (outer).

  o No support for inner header insertion by hardware: skb->vlan_tci contains
    information for outer header, inner header already added to packet buffer
    by software. Hardware expects to find inner header vlan_tci in transmit
    descriptor to offload header insertion.

  o Filtering based on outer vid implemented in software and turned on by
    default: user might turn it off if required. Promiscuous mode is supported
    and can be turned on either implicitly or explicitly (e.g. via
    ip-link(8)).

  o Inner header stripping supported in hardware and turned on by default:
    user might turn it off with adding neligible overhead.

Note about EtherType field for outer VLAN header:

  o On receive hardware skips outer header when EtherType value matches one
    configured in NIC specific register (VET.VET_EXT or VET.EXT_VET in newer
    Inte(R) documents).

    If it does not match, inner header stripping, filtering by vid and any
    hardware offload for next protocol isn't provided. Frame is delivered as
    is to upper layers.

    This behavior is identical to one in single vlan mode: no offload for
    frames with unknown EtherType.

  o Turning off "rx-vlan-filter" (e.g. by ethtool) isn't supported when
    "rx-vlan-stag-hw-parse" is "on": though not active in hardware since
    applies to inner header this flag is checked by Linux before calling
    driver specific routines to add/kill vids to VFTA.

    As said before Linux has strict assumption about EtherType and EncapProto
    for inner and outer headers. This affects filtering in following manner:

      if "rx-vlan-filter" is "on" then vids for vlans with ETH_P_8021Q
      protocol will trigger call to driver specific routine to add/kill
      vids in VFTA

      if "rx-vlan-stag-hw-filter" is "on" then vids for vlans with
      ETH_P_8021AD protocol will trigger call to driver specific routine to
      add/kill vids in VFTA

    Turning off "rx-vlan-stag-hw-parse" later would not stop traffic for
    configured vlans by hardware filters for inner header.

    This preserves interoperability with "rx-vlan-stag-hw-parse" "off" mode in
    which double vlan feature is off.

** Troubleshooting and debugging

There are two NIC registers participating in Double VLAN mode configuration and
functioning: CTRL_EXT and VET. Additionally RCTL register might be interesting
to ensure filtering based on inner header vid is turned off/on:

  0x00100: RCTL (Receive control register)              0x00000000
         Receiver:                                      disabled
         Store bad packets:                             disabled
         Unicast promiscuous:                           disabled
         Multicast promiscuous:                         disabled
         Long packet:                                   disabled
         Descriptor minimum threshold size:             1/2
         Broadcast accept mode:                         ignore
         VLAN filter:                                   disabled
                                                        ^^^^^^^^
                                                        filters off for DV

  0x00018: CTRL_EXT    (Extended device control)        0x14180C00
                                                           ^
                                                           0x4000000[26]

  0x00038: VET         (VLAN Ether type)                0x81008100
                                                          ^^^^
                                                          EtherType for DV

Their contents can be dumped by ethtool(8) using following command:

  # ethtool --register-dump eth2

In CTRL_EXT register bit 26 is set when Double VLAN mode is in effect and
VET[16:31] contains Little Endian (LE) representation of EtherType.

Support
=======
For general information, go to the Intel support website at:

https://www.intel.com/support/

or the Intel Wired Networking project hosted by Sourceforge at:

https://sourceforge.net/projects/e1000

If an issue is identified with the released source code on a supported kernel
with a supported adapter, email the specific information related to the issue
to e1000-devel@lists.sf.net.
