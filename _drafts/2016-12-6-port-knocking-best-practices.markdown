# Port knocking best practices
In this post I want to explain how to use correctly port knocking with iptables.

## Prerequisite
- knowing SSH server management
- knowing Port knocking daemon management (knockd in this post)
- knowing iptables management

## Problem
When you setup your port knocking configuration every examples on internet explain to you to configure iptables firewall for reject all SSH traffic and to setup a rule in knockd configuration who allow SSH traffic in your iptables rules when the sequence is detected.

The problem, when you setup your server in this way any attackers with a port scanner like nmap can detect that the SSH daemon run on your server. Attacker know that your iptables have a filtering rule and know that SSH service run on port (by default 22).

This is a bad practice, you leak somes sensitives informations about your system configuration.

## Solution
By default you must shutdown your SSH service and launch it only when the port knocking sequence is detected
