---
title: "Intro To The Infrastructure"
description: "This is the basic intro of what this blog is trying to accomplish for deploying redteam infrastructure in using automation"
tags: ["network", "intro","diagram","network diagram"]
---


This blog is going to go through my journey of setting up my lab for deploying red team infrastructure. It's going to consist of building out the infrastructure in a development/staging environment, setting up our pipelines for building our C2s for easy deployment, creating a production-like environment, and finally deploying our redirectors in a cloud-based infrastructure that can reverse proxy to our internally-held C2s.

There are a lot of infrastructure deployments that I have seen that are mostly being deployed solely to the cloud, but the idea of this infrastructure is to contain as much as possible internally. This will help protect a lot of the infrastructure from blue team discovery except the external redirectors and the webpages they are displaying, and most importantly would be keeping any client data out of the cloud and into an infrastructure completely controlled by us.

Below is a basic network diagram of the layout of the infrastructure to be built. 

{{< mermaid >}}

flowchart TB;

opnet[Operator VLAN];

mannet[Management VLAN];

webnet[Public Web VLAN];

labnet[Lab VLAN];

WAN[WAN];

inet((internet));

VPS[(Cloud VPS)];

  

opnet --tcp/22--> mannet;

opnet --proxyserver/443--> labnet;

opnet --gitea/22-->labnet;

mannet--tcp/22-->labnet;

mannet---tcp/22---webnet;

  
  

webnet--->WAN;

opnet---->WAN;

labnet--->WAN;

WAN------>inet;

inet--tailscale-->WAN;

WAN--tailscale-->webnet;

VPS---tailscale-------->inet;

  

{{< /mermaid >}}

Part of this will be deploying an internal Gitea, but just to help anyone following along, I will also be publishing a lot of the IaC to a public GitHub.

The upcoming entry will be setting up a jumphost in management vlan the  with all our basic utilities for managing our infrastructure.

I then want to follow it up with a crash course in Ansible since we will be installing Docker on a lot of machines, and this will go over a basic setup script and some basic maintenance scripts, like a script for OS updates and drive expansion.

Eventually, I will be deploying a single-master Kubernetes infrastructure in our lab staging VLAN and using Gitea to maintain the deployed applications with Flux. This will lead to building pipelines to deploy Sliver, Havoc, and Adaptix C2s.

Well, that's a summary and an overview of what to expect in the future


