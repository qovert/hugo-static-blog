---
title: "Domain Services Part 1"
author: "Brian"
date: 2021-04-02T12:08:45-07:00
draft: false
image: "/imgs/2021.03_dirsrv1_getforest.png"
categories: ['home lab']
tags: ['windows server', 'active directory']
---

By request, here's the first part in the series of how to set up a home computer lab for testing and learning.   

Let's put together a quick checklist of what we want to build before we get started:  

1. Two Windows Server 2019 Standard core virtual machines. I'm assuming you've got a working hypervisor and access to developer or evaluation licenses for Windows Server. I highly recommend using Core for domain controllers to keep them as low profile as possible.  
2. A management machine for running scripts and other tasks. Best practice is to use a 'bastion' or 'jump' machine, dedicated to this purpose. For this project I've created a Windows Server 2019 VM with the hostname 'jump'. Windows 10 clients are also totally acceptable choices for this purpose. Just make sure you can install the latest version of Remote Server Administration Tools: [RSAT](https://docs.microsoft.com/en-us/troubleshoot/windows-server/system-management-components/remote-server-administration-tools).  
3. Decide on an internal namespace. Avoid using generic TLDs like .lan, .corp, etc. These are now being sold by ICANN so your internal domain might end up someone else's property. Even more annoying, .local addresses are used by applications that rely on multicast DNS, so you can end up troubleshooting strange behavior with applications or devices you're testing. The best choice is to use an unused subdomain of a domain you already own. In this case, I'm using: FQDN: win.hrkrdr.com NETBIOS: WIN.  
4. Once you've got your namespace, you can sketch out your DNS structure. For most small test environments this might not be very complex, but if you've got multiple internal namespaces, [piholes](https://pi-hole.net/), or other requirements, it's good to draw it out now.  

## Implementation
Now that we've got the boring parts out of the way, let's do the fun part. I will be doing everything in powershell unless there's no other option. The reasons for this (other than I just like powershell, which I do) are many, but above all it's self documenting. You can write out what you're doing and save it, edit it for clarity, or even turn it into more complex tooling for  the next time. You can also give that document to others so they know what you've done. When you build these projects in code from the beginning you're already on the path to writing Infrastructure as Code.  

### Step 1: Prepare your initial domain environment 
I've spun up three VMs on different cluster nodes (if available) with the following configuration:  

| VM | Description          | vCPU Count | RAM (MB) | HDD (GB) |
|:----|:--------------------|-----------:|---------:|---------:|
| DC0 | Domain Controller   |          2 |     1024 |       32 |
| DC1 | Domain Controller   |          2 |     1024 |       32 |
| Jump | Management VM      |          4 |     4096 |       60 |

I'm using [proxmox](https://www.proxmox.com/en/), which is a Linux KVM based hypervisor solution. For best performance and compatibility, you should use [virtio drivers](https://pve.proxmox.com/wiki/Windows_VirtIO_Drivers) for Windows guests. I won't go into that  process in this here, but it will merit a future post.  

Once you've logged in and changed the default administrator password, you'll be presented with a command window on a black background. Run the [sconfig](https://social.technet.microsoft.com/wiki/contents/articles/52672.windows-server-sconfig-exe.aspx) utility and get the VM prepared:  

- Install all available updates (this takes a bit)  
- Set a static IP Address and DNS resolvers  
- Change the computer name (I chose dc0 and dc1). This is an important step, as renaming a domain controller is a bit more involved than a member server. You don't want to have to refer to WIN-QK2CYN4P1IE over and over do you?  

### Step 2: Install ADDS and build a Forest
Everything is updated and we've set and recorded some static IPs. Now we can do the thing we started out wanting to do! Log into your first domain controller and type 'powershell' at the command prompt. If you're in sconfig choose 'Exit to Command Line' first.  

Install Active Directory Domain Services:  

```powershell
# On your first domain controller (DC0)

Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
# Define a password for safe mode (store this in your password manager)
$Password = Read-Host -Prompt ' => Provide a SafeMode Admin Password' -AsSecureString

# Since this is the first DC in our new domain, we create a new forest:
$params = @{
    CreateDnsDelegation           = $false
    DomainMode                    = 'WinThreshold'
    DomainName                    = 'win.hrkrdr.com'
    DomainNetbiosName             = 'WIN'
    ForestMode                    = 'WinThreshold'
    InstallDns                    = $true
    NoRebootOnCompletion          = $true
    SafeModeAdministratorPassword = $Password
    Force                         = $true    

}    
Install-ADDSForest @params
``` 

Congrats! You now have a shiny new Active Directory forest with one domain controller. Once your domain controller reboots, sign in with your administrator credentials and check it out:  

![Get-ADForest Output](/imgs/2021.03_dirsrv1_getforest.png) 

Using the domain Administrator login for routine tasks is somewhat like using a steam roller to crack cashews. If you look at the group memberships for the built-in administrator login, you'll see what I mean:  

![Administrator Groups Output](/imgs/2021.03_dirsrv1_admingrps.png) 

Let's create a Domain Admin account to use. Often it's a good idea to make certain types of accounts visually identifiable for reporting purposes, for example creating service accounts with a prefix of srv_ as in: srv_mssql. This is a great practice for auditing, so let's create an admin account with that in mind:  

```powershell
# Using a parameter block makes things easier to read
$params = @{
   Name              = 'Brian Admin Account';
   GivenName         = 'Brian';
   SurName           = 'AdminAcct';
   SamAccountName    = 'adm_brian';
   UserPrincipalName = "adm_brian@$env:USERDNSDOMAIN";
   Enabled           = $true;
}

New-ADUser @params -AccountPassword $password
Add-ADGroupMember -Identity 'domain admins' -Members 'adm_brian'
```

Now you've got a more limited access, less generic domain admin account to use moving forward. Change the built-in Administrator logon's password to something long and complex, and save it in a password manager for emergencies only.  

You'll notice there are two GlobalCatalog entries in my screenshot. That's because I've already joined my second DC, which you'll now do: 

### Step 3: Add Second Domain Controller
Log into the second VM and run the sconfig utility and get the VM prepared: 
- Change the computer name to DC1.
- Install all available updates (this takes a bit, again)
- Set a static IP Address and configure the DNS resolvers with DC0's static IP as the first resolver, and 127.0.0.1 as the second resolver. This is important, and will ensure domain functionality in the event that DC0 is unavailable. If you take a look at DC0's ipconfig /all after you promoted it, you'll see the process modified your DNS configuration similarly:  

![DC0 DNS Output](/imgs/2021.03_dirsrv1_dc0dns.png)

Now that your VM is prepped, you can join it to your pristine AD forest:  

```powershell
# Install ADDS features
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Set your domain admin credentials
$Creds = Get-Credential

# Set the safe mode password for the DC
$Password = Read-Host -Prompt ' => Provide a SafeMode Admin Password' -AsSecureString 

# Promote Domain Controller
$params = @{
	InstallDns                    = $true;
    Credential                    = $Creds;
    DomainName                    = 'win.hrkrdr.com';
    SafeModeAdministratorPassword = $Password;
}
Install-ADDSDomainController @params
```

### Step 4: Management VM
Now that we have a working domain, let's get our Jump host joined so we can start using it to manage everything. All of the preparation steps apply (updates, renaming, etc.) but we also need to consider DNS. If we expect clients to hop on the network and be able to enjoy domain services, they need to be able to resolve the domain.  

![Ping domain](/imgs/2021.03_dirsrv1_pingdomain.png) 

Pinging your domain name must always resolve to a domain controller's address. The easiest way to make that happen for your Jump host is to statically assign your two DCs as DNS resolvers on the host. There are lots of other ways to accomplish this depending on your home lab topology. 
One additional consideration is Remote Server Administration Tools. If you're using a Windows Server 2019 VM, installing them is very easy:  

```powershell
Install-WindowsFeature RSAT -IncludeAllSubFeature
```
In Windows 10 1809 and up, you should be able to install them as such: 

```powershell
Get-WindowsCapability -Name RSAT* -Online | Add-WindowsCapability -Online
```

Once you can resolve the domain, go ahead and join it:  

```powershell
# Add the computer to the domain using your new domain admin creds
Add-Computer -DomainName Domain01 -Restart -Credential (Get-Credential)
```

Reboot and log into your new management host.  
If you've followed along you should have:  
- Two functional domain controllers using your own internal domain
- A jump VM with RSAT tools installed to manage everything
- An administrative account (that isn't Administrator)  

In the next section, we'll start populating the domain with users and organize them into Organizational Units.
