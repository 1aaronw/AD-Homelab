---

# Homelab that runs Active Directory with 1000+ generated users

## Table of contents

- [[00 — Environment Overview]]
- [[01 - Walkthrough]]
- [[02 - Troubleshooting-Log]]

---

---

# [[00 — Environment Overview]]

## Host / Hypervisor

- **Software:** VirtualBox (version: 7.2.12 )
- **Host machine specs:** CPU: Intel i5-13500 (14 Cores, 20 Threads) / RAM: 32 GB DDR4 Memory / Storage: 300 GB NVMe SSD + 2 TB NVMe SSD
- **Networking mode:**
1. Internet-facing NIC -> NAT
	This is to grant internet access for the Windows Server and Windows 11 virtual environments
2. DC Internal NIC + Client's Internal NIC -> Internal Network
	An isolated network is created with this to simulate a virtual private network of clients

## Network Diagram

![ADhomelabdiagram.png](Images/ADhomelabdiagram.png)

(credit: https://www.youtube.com/watch?v=MHsI8hJmggI)
## IP Addressing Scheme

| Subnet                 | Purpose                 | Notes                                                                                     |
| ---------------------- | ----------------------- | ----------------------------------------------------------------------------------------- |
| 172.16.0.0/24          | Internal lab network    | DC's internal NIC is the gateway at 172.16.0.1; DHCP scope hands out .100–.200 to clients |
| DHCP (via home router) | Internet-facing network | Dynamically assigned by home router's DHCP                                                |

## VM Inventory

| VM Name | Role                      | OS          | IP Address                                     | Notes                                                                                 |
| ------- | ------------------------- | ----------- | ---------------------------------------------- | ------------------------------------------------------------------------------------- |
| DC      | Domain Controller         | Server 2022 | DHCP (internet facing) / 172.16.0.1 (internal) | One network adapter for internet access + another for the private network for clients |
| CLIENT1 | Domain-joined workstation | Win 11      | DHCP -- 172.16.0.100-200                       | Private VirtualBox network                                                            |

---

# [[01 — Walkthrough]]

## VirtualBox DC Setup
After installing VirtualBox and the necessary Windows 2022 Server + Windows 11 ISOs, we can continue by setting up the Domain Controller in VirtualBox
>My VirtualBox DC, uses default settings except:
>1. Base memory supports 2048 MB
>2. Changed to support 4 processors
>3. Added a second network adapter for internal network (needed for private client network)
>4. Edited my video memory to be 128 MB
![virtualboxdcsettings.png](Images/virtualboxdcsettings.png)

Mounted Windows 2022 Server ISO and installed Guest Additions via VirtualBox.

## DC NICs Configuration
After initial Windows setup, I immediately renamed the network connections to `INTERNET` and `INTERNAL` to better reflect the intentions of both networks and make them easier to identify

>Went to Ethernet Settings -> Change adapter options.
>- I identified which network's status details displayed an IPv4 address in the form of `10.0.2.x` and renamed it to `Internet`
>- The other is renamed to `Internal`
![nic1.png](Images/nic1.png)

We additionally need to change the Internal's IP address to the one in our diagram, so while on the network connections screen:
> Go to the Internal's properties -> Internet Protocol Version 4's properties -> Use the following IP address where:
> - IP address: `172.16.0.1`
> - Subnet mask: `255.255.255.0`
> - Preferred DNS server: `127.0.0.1`
![nic2.png](Images/nic2.png)

## Active Directory DS Configuration
Now that we have our Internet and Internal NICs, we install active directory domain services to create our domain and serve as centralized management to manage user accounts

In the Server Manager, I access the Add roles and features setting to install Active Directory Domain Services
![ad1.png](Images/ad1.png)

I then promoted the server to a domain controller by adding a new forest and labelling the root domain `mydomain.com`
![ad2.png](Images/ad2.png)

For the rest of the installation, I used the default configs and restarted the PC
![ad3.png](Images/ad3.png)

Now, we will create our own dedicated domain admin account. While we are already on a built-in administrator account, a domain admin account helps track who exactly made the specific changes  and can securely lock and monitor account as a security measure.

So, to do so I used the start menu and went to Active Directory Users and Computers, created a new organizational unit object in mydomain.com named `ADMINS`
![ad4.png](Images/ad4.png)

The ADMINS object will allow us to create users in it, so I will be using myself as the admin account in the form "a-FIRSTINITIALlastname" to better identify that this is the admin account
![ad5.png](Images/ad5.png)

We still need to give access to my user in `ADMINS` to be a Domain Admin, so I add that in my user's "Member of" properties
![ad6.png](Images/ad6.png)

Now, we can sign out and log back in as our new admin account!
![ad7.png](Images/ad7.png)

### Issue #1: [Admin account disabled]

- **Symptom:** I initially couldn't successfully log onto my admin account as it notified that my account was disabled.![is1_1.png](Images/is1_1.png)
- **What I tried:** 
1.  I went to my user's properties and checked `Unlock account`![is1_2](Images/is1_2.png)
2. I went back onto the built-in administrator account and searched for my username in Active Directory Users and Computers to see if this was indeed disabled. After finding my account, I went to my user's properties and enabled the account. 2.![is1_3.png](Images/is1_3.png)
- **Root cause:** The account was disabled from the start
- **Fix:** Enabling the specific user account via Find Users, Contacts, and Groups
- **Lesson learned / how I'd recognize this faster next time: I have to ensure that the account is enabled

## RAS / NAT Configuration
We successfully installed our domain / AD DS. We move onto installing Remote Access Service/Network Address Translation in order to allow our Windows 11 client to be on a private virtual network but still be able to access the internet through the DC

In server manager, we use Add Roles and Features to select Remote Access in Server Roles and Routing in Role Services
![ras1.png](Images/ras1.png)
![ras2.png](Images/ras2.png)

After installed, we move onto Routing and Remote Access via the tools section
![ras3.png](Images/ras3.png)

In here, we can configure the DC to use NAT
![ras4.png](Images/ras4.png)

## DHCP Configuration

We will now set up a DHCP server on our DC so our Windows 11 clients can get an IP address to be able to browse the internet

This will be done through the Add Roles and Features Wizard once again and selecting DHCP Server
![dhcp1.png](Images/dhcp1.png)

We now have DHCP as a tool, so access it and configure IPv4 to have a new scope, and assign the IP addresses based on the DHCP section of our diagram
![dhcp2.png](Images/dhcp2.png)
![dhcp3.png](Images/dhcp3.png)

>Accidentally mistyped IP address, see Issue #2
![dhcp4.png](Images/dhcp4.png)

I finished with the new scope wizard, then authorized dc.mydomain.com so that the IPv4 can go online 
![dhcp5.png](Images/dhcp5.png)

Additionally, I disabled the Internet Explorer Enhanced Security Configuration in Sever Manager's local server settings as to be able to browse the internet through the domain controller 
![dhcp6.png](dhcp6.png)

## PowerShell Script

We want to simulate actual users in this lab, so to do so we will need a PowerShell script that can populate 1000+ user accounts, and a list of generated names in a text file

```powershell
# Script for bulk creating Active Directory user accounts from a names txt file
$PASSWORD_FOR_USERS   = "Password1"
$USER_FIRST_LAST_LIST = Get-Content .\names.txt

$password = ConvertTo-SecureString $PASSWORD_FOR_USERS -AsPlainText -Force

New-ADOrganizationalUnit -Name _USERS -ProtectedFromAccidentalDeletion $false

foreach ($n in $USER_FIRST_LAST_LIST) {
    $first = $n.Split(" ")[0].ToLower()
    $last = $n.Split(" ")[1].ToLower()
    $username = "$($first.Substring(0,1))$($last)".ToLower()
    Write-Host "Creating user: $($username)" -BackgroundColor Black -ForegroundColor Cyan
    
    New-AdUser -AccountPassword $password `
               -GivenName $first `
               -Surname $last `
               -DisplayName $username `
               -Name $username `
               -EmployeeID $username `
               -PasswordNeverExpires $true `
               -Path "ou=_USERS,$(([ADSI]`"").distinguishedName)" `
               -Enabled $true
}
```

Run this script by accessing Windows PowerShell ISE as admin, open the script, and type in `Set-Execution Policy Unrestricted` to allow all scripts to run
>We only do this command in a lab setting as it isn't a very secure approach to allow ALL scripts to run
![ps1.png](Images/ps1.png)

We will need the change the directory to wherever your names.txt file is. Then, run the script!
![ps2.png](Images/ps2.png)

After running the script, we created 1000+ users as seen in our active directory
![ps3.png](Images/ps3.png)

## Client1 Configuration
The only missing piece that is in our diagram to setup is our Windows 11 Client that will access the virtual private network we created.

Create a new VM for our client in VirtualBox
![[cl1.png](Images/cl1.png)

Make sure your network adapter settings are configured to use the internal network instead of NAT so we could get a DHCP address
![cl2.png](Images/cl1.png)
Run it, adding the Windows 11 ISO and installing Windows 11 Pro
![cl3.png](Images/cl3.png)

When on the Windows 11 setup, press Shift + F10 and type into ```OOBE\BYPASSNRO``` so that you can reboot and select the "I don't have internet" option. We will not need actual internet drivers on this PC as we will be accessing internet via the DC
![cl4.png](Images/cl4.png)

Name the PC user and proceed with the setup while disabling optional features

Upon logging into the client account, we can ping `mydomain.com` and see that all packets were sent, indicating that we successfully communicated with the internet
![cl5.png](Images/cl5.png)

We also will rename the PC and set the domain to mydomain.com, so in the system settings from the start menu, click advanced system settings and change the name to `CLIENT1` and set the domain to `mydomain.com`
![cl6.png](Images/cl6.png)

In our DHCP on our DC virtual machine, we now se an address lease for the CLIENT1 PC
![cl7.png](Images/cl7.png)

It shows up as a computer in our Active Directory Users and Computers
![cl8.png](Images/cl8.png)

Also, we can now log in as a user from mydomain. For instance, I can log in as my account on the CLIENT1 PC
![cl9.png](Images/cl9.png)
![cl10.png](Images/cl10.png)


### Issue #2: [Clients can't connect to internet]

- **Symptom:** Testing my ping to the internet seems to time out, indicating something wrong with my DNS server![is2_1.png](Images/is2_1.png)
- **What I tried:** 
1. ipconfig in cmd to ensure I had an IPv4 address, Subnet Mask, Default Gateway 
2. Pinged myself (mydomain.com) to ensure that a response is sent back
3. Removing my initial IP address in DHCP's IPv4 addresses and adding the correct IP address according to the diagram
- **Root cause:** The IP address I assigned in the DHCP was 172.168.0.1 instead of 172.16.0.1
- **Fix:** On my domain admin account, I went to DHCP in Server Manager and went through the properties of IPv4's Server Options to add a Router with `172.16.0.1` as the IP address and removed `172.168.0.1`. Restart dc.mydomain.com in DHCP and ping google.com to see sent/received packets ![is2_2.png](Images/is2_2.png)
- **Lesson learned / how I'd recognize this faster next time: If DHCP told it the gateway is 172.168.0.1, that address doesn't exist anywhere on its actual subnet, so the packet has nowhere to go and dies immediately. I can immediately identify this is an issue regarding my IP assignment based off of my pings actually sending but simply timing out

---


# 02 — Troubleshooting Log

## Issue #1: [Admin account disabled]

- **Symptom:** I initially couldn't successfully log onto my admin account as it notified that my account was disabled.![is1_1.png](Images/is1_1.png)
- **What I tried:** 
1.  I went to my user's properties and checked `Unlock account`![is1_2.png](Images/is1_2.png)
2. I went back onto the built-in administrator account and searched for my username in Active Directory Users and Computers to see if this was indeed disabled. After finding my account, I went to my user's properties and enabled the account. 2.![is1_3.png](Images/is1_3.png)
- **Root cause:** The account was disabled from the start
- **Fix:** Enabling the specific user account via Find Users, Contacts, and Groups
- **Lesson learned / how I'd recognize this faster next time: I have to ensure that the account is enabled

## Issue #2: [Clients can't connect to internet]

- **Symptom:** Testing my ping to the internet seems to time out, indicating something wrong with my DNS server![is2_1.png](Images/is2_1.png)
- **What I tried:** 
1. ipconfig in cmd to ensure I had an IPv4 address, Subnet Mask, Default Gateway 
2. Pinged myself (mydomain.com) to ensure that a response is sent back
3. Removing my initial IP address in DHCP's IPv4 addresses and adding the correct IP address according to the diagram
- **Root cause:** The IP address I assigned in the DHCP was 172.168.0.1 instead of 172.16.0.1
- **Fix:** On my domain admin account, I went to DHCP in Server Manager and went through the properties of IPv4's Server Options to add a Router with `172.16.0.1` as the IP address and removed `172.168.0.1`. Restart dc.mydomain.com in DHCP and ping google.com to see sent/received packets ![is2_2.png](Images/is2_2.png)
- **Lesson learned / how I'd recognize this faster next time: If DHCP told it the gateway is 172.168.0.1, that address doesn't exist anywhere on its actual subnet, so the packet has nowhere to go and dies immediately. I can immediately identify this is an issue regarding my IP assignment based off of my pings actually sending but simply timing out


---

