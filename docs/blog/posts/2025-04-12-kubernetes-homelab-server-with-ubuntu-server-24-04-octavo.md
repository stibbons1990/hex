---
date: 2025-04-12
draft: true
categories:
 - Linux
 - Hardware
 - Ubuntu
 - Server
 - Btrfs
 - Intel NUC
title: Kubernetes homelab server with Ubuntu Server 24.04 (octavo)
---

*Need. More. Server. Need. More. POWER!!!*

Just a bit more, maybe *quite a bit* more, to run services that are more
CPU-intensive than those already running in the current 
[Single-node Kubernetes cluster on an Intel NUC: lexicon](./2023-03-25-single-node-kubernetes-cluster-on-ubuntu-server-lexicon.md).

<!-- more --> 

## Hardware

Ubuntu's list of
[Recommended and Certified Hardware](https://ubuntu.com/appliance/hardware#intel-nuc)
is just as short and *ancient* as it was 2 years ago, but NUC systems have a now
long history of being well suppored under Linux, so this time I'm taking chances:

*  [ASUS NUC 13 Pro Tall PC Kit RNUC13ANHI700000I w/ Intel Core i7-1360P](https://webshop.asus.com/de/Mini-PCs/ASUS-NUC-13-Pro-Tall-PC-Kit-RNUC13ANHI700000I/90AR00C1-M000F0) ($570)
*  [Kingston FURY Impact 1x 32GB, 3200 MHz, DDR4-RAM, SODIMM](https://www.kingston.com/en/memory/gaming/kingston-fury-impact-ddr4-memory?speed=3200mt%2Fs&total%20(kit)%20capacity=32gb&kit=single%20module&dram%20density=16gbit) ($110)
*  [Kingston FURY Renegade 4000 GB, M.2 2280](https://www.kingston.com/en/ssd/gaming/kingston-fury-renegade-nvme-m2-ssd) ($250)

### Bootable USB stick

[Get Ubuntu Server](https://ubuntu.com/download/server) (24.04.**2** LTS) and 
[create a bootable USB stick on Ubuntu](https://ubuntu.com/tutorials/create-a-usb-stick-on-ubuntu#1-overview).

## Install Ubuntu Server 24.04

https://ubuntu.com/tutorials/install-ubuntu-server

## Kubernetes
