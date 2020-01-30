---
title: "Setup Ansible on your Mac to connect to remote Windows Domain Servers/Workstations using secure Kerberos"
date: 2020-01-30
description: "This is about how to setup Ansible on your Mac to connect to remote Windows Domain Servers/Workstations using secure Kerberos."
ghissueid: 3
tags: ["ansible","kerberos"]
draft: false
---

Ansible supports authentication to Windows using NTLM over http (bad), NTLM over https (better) or Kerberos over HTTPS (best).  Surprisingly it's the easiest to setup also, so no excuses.  Ansible's documentation states that if your running in a domain environment, Kerberos should be used over NTLM.

We're going to cover the following setup requirements, then we will throw a couple commands at Windows to prove we have successfully authed.

## Installing the requirements

As i said, Kerberos is already installed on your Mac, so we just need to install the python winrm library.  To do so, do this `pip install pywinrm`.

## Installing a Certificate

Its best to do this in a Group Policy, so that the certificate will auto-enroll it.  Go to your AD Certificate Authority, copy the Web Server template, choose your expiry data save etc. Now setup your AD GP to auto-enrol.

## Configuring a listener

Unfortunately you can't setup a GP for this, so you need to use PowerShell Remoting, like this. `Invoke-Command -ComputerName $list_of_computers -ScriptBlock {winrm quickconfig -transport:https -q}`

You can now run ansible playbooks and ansible ad-hoc commands against remote Windows domain machines, using secure Kerberos over HTTPS.  Like this `ansible -i inventories/dev.yaml hv -m win_shell -a 'Get-CimInstance -ClassName win32_operatingsystem | select csname, lastbootuptime'` and get results like this

```
host1.domain.net | CHANGED | rc=0 >>
csname  lastbootuptime       
------  --------------       
HOST1 24/07/2019 1:49:10 PM

host2.domain.net | CHANGED | rc=0 >>
csname  lastbootuptime       
------  --------------       
HOST2 18/09/2019 6:56:56 AM

host3.domain.net | CHANGED | rc=0 >>
sname  lastbootuptime       
------  --------------       
HOST3 23/07/2019 1:48:38 PM
```
