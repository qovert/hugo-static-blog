---
title: "Domain Services Part 2"
date: 2021-04-02T13:58:50-07:00
draft: false
image: "/imgs/2021.03_random_user_script.png"
categories: ['home lab']
tags: ["windows server", "active directory"]
---

Ok, now we've got a big empty domain. What to do next?   

### Structure
Let's add some logical structure so we can target objects better. By default every user object in AD ends up in CN=Users,DC=win,DC=hrkrdr,DC=com. There are already built in objects that live there, such as Administrator and some security groups. In order to start granular targeting or messing with delegation we want some Organizational Units.  

### Build OUs
Before we build a ton of new objects into AD, let's enable the [Active Directory Recycle Bin](https://techcommunity.microsoft.com/t5/ask-the-directory-services-team/the-ad-recycle-bin-understanding-implementing-best-practices-and/ba-p/396944). We can use this to recover objects if they're mistakenly removed, which can become very important. This is always a good idea on production domains.

```powershell
Enable-ADOptionalFeature 'Recycle Bin Feature' -Scope ForestOrConfigurationSet -Target $env:USERDNSDOMAIN
```

With that in place, lets begin building the Organizational Unit structure to our needs: 

```powershell
# These are just a few example names. Feel free to add as many or few as you need. 

$newOUs = @('Staff','IT','Service Accounts','Servers','Staff PCs')
foreach ($ou in $newous) {
    New-ADOrganizationalUnit -Name $ou
}
```

Success:
![Image of Default OUs](/imgs/2021.03_default_ous.png)

### Populate the domain
Now let's create a ton of user accounts. Having fake accounts that look like real user objects will make it much more interesting than an bunch of 'User1' 'User2' etc. output. To automate this, I've written a quick script that pulls enough data to create 500 random users from [randomuser.me's](https://randomuser.me/) excellent API. 

```powershell
# Make a request to randomuser.me's API for CSV data and save it to a file for reference.

if (!(Test-Path ~\Downloads\users.csv)) {
    Invoke-RestMethod -Uri 'https://randomuser.me/api/?format=csv&results=500&password=upper,lower,number,special' -OutFile '~\Downloads\users.csv'
    Start-Sleep -Seconds 5 
}

$userData = Import-Csv '~\Downloads\users.csv'

# Actual creation process 
foreach ($user in $userData) {
    $mainparams = @{
        SamAccountName = $user."login.username";
        AccountPassword = $user."login.password" | ConvertTo-SecureString -AsPlainText -Force;
        GivenName = $user."name.first";
        Surname = $user."name.last";
        DisplayName = $user."name.first" + " " + $user."name.last";
        Name = $user."name.first" + " " + $user."name.last";
        EmailAddress = $user.email;
        City = $user."location.city";
        Country = $user.nat;
        PostalCode = $user."location.postcode";
        OfficePhone = $user.phone;
        MobilePhone = $user.cell;
        State = $user."location.state";
        StreetAddress = $user."location.street.number" + " " + $user."location.street.name";
        UserPrincipalName = $user."name.first" + "." + $user."name.last" + "@$env:USERDNSDOMAIN";
        PasswordNeverExpires = $true;
        Enabled = $true;
    }
    New-ADUser @mainparams -Verbose
}
```

The user data includes a ton of fun things like location data, passwords, email info, etc. This will take a few moments to run. When it's complete, let's verify we've got new user objects: 

![New-ADUser output](/imgs/2021.03_new_aduser.png)

### User organization
Now is a great time to realize that you've just dumped 500 new users into the ***CN=Users*** container. This isn't very readable, nor is it especially useful for management. Optimally, we want them to be in the Staff OU we created above, which we can set as default for future user additions using a command line tool:

```powershell
# Setting OUs as default uses two command line tools: 

# For user objects
redirusr "OU=Staff,DC=hrkrdr,DC=com"

# For computer objects
redircmp "OU=Staff PCs,DC=hrkrdr,DC=com"
```

I think it would be more useful to have them separated further. Based on the fields we got back from our random generator, I'm going to divide everyone by the country they work in:

```powershell
# First let's see which countries we do business in
Get-ADUser -Filter * -Properties * | Select -Unique Country

country
-------

ES
US
BR
TR
CA
FR
NL
DK
FI
IE
GB
NO
AU
IR
NZ
CH
DE

# Now create the OUs
$country = Get-ADUser -Filter * -Properties * | Select -Unique Country

foreach ($i in $country.Country) { 
	New-ADOrganizationalUnit -Name $i -Path 'OU=staff,DC=win,DC=hrkrdr,DC=com'
 }
 ```

Ok, now that we've got our country specific OUs, we can move users around with that same $country variable we already populated:

```powershell
foreach ($ou in $country.Country){
    if ($ou) {
        Get-ADUser -Filter {Country -eq $ou} | `
        Move-ADObject -TargetPath (Get-ADOrganizationalUnit -Filter {name -eq $ou}) 
    }
}
```

Now if we look in Active Directory Users and Computers, we'll see all the user objects have been appropriately sorted: 

![ADUC Screenshot](/imgs/2021.03_aduc_screenshot.png)

You can see how powerful a tool powershell is. Having to make any of these modifications in the GUI would be incredibly tedious and error prone. 
Now that we have a working list of users, we can add new member servers, build reports on the health of our Active Directory system, and set up some backups.
