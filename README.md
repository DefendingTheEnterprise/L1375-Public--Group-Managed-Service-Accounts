![][DET]

## Sample Lab from Defending the Enterprise 
This lab is a sample lab of nearly 20 hands-on labs from the Defending the Enterprise Course.  

For more information, see [Defending the Enterprise on AntiSyphon Training][DTE-AST]

------

# L1375 - Group Managed Service Accounts
In this lab we will create a group managed service account that can be used to start services on computers that are a member of the associated group.  

For additional information on GMSA's, see
https://docs.microsoft.com/en-us/windows-server/security/group-managed-service-accounts/getting-started-with-group-managed-service-accounts



![Lab Overview][LabOverview]


### Lab Contents

<!-- Start Document Outline -->

* [Step One - Create the required components.](#step-one---create-the-required-components)
	* [Open an Administrative PowerShell session.](#open-an-administrative-powershell-session)
	* [Create a new OU, and group for the GMSA.](#create-a-new-ou-and-group-for-the-gmsa)
* [Step Two - Setup Domain for GMSA](#step-two---setup-domain-for-gmsa)
* [Step Three - Generate the GMSA](#step-three---generate-the-gmsa)
* [Step Four - Using the GMSA](#step-four---using-the-gmsa)
	* [Setup a batch program](#setup-a-batch-program)
	* [Test the batch program](#test-the-batch-program)
	* [Provide Permissions](#provide-permissions)
	* [Reboot](#reboot)
	* [Generate Scheduled Task using GMSA](#generate-scheduled-task-using-gmsa)
* [Step Five - Confirming Success](#step-five---confirming-success)
* [Lab Complete](#lab-complete)

<!-- End Document Outline -->


![Lab Overview][LabMethodology]


--- 

## Step One - Create the required components.
We need to create a new group that will be used to manage and access the GMSA and use it for services.  

### Open an Administrative PowerShell session.

First, lets open an administrative PowerShell session.

![PowerShell Input][PowershellInput]
```powershell
whoami
hostname
```

![PowerShell Output][PowershellOutput]

```powershell
PS C:\Windows\system32> whoami
dteclass\dteadmin
PS C:\Windows\system32> hostname
DTE-DC1
```

### Create a new OU, and group for the GMSA.
Here we will create a new OU called SecurityGroups, followed by a new OU called GMSAGroups within it.   Then we will create a new group called **sec_gmsa_SOXReportingService** that will contain computer objects responsible for running SOXReportingService.

![PowerShell Input][PowershellInput]
```powershell
New-ADOrganizationalUnit -Name "SecurityGroups" -Path "DC=dteclass,DC=com"
New-ADOrganizationalUnit -Name "GMSAGroups" -Path "OU=SecurityGroups,DC=dteclass,DC=com"
New-ADGroup "sec_gmsa_SOXReportingService" -Path "OU=GMSAGroups,OU=SecurityGroups,DC=dteclass,dc=com" -GroupCategory Security -GroupScope DomainLocal -PassThru –Verbose
New-ADComputer -Name "svr_SOXReporter" -SamAccountName "svr_SOXReporter" -Path "OU=ComputerAccounts,DC=dteclass,DC=com"
```
Next, lets add some computers to the new group whose members can use the GMSA.

![PowerShell Input][PowershellInput]
```powershell
ADD-ADGroupMember “sec_gmsa_SOXReportingService” –members “svr_SOXReporter$”
ADD-ADGroupMember “sec_gmsa_SOXReportingService” –members “dte-dc1$”
```

![][NextStep]

---

## Step Two - Setup Domain for GMSA
The domain needs an additional key used for managing GMSA passwords.  Follow the next step to generate the key.  Note that the KDS key cannot be used until after 10 hours it is generated, to allow for full replication across domain controllers.  In a test environment, you can force the effective time to be ten hours prior, eliminating the time requirement.

![PowerShell Input][PowershellInput]
```powershell
Add-KdsRootKey –EffectiveTime ((get-date).addhours(-10))     
```
This will return the GUID created for the KDS key

![PowerShell Output][PowershellOutput]

```powershell
PS C:\Windows\system32> Add-KdsRootKey –EffectiveTime ((get-date).addhours(-10)) 

Guid
----
e68d36d9-fbf5-69a1-0cb2-797c1d25a446
````

![][NextStep]

---

## Step Three - Generate the GMSA
Next, lets create a new OU specifically for GMSAs.

![PowerShell Input][PowershellInput]
```powershell
New-ADOrganizationalUnit -Name "GMSAs" -Path "DC=dteclass,DC=com"
```

Finally, lets create the GMSA.

![PowerShell Input][PowershellInput]
```powershell
New-ADServiceAccount -name gmsa_soxrep -PrincipalsAllowedToRetrieveManagedPassword sec_gmsa_SOXReportingService -path "OU=GMSAs,dc=dteclass,dc=com" -DNSHostName gmsa_soxrep$
```

![][NextStep]

---

## Step Four - Using the GMSA
Our environment only has one computer system, the Domain Controller.  Lets create a batch file and then schedule the batch file to be started by Scheduled Tasks using the GMSA account.  When the scheduled task executes, the computer will procure the password for the account from Active Directory and use it to logon.  

In this case the system accessing the password is also the domain controller, however any system that is a member of of the **sec_gmsa_SOXReportingService** group can use the GMSA account.

### Setup a batch program

First, lets create our batch file.  The batch file created here will simply report back into log.txt the user context and time the batch file was executed.  

![PowerShell Input][PowershellInput]
```powershell
mkdir c:\DTE\GMSASOX
cd c:\DTE\GMSASOX
echo "" | Out-File log.txt -Encoding ASCII
"whoami >>c:\DTE\GMSASOX\log.txt" |Out-File gmsalog.bat -Encoding ASCII
"echo %time% >>c:\DTE\GMSASOX\log.txt" |Out-File gmsalog.bat -Append -Encoding ASCII
```

### Test the batch program
![PowerShell Input][PowershellInput]
```powershell
cd c:\DTE\GMSASOX
./gmsalog.bat
cat log.txt
```
This should produce a result similar to below:

![PowerShell Output][PowershellOutput]
```
PS C:\DTE\GMSASOX> cat .\log.txt
dteclass\dteadmin
12:46:22.29
```

This indicated that the batch file executed at 12:46:22 and the user executing the batch file was dteclass\dteadmin.

### Provide Permissions

Next, lets setup permissions for the GMSA to use the batch script and the log file.

![PowerShell Input][PowershellInput]
```powershell

$acl = Get-Acl c:\DTE\GMSASOX
$AccessRule = New-Object System.Security.AccessControl.FileSystemAccessRule("dteclass\gmsa_soxrep$","FullControl","Allow")
$acl.SetAccessRule($AccessRule)
$acl | Set-Acl c:\DTE\GMSASOX
$acl | Set-Acl c:\DTE\GMSASOX\gmsalog.bat
$acl | Set-Acl c:\DTE\GMSASOX\log.txt
```

### Reboot 
Next, reboot to allow the GMSA authentication to be available.

![PowerShell Input][PowershellInput]

```powershell
Restart-Computer
```

### Generate Scheduled Task using GMSA

Next, lets setup the Scheduled Task to use the new GMSA account.

```
$action = New-ScheduledTaskAction “c:\DTE\GMSASOX\gmsalog.bat”
$trigger = New-ScheduledTaskTrigger -Once -At (Get-Date) -RepetitionInterval (New-TimeSpan -Minutes 1)
$principal = New-ScheduledTaskPrincipal -UserID dteclass\gmsa_soxrep$ -LogonType Password
Register-ScheduledTask -Action $action -Trigger $trigger -TaskName "Test SOX Reporter GMSA" -Description "Runs every minute" -Principal $principal
```

The scheduled task show start and begin logging in C:\DTE\GMSASOX\log.txt


![][NextStep]

---

## Step Five - Confirming Success

Wait a few moments and then view the contents of the file.

![PowerShell Input][PowershellInput]
```powershell
cat c:\DTE\GMSASOX\log.txt
```


Has the GMSA successfully authenticated, executed the batch script, and updated the file?

After reviewing the log.txt file, we can see that the GMSA account, gmsa_soxrep$ is authenticating, executing the batch script, and updating the log file.

![PowerShell Output][PowershellOutput]
```
PS C:\DTE\GMSASOX> cat .\log.txt
dteclass\dteadmin
12:46:22.29
dteclass\gmsa_soxrep$
12:46:28.19
dteclass\gmsa_soxrep$
12:47:07.96
dteclass\gmsa_soxrep$
12:48:07.98
dteclass\gmsa_soxrep$
12:49:07.99
dteclass\gmsa_soxrep$
12:50:08.01
dteclass\gmsa_soxrep$
12:51:08.02
dteclass\gmsa_soxrep$
12:52:08.02
```



---

## Bonus

Take a look at the Task Scheduler and Event Logs.

Note interesting EventIDs: 
- EVENT ID 4624 - An account successfully logged on (NULL SID [SYSTEM] DTE-DC1$ Impersonation)
- EVENT ID 4768 - Kerberos Authentication Ticket Requested
- EVENT ID 4769 - Kerberos Service Ticket Requested
- EVENT ID 4648 - Logon was attemped usign explicit credentials (Note Subject and Account) 
- EVENT ID 4624 - An account successfully logged on (Logon type 5, DTE-DC1$ Impersonation)
- EVENT ID 4672 - Special privileges assigned to new logon.
- Sysmon EVENT ID 1 - Process Creation


## Lab Complete

---

Copyright Defensive Origins 2022


  [LabContents]:https://img.shields.io/badge/Lab-Contents-purple.svg?style=for-the-badge
  [APT-Lab-Terraform]:https://github.com/DefensiveOrigins/APT-Lab-Terraform
  [LabAddendum]:https://img.shields.io/badge/Lab-Addendum-magenta.svg?style=for-the-badge
  [LabOverview]:https://img.shields.io/badge/Lab-Overview-darkblue.svg?style=for-the-badge
  [LabObjectives]:https://img.shields.io/badge/Lab-Objectives-darkblue.svg?style=for-the-badge
  [LabMethodology]:https://img.shields.io/badge/Lab-Methodology-darkblue.svg?style=for-the-badge
  [LabComplete]:https://img.shields.io/badge/Lab-Complete-red.svg?style=for-the-badge
  [NextStep]:https://img.shields.io/badge/Step%20Complete-Onward!-darkgreen.svg?style=flat-sware
  [PowershellInput]:https://img.shields.io/badge/PowerShell-Input-green.svg?style=flat-sware
  [BashInput]:https://img.shields.io/badge/Bash-Input-green.svg?style=flat-sware
  [BashOutput]:https://img.shields.io/badge/Bash-Output-orange.svg?style=flat-sware
  [MSFInput]:https://img.shields.io/badge/MSF-Input-green.svg?style=flat-sware
  [MSFOutput]:https://img.shields.io/badge/MSF-Output-orange.svg?style=flat-sware
  [STInput]:https://img.shields.io/badge/SilentTrinity-Input-green.svg?style=flat-sware
  [STOutput]:https://img.shields.io/badge/SilentTrinity-Output-orange.svg?style=flat-sware
  [HuntIndex]:https://img.shields.io/badge/Hunt-Index%20Term-darkgreen.svg?style=flat-sware
  [HuntSearchTerm]:https://img.shields.io/badge/Hunt-Search%20Term-blue.svg?style=flat-sware
  [PowershellOutput]:https://img.shields.io/badge/PowerShell-Output-orange.svg?style=flat-sware
  [GuiNav]:https://img.shields.io/badge/GUI-Navigation-orange.svg?style=flat-sware
  [StepOne]:https://img.shields.io/badge/Step-One-blue.svg?style=for-the-badge
  [StepTwo]:https://img.shields.io/badge/Step-Two-blue.svg?style=for-the-badge
  [StepThree]:https://img.shields.io/badge/Step-Three-blue.svg?style=for-the-badge
  [StepFour]:https://img.shields.io/badge/Step-Four-blue.svg?style=for-the-badge
  [StepFive]:https://img.shields.io/badge/Step-Five-blue.svg?style=for-the-badge
  [StepSix]:https://img.shields.io/badge/Step-Six-blue.svg?style=for-the-badge
  [AttackStepOne]:https://img.shields.io/badge/Attack-Step%20One-red.svg?style=for-the-badge 
  [AttackStepTwo]:https://img.shields.io/badge/Attack-Step%20Two-red.svg?style=for-the-badge
  [AttackStepThree]:https://img.shields.io/badge/Attack-Step%20Three-red.svg?style=for-the-badge 
  [AttackStepFour]:https://img.shields.io/badge/Attack-Step%20Four-red.svg?style=for-the-badge
  [AttackStepFive]:https://img.shields.io/badge/Attack-Step%20Five-red.svg?style=for-the-badge
  [AttackStepSix]:https://img.shields.io/badge/Attack-Step%20Six-red.svg?style=for-the-badge
  [HuntStepOne]:https://img.shields.io/badge/Hunt-Step%20One-blue.svg?style=for-the-badge
  [HuntStepTwo]:https://img.shields.io/badge/Hunt-Step%20Two-blue.svg?style=for-the-badge
  [HuntStepThree]:https://img.shields.io/badge/Hunt-Step%20Three-blue.svg?style=for-the-badge
  [HuntStepFour]:https://img.shields.io/badge/Hunt-Step%20Four-blue.svg?style=for-the-badge
  [APTStepOne]:https://img.shields.io/badge/APT-Step%20One-purple.svg?style=for-the-badge
  [PurpleTeam]:https://img.shields.io/badge/Team-Purple-purple.svg?style=for-the-badge
  [DO]: https://www.defensiveorigins.com
  [DOTraining]: https://training.defensiveorigins.com
  [DORegister]: https://defensiveorigins.com/first-to-know/
  [DOAboutUs]: https://defensiveorigins.com/about-us
  [WWHF]: https://wildwesthackinfest.com/  
  [APT]:https://github.com/DefensiveOrigins/AtomicPurpleTeam 
  [DTE-L0100]: 2-Labs/DTE-L0100.md
  [DTE-L0110]: 2-Labs/DTE-L0110.md
  [DTE-L0111]: 2-Labs/DTE-L0111.md
  [DTE-L0150]: 2-Labs/DTE-L0150.md
  [DTE-L0151]: 2-Labs/DTE-L0151.md
  [DTE-AST]: https://www.antisyphontraining.com/defending-the-enterprise-w-kent-ickler-and-jordan-drysdale/
  [DET]:resources/img-1.png
