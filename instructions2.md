# Implementing Microsoft Security Technologies (AIP/MCAS/Azure ATP)

## INSTRUCTOR LED LAB - M365HOL2

### Introduction

This lab is designed to be used as a supplement to Instructor Led Training and has several sections that you will go through over the next few days. Please click the lab below that corresponds to the technology your are working with.  

## [Lab Environment Configuration](#lab-environment-configuration)

## [Lab 1: Azure Information Protection](#azure-information-protection-lab)

## [Lab 2: Azure Advanced Threat Protection](#azure-atp-immersion-lab)

## [Lab 3: Microsoft Cloud App Security](#microsoft-cloud-app-security)

> [!ALERT] When stopping each section, please ensure that you SAVE the session in between labs rather than END the lab.  If you end the lab, all VM configuration will be reset to initial state and will hinder the experience during future labs.  We have designed this lab to be a good representation of the interoperability between Microsoft 365 Security Technologies so several of the labs will feed information into future labs.

===
# Lab Environment Configuration
[:arrow_left: Home](#introduction)

There are a few prerequisites that need to be set up to complete all the sections in this lab.  This Exercise will walk you through the items below.

- [Redeem Azure Pass](#redeem-azure-pass)

- [Azure AD User Configuration](#azure-ad-user-configuration)

- [Workplace Join Clients](#workplace-join-clients)

- [MCAS Environment Proparation](#mcas-environment-preparation)
  
- [Azure Advanced Threat Protection Setup](#azure-advanced-threat-protection-setup)

---

## Redeem Azure Pass
[:arrow_left: Home](#lab-environment-configuration)

For several of the exercises in this lab series, you will require an active subscription.  We are providing an Azure Pass for this purpose.  You will be provided with an Azure Pass code to use with the instructions below.

### Redeeming a Microsoft Azure Pass Promo Code:

2. [] Right-click on **Edge** in the taskbar and click on **New InPrivate window**.

3. [] In the InPrivate window, navigate to ```https://www.microsoftazurepass.com```

4. [] Click the **Start** button to get started.

	> !IMAGE[wdir7lb3.jpg](\Media\wdir7lb3.jpg)
1. [] Log in using the credentials below.

	```@lab.CloudCredential(139).Username```

	```@lab.CloudCredential(139).Password``` 

1. [] Click **Confirm** if the correct email address is listed.

	> !IMAGE[teyx280d.jpg](\Media\teyx280d.jpg)
7. [] Click in the Promo code box and type ```@lab.CloudCredential(215).PromoCode```, then click the **Claim Promo Code** button.

	> !IMAGE[e1l35ko2.jpg](\Media\e1l35ko2.jpg)

	>[!NOTE] It may take up to 5 minutes to process the redemption.

8. [] Scroll to the bottom of the page and click **Next**.

    > !IMAGE[ihrjazqi.jpg](\Media\ihrjazqi.jpg)

	>[!NOTE] You can keep the pre-populated information.

9. [] Check the box to agree to the terms and click **Sign up**.

	> !IMAGE[k2a97g8e.jpg](\Media\k2a97g8e.jpg)

	> [!NOTE] It may take a few minutes to process the request.
---

## Azure AD User Configuration
[:arrow_left: Home](#lab-environment-configuration)

In this task, we will create new Azure AD users and assign licenses via PowerShell.  In a procduction evironment this would be done using Azure AD Connect or a similar tool to maintain a single source of authority, but for lab purposes we are doing it via script to reduce setup time.

1. [] Log into @lab.VirtualMachine(Scanner01).SelectLink using the password +++@lab.VirtualMachine(Client01).Password+++
2. [] On the desktop, **right-click** on **AADConfig.ps1** and click **Run with PowerShell**.

	!IMAGE[AADConfig](\Media\AADConfig.png)

	> [!NOTE] If prompted to change the execution policy, type **y** and **Enter**.

1. [] When prompted for the **Tenant name**, **click in the text box** and enter ```@lab.CloudCredential(134).TenantName```.
1. [] When prompted, provide the credentials below:

	```@lab.CloudCredential(134).Username```

	```@lab.CloudCredential(134).Password``` 
   
	> [!KNOWLEDGE] We are running the PowerShell code below to create the accounts and groups in AAD and assign licenses for EMS E5 and Office E5. This script is also available at [https://aka.ms/labscripts](https://aka.ms/labscripts) as AADConfig.ps1.
    > 
    > #### Azure AD User and Group Configuration
    > $tenantfqdn = "@lab.CloudCredential(134).TenantName"
    > $tenant = $tenantfqdn.Split('.')[0]
	> 
    > #### Build Licensing SKUs
    > $office = $tenant+":ENTERPRISEPREMIUM"
    > $ems = $tenant+":EMSPREMIUM"
	> 
    > #### Connect to MSOLService for licensing Operations
    > Connect-MSOLService -Credential $cred
	> 
    > #### Remove existing licenses to ensure enough licenses exist for our users
    > $LicensedUsers = Get-MsolUser -All  | where {$_.isLicensed -eq $true}
    > $LicensedUsers | foreach {Set-MsolUserLicense -UserPrincipalName $_.UserPrincipalName -RemoveLicenses $office, $ems}
	> 
    > #### Connect to Azure AD using stored credentials to create users
    > Connect-AzureAD -Credential $cred
	> 
    > #### Import Users from local csv file
    > $users = Import-csv C:\users.csv
	> 
    > foreach ($user in $users){
    > 	
    > #### Store UPN created from csv and tenant
    > $upn = $user.username+"@"+$tenantfqdn
	> 
    > #### Create password profile preventing automatic password change and storing password from csv
    > $PasswordProfile = New-Object -TypeName Microsoft.Open.AzureAD.Model.PasswordProfile 
    > $PasswordProfile.ForceChangePasswordNextLogin = $false 
    > $PasswordProfile.Password = $user.password
	> 
    > #### Create new Azure AD user
    > New-AzureADUser -AccountEnabled $True -DisplayName $user.displayname -PasswordProfile $PasswordProfile -MailNickName $user.username -UserPrincipalName $upn
    > }
    > 
    > #### MCAS user and group creation
	> $upn = "mcasAdminUS@"+$tenantfqdn
	> New-AzureADUser -AccountEnabled $True -DisplayName "MCAS US admin" -PasswordProfile $PasswordProfile -MailNickName "mcasadminUS" -UserPrincipalName $upn
    > New-AzureADGroup -DisplayName "US employees" -MailNickName "USemployees" -SecurityEnabled $true -MailEnabled $false
    > $groupId = Get-AzureADGroup -SearchString "usemployees"
    > $userId = Get-AzureADUser -SearchString "mcasadminus"
    > Add-AzureADGroupMember -RefObjectId $userId.ObjectId -ObjectId $groupId.ObjectId
	> 
	> Start-Sleep -s 10
	> foreach ($user in $users){
	> 
    > #### Store UPN created from csv and tenant
    > $upn = $user.username+"@"+$tenantfqdn
	> 
    > #### Assign Office and EMS licenses to users
    > Set-MsolUser -UserPrincipalName $upn -UsageLocation US
    > Set-MsolUserLicense -UserPrincipalName $upn -AddLicenses $office, $ems
    > }
	> 
    > #### Assign Office and EMS licenses to Admin user
    > $upn = "admin@"+$tenantfqdn
    > Set-MsolUser -UserPrincipalName $upn -UsageLocation US
    > Set-MsolUserLicense -UserPrincipalName $upn -AddLicenses $office, $ems

	> [!NOTE] The PowerShell window will automatically close once users have been created and licenses have been assigned

---

## Azure Security Center Setup
[:arrow_left: Home](#lab-environment-configuration)

### VM and Workspace Deployment

This section is intended to deploy dependency Azure resources in an automated way to get you started quickly or in case you need to re-provision your environment.

***Additional notes**:*
- Close the Azure Advisor recommendations if that opens for you.
- When deploying the template below, a number of fields are locked to ensure that the references in the labs are accurate. You are free to choose your location for deployment

#### Deploy VMs and Create a Workspace
Use the link below to deploy the following:
- A Log Analytics workspace (ASC-Workspace-xxxx)
- Security Center OMS solution
- 2 Windows VM's (*Win-x*, connected to the workspace)
- 2 Linux Ubuntu VM's (*Linux-x*, connected to the workspace) 

>[!NOTE] Because we are deploying the Security Center solution in the workspace, the VM's will become security managed by Security Center.

1. [] Switch to @lab.VirtualMachine(Client01).SelectLink and log in with the password +++@lab.VirtualMachine(Client01).Password+++.
1. [] Open a new tab and navigate to ```https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Ftianderturpijn%2FAzure-Security-Center%2Fmaster%2FLabs%2F01%2520-%2520VM%2520and%2520Workspace%2520Deployment%2FFiles%2FdeployAscManagedVmsWithLA.json```

	> [!KNOWLEDGE] If needed, log in using the credentials below:
	>
	>```@lab.CloudCredential(134).Username```
	>
	>```@lab.CloudCredential(134).Password```

1. [] On the Custom deployment page, below the **Resource group** drop-down, click **Create new**.

	!IMAGE[NewRG.png](\Media\NewRG.png)
1. [] When prompted for a Name, provide a name such as ```ASC-Labs``` and press **OK**.

	^IMAGE[Open Screenshot](\Media\RGName.png)
1. [] Under Location, choose **East US**.

	> [!ALERT] Ensure that you have chosen East US (or a datacenter near the event) or the workspace will take longer to deploy and the VMs will have latency.

1. [] **Select a location**, then under **Settings**, next to **Pwd Or Ssh** type the password ```Securitycenter4ever!```.
1. [] Finally, check the Box to agree to the terms and click **Purchase**.

	^IMAGE[Open Screenshot](\Media\Password.png)

1. [] The deployment takes about **13 minutes**. **Continue to the next task** and we will return to the ASC deployment later.

---

## Windows Defender ATP Onboarding
[:arrow_left: Home](#lab-environment-configuration)

In this task, we will perform initial setup of WD ATP and onboard 2 machines.  

1. [] Switch to @lab.VirtualMachine(AdminPC).SelectLink and log in with the password +++@lab.VirtualMachine(AdminPC).Password+++.

1. [] Open an Edge InPrivate window and navigate to ```https://securitycenter.windows.com```.
1. [] Log in using the credentials below:

	```@lab.CloudCredential(134).Username```

	```@lab.CloudCredential(134).Password```

1. [] On Step 1, click **Next**.
1. [] On Step 2, choose a data storage location and click **Next**.
1. [] On Step 3, click **Next** several times until the Create your cloud instance dialog pops up, then click **Continue**.
1. [] On Step 4, click the **Download package** button and save the package to your desktop.
1. [] **Extract the zip file** to your desktop.
1. [] Right-click on **WindowsDefenderATPLocalOnboardingScript** and click **Run as Administrator**.
1. [] In the Windows protected your PC dialog, click the **More info** link and click **Run anyway**.
1. [] In the User Account Control (UAC) window, click on **More choices** and select use a different account.

	> [!NOTE] If you do not see the UAC window, minimize all windows and it will be in the background.
1. [] Enter the credentials below and click **Yes**:

	```LabUser```

	```Pa$$w0rd```

1. [] Press **(Y)** to confirm onboarding.
2. [] Return to the browser, scroll to the bottom of the page, and click **Start using Windows Defender ATP**.
1. [] Click **Proceed anyway** if you receive a Setup incomplete popup.
3. [] In the Windows Defender Security Center, click on **Settings > Advanced Features** and toggle the switches on for **Azure ATP integration**, **Office 365 Threat Intelligence Connection**, **Microsoft Cloud App Security**, and **Azure Information Protection** and press the **Save preferences** button at the bottom of the page.
	
	!IMAGE[g47p8c30.jpg](\Media\wdatpadv.png)

1. [] Switch to @lab.VirtualMachine(VictimPC).SelectLink and log in with the password +++@lab.VirtualMachine(VictimPC).Password+++.

1. [] Open an Edge InPrivate window and navigate to ```https://securitycenter.windows.com/```.
1. [] Log in using the credentials below:

	```@lab.CloudCredential(134).Username```

	```@lab.CloudCredential(134).Password```
1. [] On the left, click the **Settings** icon.

	!IMAGE[settings.png](\Media\settings.png)

3. [] On the Settings page, under **Machine management**, click **Onboarding**.
   
	!IMAGE[Open Screenshot](\Media\onboarding.png)

1. [] Click **Download package** and **Open** when the download dialog pops up.
2. [] Copy the **WindowsDefenderATPLocalOnboardingScript** to the desktop.
3. [] Right-click on **WindowsDefenderATPLocalOnboardingScript** and click **Run as Administrator**.
1. [] If you get the Windows protected your PC dialog, click the **More info** link and click **Run anyway**.
4. [] Press **(Y)** to confirm onboarding.
5. [] To run **Attack Simulation #1 "Automated investigation (fileless attack)"**, right-click **on AttackSimulation1.ps1** on the desktop, and click **Run with PowerShell**.

	> [!NOTE] A notepad window will launch to show that the attack was successful. Leave the notepad window open and continue with the lab.
	
	> [!KNOWLEDGE] The powershell commands that are included in the ps1 file are in the block below.
	> 
	> \[Net.ServicePointManager\]::SecurityProtocol = \[Net.SecurityProtocolType\]::Tls12;$xor = \[System.Text.Encoding\]::UTF8.GetBytes('WinATP-Intro-Injection');$base64String = (Invoke-WebRequest -URI https://winatpmanagement.windows.com/client/management/static/WinATP-Intro-Fileless.txt -UseBasicParsing).Content;Try{ $contentBytes = \[System.Convert\]::FromBase64String($base64String) } Catch { $contentBytes = \[System.Convert\]::FromBase64String($base64String.Substring(3)) };$i = 0; $decryptedBytes = @();$contentBytes.foreach{ $decryptedBytes += $_ -bxor $xor[$i]; $i++; if ($i -eq $xor.Length) {$i = 0} };Invoke-Expression (\[System.Text.Encoding\]::UTF8.GetString($decryptedBytes))

1. [] Switch to @lab.VirtualMachine(AdminPC).SelectLink and log in with the password +++@lab.VirtualMachine(AdminPC).Password+++.
1. [] Run **Attack Simulation #2 "Automated investigation (backdoor)"** by following the instructions below:
	1. [] On the desktop, double-click on **RS4_WinATP-Intro-Invoice.docm** and enter ```WDATP!diy#``` when prompted for a password.

		> [!ALERT] If you receive an activation warning, click **Cancel**.

	2. [] Once the file opens, in the Security Warning ribbon at the top of the document, click **Enable Content**.
	3. [] Click **OK** to confirm the attack.
	4. [] Wait about a minute, then **close the generated command prompt** window.

---

## Workplace Join Clients
[:arrow_left: Home](#lab-environment-configuration)

In this task, we will join 3 systems to the Azure AD tenant to provide SSO capabilities in Office.

1. [] Log into @lab.VirtualMachine(Client01).SelectLink using the password +++@lab.VirtualMachine(Client01).Password+++.
1. [] Right-click on the start menu and click **Run**.
1. [] In the Run dialog, type ```ms-settings:workplace``` and click **OK**.

	>!IMAGE[mssettings.png](\Media\mssettings.png)

1. [] In the Access Work or School settings menu, click on **+ Connect** and enter the credentials below to workplace join the client.

	```adams@@lab.CloudCredential(134).TenantName```
		
	```pass@word1```
1. [] Click **Done**.

1. [] Log into @lab.VirtualMachine(Client02).SelectLink using the password +++@lab.VirtualMachine(Client01).Password+++.
1. [] Right-click on the start menu and click **Run**.
1. [] In the Run dialog, type ```ms-settings:workplace``` and click **OK**.

	^IMAGE[Open Screenshot](\Media\mssettings.png)

1. [] In the Access Work or School settings menu, click on **+ Connect** and enter the credentials below to workplace join the client.

	```alicea@@lab.CloudCredential(134).TenantName```
		
	```pass@word1```
1. [] Click **Done**.

1. [] Log into @lab.VirtualMachine(Client03).SelectLink using the password +++@lab.VirtualMachine(Client01).Password+++.
1. [] Right-click on the start menu and click **Run**.
1. [] In the Run dialog, type ```ms-settings:workplace``` and click **OK**.

	^IMAGE[Open Screenshot](\Media\mssettings.png)

1. [] In the Access Work or School settings menu, click on **+ Connect** and enter the credentials below to workplace join the client.

	```evang@@lab.CloudCredential(134).TenantName```
		
	```pass@word1```
1. [] Click **Done**.

---

## MCAS Environment Preparation
[:arrow_left: Home](#lab-environment-configuration)

To be able to complete the different parts of the Cloud App Security labs, the following configuration steps are required.

* [Enabling Office 365 Auditing](#enabling-office-365-auditing)
* [Enabling File Monitoring](#enabling-file-monitoring)
* [Create a Developer Box Account](#create-a-developer-box-account)
* [Connect Office 365 and Box to Cloud App Security](#connect-office-365-and-box-to-cloud-app-security)
* [Enabling Azure Information Protection integration](#enabling-azure-information-protection-integration)

---

### Enabling Office 365 Auditing
[:arrow_up: Top](#mcas-environment-preparation)

Most Cloud App Security treat detections capabilities rely on auditing being enabled in your environment. By default, auditing is not enabled in Office 365 and must then be turned on using the **Security & Compliance** admin console or PowerShell.

> [!ALERT] As this operation can take up to 24h, your instructor will provide you access to another environment to review the alerts for the threat detection lab.

1. [] Switch to @lab.VirtualMachine(Client01).SelectLink and log in with the password +++@lab.VirtualMachine(Client01).Password+++.
2. [] Open a new InPrivate tab and navigate to ```https://protection.office.com```.	
	
	> [!KNOWLEDGE] If needed, log in using the credentials below:
	>
	>```@lab.CloudCredential(134).Username```
	>
	>```@lab.CloudCredential(134).Password```

4. [] In the **Security & Compliance Center**, Expand **Search & investigation** and click on **Audit log search**.
   
	^IMAGE[Audit log](\Media\conf-auditlog.png "Audit log")

5. [] You can see here that auditing is not enabled. Click on the **Turn on auditing** button to enable it and click **yes** at the prompt.

    ^IMAGE[Turn on auditing](\Media\conf-enableauditing.png "Turn on on auditing")

    ^IMAGE[Auditing enabled](\Media\conf-auditenabled.png "Auditing enabled")

---    

#### Exchange Auditing Configuration
[:arrow_up: Top](#mcas-environment-preparation)

In addition to enabling auditing in Office 365, some applications like Exchange Online require extra configuration. After enabling auditing at the Office 365 level, we have to enable auditing at the mailBox level. We will perform this configuration before going to the labs.

1. [] On the desktop, right-click on **EnableMailboxAudit.ps1** and click **Run with PowerShell**.

1. [] When prompted, use the credentials below:

	+++@lab.CloudCredential(134).Username+++

	+++@lab.CloudCredential(134).Password+++

	> [!KNOWLEDGE] The following commands will be run to connect to Exchange Online and Enable Mailbox Auditing on the Admin account.
    >
	> ```$UserCredential = Get-Credential```
    >
	> ```$Session = New-PSSession –ConfigurationName Microsoft.Exchange –ConnectionUri https://outlook.office365.com/powershell-liveid/ -Credential $UserCredential –Authentication Basic -AllowRedirection```
	>
	>```Import-PSSession $Session```
    >
	> ```Get-MailBox -ResultSize Unlimited -Filter {RecipientTypeDetails -eq "UserMailBox"} | Set-MailBox -AuditEnabled $true```
    >
	> ```Get-MailBox admin | fl audit*```
    
    > [!ALERT] When you create new mailBoxes, **auditing is not enabled** by default. You will have to use the same commands again to enable auditing for those newly created mailBoxes.

    !IMAGE[MailBox auditing](\Media\MailboxAudit.png "MailBox Auditing")

>:memo: **Reference:** [Enabling auditing for Exchange Online mailBoxes](https://docs.microsoft.com/en-us/office365/securitycompliance/enable-mailBox-auditing?redirectSourcePath=%252fen-us%252farticle%252fenable-mailBox-auditing-in-office-365-aaca8987-5b62-458b-9882-c28476a66918).

---

### Enabling File Monitoring
[:arrow_up: Top](#mcas-environment-preparation)

1. [] Go to Cloud App Security portal at ```https://portal.cloudappsecurity.com``` and click on the **Gear** and then **Settings**.

    !IMAGE[Settings](\Media\conf-settings.png "Settings")

1. [] Under the **Information Protection** section click on **Files** and check the **Enable file monitoring** checkbox and click on the "**Save** button.

    !IMAGE[Enable files](\Media\conf-files.png "Enable files")

---

### Create a Developer Box Account
[:arrow_up: Top](#mcas-environment-preparation)

1. [] Next, navigate to ```https://developer.box.com``` and click on **Get Started**. 

	!IMAGE[Boxdev](\Media\box-getstarted.png)

2. [] **Enter the values** from the table below, **check the box** to solve the captcha, and click **Submit**.

	|||
	|-----|-----|
	|**Full Name**|```MOD Admin```|
	|**Email Address**|```@lab.CloudCredential(134).UserName```|

	^IMAGE[Open Screenshot](\Media\box-signup.png)

3. [] In a new tab, browse to ```https://outlook.office365.com/OWA```. 
1. [] Choose a time zone and click **Save**.
1. [] In the MOD Admin inbox, click on **Other** mail, and click the **Verify Email** link in the email from Box.

	^IMAGE[Open Screenshot](\Media\box-verify.png)

1. [] In the new window that opens, enter ```@lab.CloudCredential(134).password``` in **each of the password boxes** and click the **Update** button. 

---

### Connect Office 365 and Box to Cloud App Security 
[:arrow_up: Top](#mcas-environment-preparation)

To connect Cloud App Security to Office 365, you will have to use the Office 365 app connector. App connectors use the APIs of app providers to enable greater visibility and control by Microsoft Cloud App Security over the apps you connect to.  We will also use this method to show integration with the 3rd Party API for Box.

1. [] Open a new in Private tab in your browser and navigate to ```https://portal.cloudappsecurity.com```

2. [] Go to the gear icon and select **App connectors**.

    !IMAGE[App connector](\Media\conf-appconnector.png "App connector")

3. [] Click on the **+** button and select Office 365.

    !IMAGE[Add Office](\Media\conf-addoffice.png "Add Office")

4. [] Click on **Connect Office 365**. Cloud App Security will then have access to Office 365 activities and files.

    ^IMAGE[Open Screenshot](\Media\conf-connectoffice.png "Connect Office")

5. [] Click on **Test now** to validate the configuration.

    ^IMAGE[Open Screenshot](\Media\conf-testoffice.png "Test connectivity")

	> [!NOTE] If the connection is taking more than 1 minute - close out of the portal and log back in to check if it's been connected. 

---

### Connecting Box to Cloud App Security
[:arrow_up: Top](#mcas-environment-preparation)

1. []  Click on the **+** button again, and this time click on **Box**.

	!IMAGE[2](\Media\box-connect.png)

3. [] In the Instance name box, type ```Box API Demo```, and click **Connect Box**.

	^IMAGE[Open Screenshot](\Media\apiBox3.JPG)

4. [] In the Connect Box dialog, click **follow this link**.

	!IMAGE[4](\Media\box-follow.png)

5. [] Log into Box using the credentials below:

	```@lab.CloudCredential(134).Username```

	```@lab.CloudCredential(134).Password```

6. [] Click on **Grant access to Box**

	^IMAGE[Open Screenshot](\Media\box-grant.png)

7. [] Close the Connect Box dialog and click on **Box API Demo** to expand.
1. [] Click on the **Test now** link.

	^IMAGE[Open Screenshot](\Media\apiBox7.JPG)

	> [!KNOWLEDGE] Once the connection is succesful - it will say **Connected.** 
	>
	> !IMAGE[8](\Media\apiBox8.JPG)

8. []  Close the dialog and you should be able to see **Box API Demo** as a **Connected** app in the list. 

	^IMAGE[Open Screenshot](\Media\apiBox9.JPG) 

---

### Enabling Azure Information Protection integration
[:arrow_up: Top](#mcas-environment-preparation)

To prepare the **Information Protection** lab, we have to enable the integration between Cloud App Security and Azure Information Protection as explained in the [Cloud App Security documentation](https://docs.microsoft.com/en-us/cloud-app-security/azip-integration). Enabling the integration between the two solutions is as easy as selecting one single checkBox.

1. [] Go to Cloud App Security settings.

    !IMAGE[Settings](\Media\conf-settings.png "Settings")

2. [] Go down in the settings to the **Azure Information Protection** section and check the **Automatically scan new files** checkBox and click on the "**Save** button.
    !IMAGE[Enable AIP](\Media\conf-aip.png "Enable AIP")

>:memo: It takes up to **1h** for Cloud App Security to sync the Azure Information classifications.

---

## Azure Advanced Threat Protection Setup
[:arrow_left: Home](#lab-environment-configuration)

### Create and configure Azure ATP Workspace 

1. []	Login into @lab.VirtualMachine(ContosoDC).SelectLink using the password	+++@lab.VirtualMachine(ContosoDC).Password+++ 

2. []	Open Internet Explorer and browse to ```https://portal.atp.azure.com``` and login with the following credentials.  
   
   ```@lab.CloudCredential(134).UserName``` 

   ```@lab.CloudCredential(134).Password``` 
   
6. []	Click **Create**. 
8. []	Click **Provide a username and password to connect to your Active Directory forest**.  
9. []	On the Directory Services page enter the following and click **Save**: 

|||
|-----|-----|
|Username|```aatpservice```|
|Password|```Password123!@#```|
|Domain|```contoso.azure```|

---
### Deploy the Azure ATP Sensor  

1. []	Scroll up and click the **Download Sensor Setup** link. 
1. []   Click  **Download** to download the Sensor installer package. 
1. []   Copy the **Access key**, this will be needed during the installation of the Sensor. 
1. []   **Extract the installation files** from the Zip file and run **Azure ATP sensor setup.exe**. 

	>[!NOTE] Do not run the installer from within the Zip file, you need to extract the files before running the installer.

1. []	Click **Run** in the Open File Security Warning page. 
1. []	Select the installation language of choice and click **Next**. 
1. []	Click **Next** on the Sensor deployment type page.  
1. []	**Paste the Access key** copied from above and click **Install**.  
1. []   Click **Finish** to complete the installation.

---

### Configure Domain Synchronizer 
1. []	In the Azure ATP console **click on the deployed Sensor (ContosoDC)** and **toggle the Domain synchronizer candidate switch** to **On** and click **Save**. 

## Configure Windows Defender ATP Integration 
1. [] In the Azure ATP console click **Windows Dender ATP** and then toggle the **Integration with Widnows Defender ATP** to **On** and click **Save**

	>[!NOTE] This requires that you have already enabled the Windows Defender ATP service. 

---

### Adding Guest User access to Azure ATP Console.
[:arrow_left: Home](#lab-environment-configuration)  

To allow users not in the companies Azure Active Directory to access the Azure ATP console you configure a guest user and then add them to the proper Azure ATP AAD group.  

1. [] On ContosoDC, open a new tab in IE and browse to ```https://portal.azure.com```. You should be automatically logged in. If not, login with the following credentials.  
   
   ```@lab.CloudCredential(134).UserName``` 

   ```@lab.CloudCredential(134).Password``` 

2. []	Close any popup windows that might have opened.  
3. []	Click **Azure Active Directory**. 
4. []	Click **Users**. 
5. []	Click **New guest user**. 
6. []	Enter email address for guest user such as ```@lab.User.Email``` and click **Invite**. 
7. []	At the top of the window, click on the **Contoso** link or browse to ```https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade```.  
8. []	Click **Groups**. 
9. []	Click **Azure ATP {workspace name} Administrators group** (the first Azure ATP Group).  
10. []	Click **Members**. 
11. []	Click **Add members**. 
12. []	Select the **guest user added above** and click **Select**. 

	> [!NOTE]	After the user accepts the invitation the user will be able to access the Azure ATP console for this workspace using their email account. 

---

===
# Lab Environment Setup Complete
[:arrow_left: Home](#introduction)

The lab environment setup is now complete. The next section will cover Azure Information Protection (Roadmap discussion then Hands On Lab). If you decide to close out of the Lab during the roadmap discussion, please ensure that you **Save** the lab using the menu in the upper right corner of the browser.

!IMAGE[Save](\Media\save.png)

===
# Azure Information Protection Lab

## Introduction

Estimated time to complete this lab

60-120 minutes*

### Objectives

After completing this lab, you should be able to (depending on the exercises you choose):

- Discover sensitive data using the Azure Information Protection scanner  
- Configure Azure Information Protection labels
- Configure Azure Information Protection policies
- Classify and protect content with Azure Information Protection in Office applications
- Bulk Protect documents using the AIP client Windows integration
- Enable and Publish labels and policies in the Security and Compliance Center
- Classify and Protect sensitive data discovered by the AIP Scanner
- Monitor Usage, User Activity, and Data Risk using the Azure Log Analytics AIP dashboards
- Configure Exchange Online Mail Flow Rules to protect content and enhance AIP usability 

### Prerequisites

Before working on this lab, you must have:

- Familiarity using Windows 10
- Familiarity with PowerShell
- Familiarity with Office 365 applications

### Lab machine technology

This lab is designed to be completed on either a native Windows 10 machine or a VM with the following characteristics:

- Windows 10 Enterprise
- Office 365 ProPlus
- Azure Information Protection client (1.41.51.0)

Microsoft 365 E5 Tenant credentials will be provided during the event.  If you want to run through this lab after the event, you may use a tenant created through https://demos.microsoft.com or your own Microsoft 365 Tenant. This Lab Guide will be publicly available after the event at https://aka.ms/AIPHOL.

*:memo: Not all exercises can be completed within 60 minutes but time estimates are provided to aid in exercise selection.

---

===
# Log Analytics Configuration
[:arrow_left: Home](#azure-information-protection)

In order to collect log data from Azure Information Protection clients and services, you must first configure the log analytics workspace.

1. [] Switch to @lab.VirtualMachine(Client01).SelectLink and log in with the password +++@lab.VirtualMachine(Client01).Password+++.
1. [] In the Azure portal, type the word ```info``` into the **search bar** and press **Enter**, then click on **Azure Information Protection**. 

	!IMAGE[2598c48n.jpg](\Media\2598c48n.jpg)
	
	> [!HINT] If you do not see the search bar at the top of the portal, click on the **Magnifying Glass** icon to expand it.
	>
	> !IMAGE[ny3fd3da.jpg](\Media\ny3fd3da.jpg)

1. [] In the Azure Information Protection blade, under **Manage**, click **Configure analytics (preview)**.

1. [] Next, click on **+ Create new workspace**.

	!IMAGE[qu68gqfd.jpg](\Media\qu68gqfd.jpg)

	> [!ALERT] The reason we recommend creating a new workspace here is that, by default, only the creator and subscription administrators have access to an Azure Log Analytics workspace.  
	>
	> The data contained in this workspace will contain details about the **location** and **contents** of files containing **sensitive information**. 
	>
	> Restricting access to this workspace only to **trusted administrators** with a **need to know** is **highly recommended**.

1. [] Configure the Log analytics workspace using the values in the table below and click **OK**.

	|||
	|-----|-----|
	|OMS Workspace|**Type a unique Workspace Name**|
	|Resource Group|```AIP-RG```|
	|Location|**East US** (Or a location near the event)|

	^IMAGE[Open Screenshot](\Media\5butui15.jpg)
1. [] Next, back in the Configure analytics (preview) blade, **check the Boxes** next to the **workspace** and next to **Enable Content Matches**, and click **OK**.

	!IMAGE[gste52sy.jpg](\Media\gste52sy.jpg)
1. [] Click **Yes**, in the confirmation dialog.

	!IMAGE[zgvmm4el.jpg](\Media\zgvmm4el.jpg)

---

===
# Azure Information Protection
[:arrow_left: Home](#introduction)
## Overview

Azure Information Protection (AIP) is a cloud-based solution that can help organizations to protect sensitive information by classifying and (optionally) encrypting documents and emails on Windows, Mac, and Mobile devices. This is done using an organization defined classification taxonomy made up of labels and sub-labels. These labels may be applied manually by users, or automatically by administrators via defined rules and conditions.

The phases of AIP are shown in the graphic below.  

!IMAGE[Phases.png](\Media\Phases.png)

In this lab, we will give you options for addressing each of these phases using various features of AIP.  

The [AIP Scanner Discovery](#aip-scanner-discovery) exercise, will guide you through performing **Discovery using the AIP scanner**. We recommend that everyone complete this exercise first as this step is important to help show the current state of sensitive data in on-premises repositories. This enables you to make data based risk decisions that can help drive appropriate levels of urgency around the rest of your AIP deployment. :clock10: 10-15 min

The [Base Configuration](#base-configuration) exercise, contains information on configuring and testing **Global and Scoped Policy and Labels**. This will also include demonstrating **Recommended and Automatic labeling via the AIP client in Office 365 on Windows 10**. This is the longest exercise in the lab as it requires configuration of policy and the use of multiple clients.  We recommend this exercise if you have minimal experience with AIP. :clock10: 30-45 min

The [Bulk Classification](#bulk-classification) exercise, shows how to manually classify, label, and protect content using the Windows integration features of the AIP client. :clock10: 5 min

The [AIP Scanner Classification, Labeling, and Protection](#aip-scanner-classification-labeling-and-protection) exercise, will show how to use the **AIP scanner in Enforce mode** to take advantage of features like Automatic Conditions to help you **Classify, Label, and Protect** the discovered information easily. This exercise has a dependancy on completion of the AIP Scanner Dicovery exercise. :clock10: 5-10 min

The [Security and Compliance Center](#security-and-compliance-center) exercise, will help you understand how to **Enable and Publish labels in the Security and Compliance Center** so they can be used with Mac, Mobile, ISVs (like Adobe PDF), and other unified clients.  We will demonstrate this functionality using the Adobe PDF reader. :clock10: 5-10 min

In the [AIP Analytics Dashboards](#aip-analytics-dashboards) exercise, we will show how to **Monitor AIP Usage, User Activity, and Data Risk** using the new Azure Log Analytics dashboards built into the AIP Azure Portal. :clock10: 5 min

In the [Exchange IRM](#exchange-online-irm-capabilities) exercise, we will use Exchange PowerShell to create a Mail Flow Rule to prevent sensitive information from leaving your network in the clear.  We will also create a mail flow rule that prevents internal protected messages from accidentally being sent to external recipients who will be unable to open the content. :clock10: 10-15 min

Click on one of the options below to begin. At the end of each section, there will be a summary and links to the other sections so you may continue from that point.

- [AIP Scanner Discovery](#aip-scanner-discovery)
- [Base Configuration](#base-configuration)
- [Bulk Classification](#bulk-classification)
- [AIP Scanner CLP](#aip-scanner-classification-labeling-and-protection)
- [Security and Compliance Center](#security-and-compliance-center)
- [AIP Analytics Dashboards](#aip-analytics-dashboards)
- [Exchange IRM](#exchange-online-irm-capabilities)

---

===
# AIP Scanner Discovery
[:arrow_left: Home](#azure-information-protection)

Even before configuring an AIP classification taxonomy, customers can scan and identify files containing sensitive information based on the built-in sensitive information types included in the Microsoft Classification Engine.  

!IMAGE[ahwj80dw.jpg](\Media\ahwj80dw.jpg)

Often, this can help drive an appropriate level of urgency and attention to the risk customers face if they delay rolling out AIP classification and protection.  

In this exercise, we will install the AIP scanner and run it against repositories in discovery mode.  Later in this lab (after configuring labels and conditions), we will revisit the scanner to perform automated classification, labeling, and protection of sensitive documents. This Exercise will walk you through the items below.

- [Installing the AIP Scanner Service](#installing-the-aip-scanner-service)
- [Creating Azure AD Applications for the AIP Scanner](#creating-azure-ad-applications-for-the-aip-scanner)

---
## AIP Scanner Setup

In this task we will install the AIP scanner binaries and create the Azure AD Applications necessary for authentication.

### Installing the AIP Scanner Service

The first step in configuring the AIP Scanner is to install the service and connect the database.  This is done with the Install-AIPScanner cmdlet that is provided by the AIP Client software.  The AIPScanner service account has been pre-staged in Active Directory for convenience.

1. [] Switch to @lab.VirtualMachine(Scanner01).SelectLink and log in using the password +++@lab.VirtualMachine(Client01).Password+++.

1. [] Open an **Administrative PowerShell Window** and type ```C:\Users\LabUser\Desktop\InstallScanner.ps1``` and press **Enter**. 

1. [] In the popup box, click **OK** to accept the default of **Scanner01**.

	> [!NOTE] We have preconfigured SQL Server on Scanner01 with a **default instance**. If using a **named instance** or **SQL Server Express**, you would populate this with **ServerName\\InstanceName** or **ServerName\\SqlExpress** respectively.

3. [] When prompted, provide the credentials for the **local** AIP scanner service account.
	
	```Contoso\AIPScanner```

	```Somepass1```

	^IMAGE[Open Screenshot](\Media\pc9myg9x.jpg)

	> [!KNOWLEDGE] This script installs the AIP scanner Service account using the **local domain user** account provisioned for the AIP Scanner. This account will need to be provided **read** access to **all repositories** that need to be scanned during **discovery**.  
	>
	> When you begin **classifying and labeling** files with the AIP scanner, this account will also need **write** access to the repositories, so this is something you should consider during rights assignment. 
	>
	> This script will run the code below. This script is available online at https://aka.ms/labscripts
	>
	> Add-Type -AssemblyName Microsoft.VisualBasic
	> 
	> $SQL = [Microsoft.VisualBasic.Interaction]::InputBox('Enter the name of your SQL Server or Server\Instance', 'SQL Server', "Scanner01")
	>
	> Install-AIPScanner -SQLServerInstance $SQL
	
	^IMAGE[Open Screenshot](\Media\w7goqgop.jpg)

### Creating Azure AD Applications for the AIP Scanner

Now that you have installed the scanner service, you need to get an Azure AD token for the scanner service account to authenticate so that it can run unattended. This requires registering both a Web app and a Native app in Azure Active Directory.  The commands below will do this in an automated fashion rather than needing to go into the Azure portal directly.

1. [] Next, on the desktop, right-click on **GenerateAuthToken.ps1** and click **Run with PowerShell**.
1. [] When prompted, provide the username and password below. 
	
	```@lab.CloudCredential(134).Username```
	
	```@lab.CloudCredential(134).Password```

	> [!HINT] This will create a new **Web App Registration**, **Native App Registration**, and associated **Service Principals** in Azure AD. 
	>
	> Next, the script will output a new text file containing the **Set-AIPAuthentication** command and the **required values to generate the authentication token** for **any** AIP scanner server in an environment.
	
	> [!KNOWLEDGE] This script will run the code below. This script is available online at https://aka.ms/labscripts
	>
	> New-AzureADApplication -DisplayName AIPOnBehalfOf -ReplyUrls http://localhost
	> $WebApp = Get-AzureADApplication -Filter "DisplayName eq 'AIPOnBehalfOf'"
	> New-AzureADServicePrincipal -AppId $WebApp.AppId
	> $WebAppKey = New-Guid
	> $Date = Get-Date
	> New-AzureADApplicationPasswordCredential -ObjectId $WebApp.ObjectID -startDate $Date -endDate $Date.AddYears(1) -Value $WebAppKey.Guid -CustomKeyIdentifier "AIPClient"
	>
	> $AIPServicePrincipal = Get-AzureADServicePrincipal -All $true | ? {$_.DisplayName -eq 'AIPOnBehalfOf'}
	> $AIPPermissions = $AIPServicePrincipal | select -expand Oauth2Permissions
	> $Scope = New-Object -TypeName "Microsoft.Open.AzureAD.Model.ResourceAccess" -ArgumentList $AIPPermissions.Id,"Scope"
	> $Access = New-Object -TypeName "Microsoft.Open.AzureAD.Model.RequiredResourceAccess"
	> $Access.ResourceAppId = $WebApp.AppId
	> $Access.ResourceAccess = $Scope
	>
	> New-AzureADApplication -DisplayName AIPClient -ReplyURLs http://localhost -RequiredResourceAccess $Access -PublicClient $true
	> $NativeApp = Get-AzureADApplication -Filter "DisplayName eq 'AIPClient'"
	> New-AzureADServicePrincipal -AppId $NativeApp.AppId
	>
    > "Set-AIPAuthentication -WebAppID " + $WebApp.AppId + " -WebAppKey " + $WebAppKey.Guid + " -NativeAppID " + $NativeApp.AppId | Out-File ~\Desktop\Set-AIPAuthentication.txt
	> Start ~\Desktop\Set-AIPAuthentication.txt

1. [] Leave the notepad window open in the background.
1. [] **Click on the Start menu** and type ```PowerShell```, right-click on the PowerShell program, and click **Run as a different user**.

	!IMAGE[zgt5ikxl.jpg](\Media\zgt5ikxl.jpg)

1. [] When prompted, enter the username and password below and click **OK**.

	```Contoso\AIPScanner``` 

	```Somepass1```

1. [] Return to the **Notepad** window and copy the **full Set-AIPAuthentication** command into this window and run it.
1. [] When prompted, enter the username and password below:

	```AIPScanner@@lab.CloudCredential(134).TenantName```

	```Somepass1```

	^IMAGE[Open Screenshot](\Media\qfxn64vb.jpg)

1. [] In the Permissions requested window, click **Accept**.

   !IMAGE[nucv27wb.jpg](\Media\nucv27wb.jpg)
   
	>[!knowledge] You will a message like the one below in the PowerShell window once complete.
	>
	>!IMAGE[y2bgsabe.jpg](\Media\y2bgsabe.jpg)

1. [] **Close the PowerShell window**.
1. [] Next, in the **Admin PowerShell window**, run the command below.

	```Restart-Service AIPScanner```
   
---

## Configuring Repositories 

In this task, we will configure repositories to be scanned by the AIP scanner.  As previously mentioned, these can be any type of CIFS file shares including NAS devices sharing over the CIFS protocol.  Additionally, On premises SharePoint 2010, 2013, and 2016 document libraries and lists (attachements) can be scanned.  You can even scan entire SharePoint sites by providing the root URL of the site.  There are several optional 

> [!NOTE] SharePoint 2010 is only supported for customers who have extended support for that version of SharePoint.

The next task is to configure repositories to scan.  These can be on-premises SharePoint 2010, 2013, or 2016 document libraries and any accessible CIFS based share.

1. [] In the **Admin PowerShell window**, type ```C:\Users\LabUser\Desktop\ConfigureRepository.ps1``` and press **Enter**.

	> [!HINT] This command configures a **CIFS fileshare** repository and a **SharePoint document library** repository then **displays the configuration** to verify they were added.

	^IMAGE[Open Screenshot](\Media\scanner-repo.png)

    > [!KNOWLEDGE] The script runs the code below. This script is available online at https://aka.ms/labscripts
	>
	> Add-AIPScannerRepository -Path http://Scanner01/documents -SetDefaultLabel Off
	>
	> Add-AIPScannerRepository -Path \\Scanner01\documents -SetDefaultLabel Off
	>
	> Get-AIPScannerRepository
   
	>[!HINT] Notice that we added the **-SetDefaultLabel Off** switch to each of these repositories.  This is useful to prevent any Default labels from applying to files that do not match a condition when we do the enforced scan. This is optional and may be removed if desired.


---

## Running Sensitive Data Discovery 
[:arrow_up: Top](#configuring-aip-scanner-for-discovery)

1. [] In the **Admin PowerShell window**, type ```C:\Users\LabUser\Desktop\StartDiscovery.ps1``` and press **Enter**.

	> [!HINT] This command sets the global configuration of the AIP scanner to use **any custom conditions** that you have specified for labels in the Azure Information Protection policy, and the list of all **default sensitive information types** that are available to specify as conditions for labels.
	>
	> Although the scanner will discover documents to classify, it will not classify them because the configuration for the scanner is set to Discover only mode (Enforce Off).
	>
	> This command also starts the **initial discovery scan**.

	> [!KNOWLEDGE] The script runs the code below. This script is available online at https://aka.ms/labscripts
	>
	> Set-AIPScannerConfiguration -DiscoverInformationTypes All -Enforce Off
	>
	> Start-AIPScan
	
1. [] Right-click on the **Windows** button in the lower left-hand corner and click on **Event Viewer**.

	^IMAGE[Open Screenshot](\Media\cjvmhaf0.jpg)
1. [] Expand **Application and Services Logs** and click on **Azure Information Protection**.

	^IMAGE[Open Screenshot](\Media\dy6mnnpv.jpg)

	> [!ALERT] If you see a **.NET exception**, press **OK**. This is due to SharePoint startup in the VM environment. This event **must be acknowledged** to complete the discovery scan.

	> [!NOTE] You will see an event like the one below when the scanner completes the cycle. 
	>
	>!IMAGE[agnx2gws.jpg](\Media\agnx2gws.jpg)

1. [] Next, switch to @lab.VirtualMachine(Client01).SelectLink and log in using the password +++@lab.VirtualMachine(Client01).Password+++.
1. [] Open a **File Explorer** window, and browse to ```\\Scanner01.contoso.azure\c$\users\aipscanner\AppData\Local\Microsoft\MSIP\Scanner\Reports```. Use the credentials below to authenticate:

	```Contoso\LabUser```
	
	```Pa$$w0rd```

1. [] Review the summary txt and detailed csv files available there.  

	>[!Hint] Since there are no Automatic conditions configured yet, the scanner found no matches for the files scanned despite most of them containing sensitive data.
	>
	>!IMAGE[aukjn7zr.jpg](\Media\aukjn7zr.jpg)
	>
	>The details contained in the DetailedReport.csv can be used to identify the types of sensitive data you need to create AIP rules for in the Azure Portal.
	>
	>!IMAGE[9y52ab7u.jpg](\Media\9y52ab7u.jpg)

	>[!NOTE] We will revisit this information later in the lab to review discovered data and create Sensitive Data Type to Classification mappings.

	>[!ALERT] If you see any failures, it is likely due to SharePoint startup in the VM environment.  If you rerun Start-AIPScan on Scanner01 all files will successfully scan.  This should not happen in a production environment.

---

===
# AIP Scanner Discovery Exercise Complete

In this exercise, we installed the AIP scanner and performed a discovery scan against an on premises CIFS repository and SharePoint document library.  Although this was a very limited demonstration of the capabilities of the AIP scanner for discovery, it helps to show how quickly you can configure this tool and get actionable information which can be used to make data driven decisions about your security posture.  Choose one of the exercises below or click the Next button to continue sequentially.

- [Base Configuration](#configuring-azure-information-protection-policy)
- [Bulk Classification](#bulk-classification)
- [AIP Scanner CLP](#aip-scanner-classification-labeling-and-protection)
- [Security and Compliance Center](#security-and-compliance-center)
- [AIP Analytics Dashboards](#aip-analytics-dashboards)
- [Exchange IRM](#exchange-online-irm-capabilities)

---

===
# Base Configuration
[:arrow_left: Home](#azure-information-protection)

This exercise demonstrates using the Azure Information Protection blade in the Azure portal to configure policies and sub-labels.  We will create a new sub-label and configure protection and then modify an existing sub-label.  We will also create a label that will be scoped to a specific group.  

Next, we will configure AIP Global Policy to use the General sub-label as default, and finally, we will configure a scoped policy to use the new scoped label by default for Word, Excel, and PowerPoint while still using General as default for Outlook. This Exercise will walk you through the items below.


- [Creating, Configuring, and Modifying Sub-Labels](#creating-configuring-and-modifying-sub-labels)
- [Configuring Global Policy](#configuring-global-policy)
- [Creating a Scoped Label and Policy](#creating-a-scoped-label-and-policy)
- [Configuring Advanced Policy Settings](#configuring-advanced-policy-settings)
- [Defining Recommended and Automatic Conditions](#defining-recommended-and-automatic-conditions)

---
## Creating, Configuring, and Modifying Sub-Labels

In this task, we will configure a label protected for internal audiences that can be used to help secure sensitive data within your company.  By limiting the audience of a specific label to only internal employees, you can dramatically reduce the risk of unintentional disclosure of sensitive data and help reduce the risk of successful data exfiltration by bad actors.  

However, there are times when external collaboration is required, so we will configure a label to match the name and functionality of the Do Not Forward button in Outlook.  This will allow users to more securely share sensitive information outside the company to any recipient.  By using the name Do Not Forward, the functionality will also be familiar to what previous users of AD RMS or Azure RMS may have used in the past.

1. [] On @lab.VirtualMachine(Client01).SelectLink, log in with the password +++@lab.VirtualMachine(Client01).Password+++.
2. [] In the Azure Information Protection blade, under **Classifications** in the left pane, click on **Labels** to load the Azure Information Protection – Labels blade.

	^IMAGE[Open Screenshot](\Media\mhocvtih.jpg)

1. [] In the Azure Information Protection – Labels blade, right-click on **Confidential** and click **Add a sub-label**.

	^IMAGE[Open Screenshot](\Media\uktfuwuk.jpg)

1. [] In the Sub-label blade, type ```Contoso Internal``` for the **Label display name** and for **Description** enter text similar to ```Confidential data that requires protection, which allows Contoso Internal employees full permissions. Data owners can track and revoke content.```

	^IMAGE[Open Screenshot](\Media\4luorc0u.jpg)

1. [] Then, under **Set permissions for documents and emails containing this label**, click **Protect**, and under **Protection**, click on **Azure (cloud key)**.

	^IMAGE[Open Screenshot](\Media\tp97a19d.jpg)

1. [] In the Protection blade, click **+ Add Permissions**.

	^IMAGE[Open Screenshot](\Media\layb2pvo.jpg)

1. [] In the Add permissions blade, click on **+ Add contoso – All members** and click **OK**.

	^IMAGE[Open Screenshot](\Media\zc0iuoyz.jpg)

1. [] In the Protection blade, click **OK**.

	^IMAGE[Open Screenshot](\Media\u8jv46zo.jpg)

1. [] In the Sub-label blade, scroll down to the **Set visual marking (such as header or footer)** section and under **Documents with this label have a header**, click **On**.

	> Use the values in the table below to configure the Header.

	| Setting          | Value            |
	|:-----------------|:-----------------|
	| Header text      | ```Contoso Internal``` |
	| Header font size | ```24```               |
	| Header color     | Purple           |
	| Header alignment | Center           |

	> [!NOTE] These are sample values to demonstrate marking possibilities and **NOT** a best practice.

	^IMAGE[Open Screenshot](\Media\0vdoc6qb.jpg)

1. [] To complete creation of the new sub-label, click the **Save** button and then click **OK** in the Save settings dialog.

	^IMAGE[Open Screenshot](\Media\89nk9deu.jpg)

1. [] In the Azure Information Protection - Labels blade, expand **Confidential** (if necessary) and then click on **Recipients Only**.

	^IMAGE[Open Screenshot](\Media\eiiw5zbg.jpg)

1. [] In the Label: Recipients Only blade, change the **Label display name** from **Recipients Only** to ```Do Not Forward```.

	^IMAGE[Open Screenshot](\Media\v54vd4fq.jpg)

1. [] Next, in the **Set permissions for documents and emails containing this label** section, under **Protection**, click **Azure (cloud key): User defined**.

	^IMAGE[Open Screenshot](\Media\qwyranz0.jpg)

1. [] In the Protection blade, under **Set user-defined permissions (Preview)**, verify that only the Box next to **In Outlook apply Do Not Forward** is checked, then click **OK**.

	^IMAGE[Open Screenshot](\Media\16.png)

	> [!knowledge] Although there is no action added during this step, it is included to show that this label will only display in Outlook and not in Word, Excel, PowerPoint or File Explorer.

1. [] Click **Save** in the Label: Recipients Only blade and **OK** to the Save settings prompt. 

	^IMAGE[Open Screenshot](\Media\9spkl24i.jpg)

1. []  Click the **X** in the upper right corner of the blade to close.

	^IMAGE[Open Screenshot](\Media\98pvhwdv.jpg)

---

## Configuring Global Policy
[:arrow_up: Top](#base-configuration)

In this task, we will assign the new sub-label to the Global policy and configure several global policy settings that will increase Azure Information Protection adoption among your users and reduce ambiguity in the user interface.

1. [] In the Azure Information Protection blade, under **Classifications** on the left, click **Policies**. 
2. [] Click the **Global** policy.

	^IMAGE[Open Screenshot](\Media\24qjajs5.jpg)

1. [] In the Policy: Global blade, **wait for the labels to load**.

	>[!knowledge] The policies should look like the image below.  If they show as loading, refresh the full browser on this page and go back into the **Global** policy and they should load.
	>
	>!IMAGE[labels.png](\Media\labels.png)

2. [] Below the labels, click **Add or remove labels**.

3. [] In the Policy: Add or remove labels blade, ensure that the **Boxes** next to **all labels including the new Contoso Internal label** are **checked** and click **OK**.

4. [] In the Policy: Global blade, under the **Configure settings to display and apply on Information Protection end users** section, configure the policy to match the settings shown in the table and image below.

	| Setting | Value |
	|:--------|:------|
	| Select the default label | General |
	|All documents and emails must have a label…|On|
	|Users must provide justification to set a lower…|On|
	|For email messages with attachments, apply a label…|Automatic|
	|Display the Information Protection Bar in Office apps|On|
	|Add the Do Not Forward button to the Outlook ribbon|Off|

	!IMAGE[Open Screenshot](\Media\mtqhe3sj.jpg)

1. [] Click **Save**, then **OK** to complete configuration of the Global policy.

	^IMAGE[Open Screenshot](\Media\1p1q4pxe.jpg)

1. [] Click the **X** in the upper right corner to close the Policy: Global blade.

	^IMAGE[Open Screenshot](\Media\m6e4r2u2.jpg)

---

## Creating a Scoped Label and Policy
[:arrow_up: Top](#base-configuration)

Now that you have learned how to work with global labels and policies, we will create a new scoped label and policy for the Legal team at Contoso.  

1. [] Under **Classifications** on the left, click **Labels**.

	^IMAGE[Open Screenshot](\Media\50joijwb.jpg)

1. [] In the Azure Information Protection – Labels blade, right-click on **Highly-Confidential** and click **Add a sub-label**.

	^IMAGE[Open Screenshot](\Media\tasz9t0i.jpg)

1. [] In the Sub-label blade, enter ```Legal Only``` for the **Label display name** and for **Description** enter ```Data is classified and protected. Legal department staff can edit, forward and unprotect.```.

	^IMAGE[Open Screenshot](\Media\lpvruk49.jpg)

1. [] Then, under **Set permissions for documents and emails containing this label**, click **Protect** and under **Protection**, click **Azure (cloud key)**.

	^IMAGE[Open Screenshot](\Media\6ood4jqu.jpg)

1. [] In the Protection blade, under **Protection settings**, click the **+ Add permissions** link.

	!IMAGE[ozzumi7l.jpg](\Media\ozzumi7l.jpg)

1. [] In the Add permissions blade, click **+ Browse directory**.

	^IMAGE[Open Screenshot](\Media\2lvwim24.jpg)

1. [] In the AAD Users and Groups blade, **wait for the names to load**, then check the Boxes next to **Adam Smith** and **Alice Anderson**, and click the **Select** button.

	^IMAGE[Open Screenshot](\Media\uishk9yh.jpg)

	> [!Note] In a production environment, you will typically use a synced or Azure AD Group rather than choosing individuals.

1. [] In the Add permissions blade, click **OK**.

	^IMAGE[Open Screenshot](\Media\stvnaf4f.jpg)

1. [] In the Protection blade, under **Allow offline access**, reduce the **Number of days the content is available without an Internet connection** value to ```3``` and press **OK** .

	> [!Knowledge] This value determines how many days a user will have offline access from the time a document is opened, and an initial Use License is acquired.  While this provides convenience for users, it is recommended that this value be set appropriately based on the sensitivity of the content.

	^IMAGE[Open Screenshot](\Media\j8masv1q.jpg)

1. [] Click **Save** in the Sub-label blade and **OK** to the Save settings prompt to complete the creation of the Legal Only sub-label.

	^IMAGE[Open Screenshot](\Media\dfhoii1x.jpg)

1. [] In the Azure Information Protection blade, under **Classifications** on the left, click **Policies** then click the **+Add a new policy** link.

	^IMAGE[Open Screenshot](\Media\ospsddz6.jpg)

1. [] In the Policy blade, for Policy name, type ```No Default Label Scoped Policy``` and click on **Select which users or groups get this policy. Groups must be email-enabled.**

	!IMAGE[1sjw3mc7.jpg](\Media\1sjw3mc7.jpg)

1. [] In the AAD Users and Groups blade, click on **Users/Groups**.  
1. [] Then in the second AAD Users and Groups blade, **wait for the names to load** and check the Boxes next to **AIPScanner**, **Adam Smith**, and **Alice Anderson**.

	>[!NOTE] The **AIPScanner** account is added here to prevent all scanned documents from being labeled with a default label.
1. [] Click the **Select** button.
1. [] Finally, click **OK**.

	^IMAGE[Open Screenshot](\Media\onne7won.jpg)

1. [] In the Policy blade, under the labels, click on **Add or remove labels** to add the scoped label.

	!IMAGE[b6e9nbui.jpg](\Media\b6e9nbui.jpg)

1. [] In the Policy: Add or remove labels blade, check the Box next to **Legal Only** and click **OK**.

	^IMAGE[Open Screenshot](\Media\c2429kv9.jpg)

1. [] In the Policy blade, under **Configure settings to display and apply on Information Protection end users** section, under **Select the default label**, select **None** as the default label for this scoped policy.

	!IMAGE[4mxceage.jpg](\Media\4mxceage.jpg)

1. [] Click **Save**, then **OK** to complete creation of the No Default Label Scoped Policy.

	^IMAGE[Open Screenshot](\Media\41jembjf.jpg)

1. [] Click on the **X** in the upper right-hand corner to close the policy.

---

## Configuring Advanced Policy Settings
[:arrow_up: Top](#base-configuration)

There are many advanced policy settings that are useful to tailor your Azure Information Protection deployment to the needs of your environment.  In this task, we will cover one of the settings that is very complimentary when using scoped policies that have no default label or a protected default label.  Because the No Default Label Scoped Policy we created in the previous task uses a protected default label, we will be adding an alternate default label for Outlook to provide a more palatable user experience for those users.

1. [] In the Azure Information Protection blade, under **Classifications** on the left, click on **Labels** and then click on the **General** label.

    ^IMAGE[Open Screenshot](\Media\rvn4xorx.jpg)

1. [] In the Label: General blade, scroll to the bottom and copy the **Label ID** and close the blade using the **X** in the upper right-hand corner.

    !IMAGE[8fi1wr4d.jpg](\Media\8fi1wr4d.jpg)

1. [] In the AIP Portal, under **Classifications** on the left, click on **Policies**. 
1. [] **Right-click** on the **No Default Label Scoped Policy** and click on **Advanced settings**.

    ^IMAGE[Open Screenshot](\Media\2jo71ugb.jpg)

1. [] In the Advanced settings blade, in the textBox under **VALUE**, paste the **Label ID** for the **General** label you copied previously. In the textBox under **NAME**, type ```OutlookDefaultLabel```, then click **Save and close**.

    > [!ALERT] CAUTION: Please check to ensure that there are **no spaces** before or after the **Label ID** when pasting as this will cause the setting to not apply.

    !IMAGE[ezt8sfs3.jpg](\Media\ezt8sfs3.jpg)

	> [!HINT] This and additional Advanced Policy Settings can be found at [https://docs.microsoft.com/en-us/azure/information-protection/rms-client/client-admin-guide-customizations ](https://docs.microsoft.com/en-us/azure/information-protection/rms-client/client-admin-guide-customizations)

---

## Defining Recommended and Automatic Conditions
[:arrow_up: Top](#base-configuration)

One of the most powerful features of Azure Information Protection is the ability to guide your users in making sound decisions around safeguarding sensitive data.  This can be achieved in many ways through user education or reactive events such as blocking emails containing sensitive data. 

However, helping your users to properly classify and protect sensitive data at the time of creation is a more organic user experience that will achieve better results long term.  In this task, we will define some basic recommended and automatic conditions that will trigger based on certain types of sensitive data.

1. [] Under **Dashboards** on the left, click on **Data discovery (Preview)** to view the results of the discovery scan we performed previously.

	!IMAGE[Dashboard.png](\Media\Dashboard.png)

	> [!KNOWLEDGE] Notice that there are no labeled or protected files shown at this time.  This uses the AIP P1 discovery functionality available with the AIP Scanner. Only the predefined Office 365 Sensitive Information Types are available with AIP P1 as Custom Sensitive Information Types require automatic conditions to be defined, which is an AIP P2 feature.

	> [!NOTE] Now that we know the sensitive information types that are most common in this environment, we can use that information to create **Recommended** conditions that will help guide user behavior when they encounter this type of data.

	> [!ALERT] If no data is shown, it may still be processing. Continue with the lab and come back to see the results later.

1. [] Under **Classifications** on the left, click **Labels** then expand **Confidential**, and click on **Contoso Internal**.

	^IMAGE[Open Screenshot](\Media\jyw5vrit.jpg)
1. [] In the Label: Contoso Internal blade, scroll down to the **Configure conditions for automatically applying this label** section, and click on **+ Add a new condition**.

	!IMAGE[cws1ptfd.jpg](\Media\cws1ptfd.jpg)
1. [] In the Condition blade, in the **Select information types** search Box, type ```EU``` and check the Boxes next to the **items shown below**.

	!IMAGE[xaj5hupc.jpg](\Media\xaj5hupc.jpg)
1. [] Next, before saving, replace EU in the search bar with ```credit``` and check the Box next to **Credit Card Number**.

	^IMAGE[Open Screenshot](\Media\9rozp61b.jpg)
1. [] Click **Save** in the Condition blade and **OK** to the Save settings prompt.

	^IMAGE[Open Screenshot](\Media\41o5ql2y.jpg)

	> [!Knowledge] By default the condition is set to Recommended and a policy tip is created with standardized text.
	>
	>  !IMAGE[qdqjnhki.jpg](\Media\qdqjnhki.jpg)

1. [] Click **Save** in the Label: Contoso Internal blade and **OK** to the Save settings prompt.

	^IMAGE[Open Screenshot](\Media\rimezmh1.jpg)
1. [] Press the **X** in the upper right-hand corner to close the Label: Contoso Internal blade.

	^IMAGE[Open Screenshot](\Media\em124f66.jpg)
1. [] Next, expand **Highly Confidential** and click on the **All Employees** sub-label.

	^IMAGE[Open Screenshot](\Media\2eh6ifj5.jpg)
1. [] In the Label: All Employees blade, scroll down to the **Configure conditions for automatically applying this label** section, and click on **+ Add a new condition**.

	^IMAGE[Open Screenshot](\Media\8cdmltcj.jpg)
1. [] In the Condition blade, select the **Custom** tab and enter ```Password``` for the **Name** and in the textBox below **Match exact phrase or pattern**, type ```pass@word1```.

	!IMAGE[ra7dnyg6.jpg](\Media\ra7dnyg6.jpg)
1. [] Click **Save** in the Condition blade and **OK** to the Save settings prompt.

	^IMAGE[Open Screenshot](\Media\ie6g5kta.jpg)
1. [] In the Labels: All Employees blade, in the **Configure conditions for automatically applying this label** section, click **Automatic**.

   !IMAGE[245lpjvk.jpg](\Media\245lpjvk.jpg)

	> [!HINT] The policy tip is automatically updated when you switch the condition to Automatic.

1. [] Click **Save** in the Label: All Employees blade and **OK** to the Save settings prompt.

	^IMAGE[Open Screenshot](\Media\gek63ks8.jpg)

1. [] Press the **X** in the upper right-hand corner to close the Label: All Employees blade.

	^IMAGE[Open Screenshot](\Media\wzwfc1l4.jpg)

---

===
# Testing AIP Policies
[:arrow_left: Home](#azure-information-protection)

Now that you have 3 test systems with users being affected by different policies configured, we can start testing these policies.  This exercise will run through various scenarios to demonstrate the use of AIP global and scoped policies and show the functionality of recommended and automatic labeling. This Exercise will walk you through the items below.

- [Testing User Defined Permissions](#testing-user-defined-permissions)
- [Testing Global Policy](#testing-global-policy)
- [Testing Scoped Policy](#testing-scoped-policy)
- [Testing Recommended and Automatic Classification](#testing-recommended-and-automatic-classification)

> [!ALERT] If you see a warning about a metered connection in Office, click **Connect anyway** to allow Office to connect.  If you do not do this you will get errors when connecting to the AIP service. The VMs are set to metered to increase network speed.

---

## Testing User Defined Permissions
[:arrow_up: Top](#testing-aip-policies)

One of the most common use cases for AIP is the ability to send emails using User Defined Permissions (Do Not Forward). In this task, we will send an email using the Do Not Forward label to test that functionality.


1. [] On @lab.VirtualMachine(Client03).SelectLink, log in using the password +++@lab.VirtualMachine(Client01).Password+++.
2. [] Launch Microsoft Outlook, and click **Accept and start Outlook**.
3. [] In the username Box, type ```EvanG@@lab.cloudcredential(134).TenantName``` and click **Connect**.
4. [] When prompted, type ```pass@word1``` and Sign in.
5. [] On the Use this account everywhere page, click **Yes** then click **Done**.
6. [] Once configuration completes, **uncheck the Box** to **Set up Outlook Mobile** and click **OK**.
7. [] **Close Outlook** and **reopen** to complete activation.
8. [] Once Outlook opens, if you receive a metered connection warning, click **Connect anyway**.
9. [] Click on the **New email** button.

  !IMAGE[6wan9me1.jpg](\Media\6wan9me1.jpg)

	> [!KNOWLEDGE] Note that the **Sensitivity** is set to **General** by default.
	>
	> !IMAGE[5esnhwkw.jpg](\Media\5esnhwkw.jpg)

8. [] Send an email to **Adam Smith** and **Alice Anderson** (```Adam Smith;Alice Anderson```). You may **optionally add an external email address** (preferably from a major social provider like gmail, yahoo, or outlook.com) to test the external recipient experience. For the **Subject** and **Body** type ```Test Do Not Forward Email```.

  ^IMAGE[Open Screenshot](\Media\h0eh40nk.jpg)

9. [] In the Sensitivity Toolbar, click on **Confidential** and then the **Do Not Forward** sub-label and click **Send**.

  !IMAGE[w8j1w1lm.jpg](\Media\w8j1w1lm.jpg)

	> [!Knowledge] If you receive the error message below, click on the Confidential \ Contoso Internal sub-label to force the download of your AIP identity certificates, then follow the steps above to change the label to Confidential \ Do Not Forward.
	>
	> !IMAGE[6v6duzbd.jpg](\Media\6v6duzbd.jpg)

10. [] Switch over to @lab.VirtualMachine(Client01).SelectLink, log in using the password +++@lab.VirtualMachine(Client01).Password+++ and open Outlook. 
11. [] Run through setup, this time using the credentials ```AdamS@@lab.CloudCredential(134).TenantName``` and ```pass@word1```. 
12. [] Review the email in Adam Smith’s Outlook.  You will notice that the email is automatically shown in Outlook natively.

   !IMAGE[0xby56qt.jpg](\Media\0xby56qt.jpg)

	> [!Hint] The **Do Not Forward** protection template will normally prevent the sharing of the screen and taking screenshots when protected documents or emails are loaded.  However, since this screenshot was taken within a VM, the operating system was unaware of the protected content and could not prevent the capture.  
	>
	>It is important to understand that although we put controls in place to reduce risk, if a user has view access to a document or email they can take a picture with their smartphone or even retype the message. That said, if the user is not authorized to read the message then it will not even render and we will demonstrate that next.

	> [!KNOWLEDGE] If you elected to send a Do Not Forward message to an external email, you will have an experience similar to the images below.  These captures are included to demonstrate the functionality for those that chose not to send an external message.
	>
	> !IMAGE[tzj04wi9.jpg](\Media\tzj04wi9.jpg)
	> 
	> Here the user has received an email from Evan Green and they can click on the **Read the message** button.
	>
	>!IMAGE[wiefwcho.jpg](\Media\wiefwcho.jpg)
	>
	>Next, the user is given the option to either log in using the social identity provider (**Sign in with Google**, Yahoo, Microsoft Account), or to **sign in with a one-time passcode**.
	>
	>If they choose the social identity provider login, it should use the token previously cached by their browser and display the message directly.
	>
	>If they choose one-time passcode, they will receive an email like the one below with the one-time passcode.
	>
	>!IMAGE[m6voa9xi.jpg](\Media\m6voa9xi.jpg)
	>
	>They can then use this code to authenticate to the Office 365 Message Encryption portal.
	>
	>!IMAGE[8pllxint.jpg](\Media\8pllxint.jpg)
	>
	>After using either of these authentication methods, the user will see a portal experience like the one shown below.
	>
	>!IMAGE[3zi4dlk9.jpg](\Media\3zi4dlk9.jpg)

---

## Testing Global Policy
[:arrow_up: Top](#testing-aip-policies)

In this task, we will create a document and send an email to demonstrate the functionality defined in the Global Policy.

1. [] Switch to @lab.VirtualMachine(Client03).SelectLink and log in with the password +++@lab.VirtualMachine(Client01).Password+++.
1. [] In Microsoft Outlook, click on the **New email** button.

	^IMAGE[Open Screenshot](\Media\6wan9me1.jpg)

1. [] Send an email to Adam Smith, Alice Anderson, and yourself (```Adam Smith;Alice Anderson;@lab.User.Email```).  For the **Subject** and **Body** type ```Test Contoso Internal Email```.

	^IMAGE[Open Screenshot](\Media\9gkqc9uy.jpg)

1. [] In the Sensitivity Toolbar, click on **Confidential** and then **Contoso Internal** and click **Send**.

  ^IMAGE[Open Screenshot](\Media\yhokhtkv.jpg)

1. [] On @lab.VirtualMachine(Client01).SelectLink, log in using the password +++@lab.VirtualMachine(Client01).Password+++ and observe that you are able to open the email natively in the Outlook client. Also observe the **header text** that was defined in the label settings.

	!IMAGE[bxz190x2.jpg](\Media\bxz190x2.jpg)
	
1. [] In your email, note that you will be unable to open this message.  This experience will vary depending on the client you use (the image below is from Outlook 2016 for Mac) but they should have similar messages after presenting credentials. Since this is not the best experience for the recipient, later in the lab we will configure Exchange Online Mail Flow Rules to prevent content classified with internal only labels from being sent to external users.
	
	!IMAGE[52hpmj51.jpg](\Media\52hpmj51.jpg)

---

## Testing Scoped Policy
[:arrow_up: Top](#testing-aip-policies)

In this task, we will create a document and send an email from one of the users in the Legal group to demonstrate the functionality defined in the first exercise. We will also show the behavior of the No Default Label policy on documents.

1. [] Switch to @lab.VirtualMachine(Client01).SelectLink and log in with the password +++@lab.VirtualMachine(Client01).Password+++.
1. [] In Microsoft Outlook, click on the **New email** button.
	
	^IMAGE[Open Screenshot](\Media\ldjugk24.jpg)
	
1. [] Send an email to Alice Anderson and Evan Green (```Alice Anderson;Evan Green```).  For the **Subject** and **Body** type ```Test Highly Confidential Legal Email```.
1. [] In the Sensitivity Toolbar, click on **Highly Confidential** and the **Legal Only** sub-label, then click **Send**.

	^IMAGE[Open Screenshot](\Media\ny1lwv0h.jpg)
1. [] Switch to @lab.VirtualMachine(Client02).SelectLink and log in with the password +++@lab.VirtualMachine(Client01).Password+++.
11. [] Run through setup, this time using the credentials ```AliceA@@lab.CloudCredential(134).TenantName``` and ```pass@word1```. 
12. [] Review the email in Alice Anderson’s Outlook. You should be able to open the message natively in the client as Alice.

	!IMAGE[qeqtd2yr.jpg](\Media\qeqtd2yr.jpg)
1. [] Switch to @lab.VirtualMachine(Client03).SelectLink and log in with the password +++@lab.VirtualMachine(Client01).Password+++.
2. [] Click on the email. You should be unable to open the message as Evan.

	!IMAGE[6y99u8cl.jpg](\Media\6y99u8cl.jpg)

	> [!Knowledge] You may notice that the Office 365 Message Encryption wrapper message is displayed in the preview pane.  It is important to note that the content of the email is not displayed here.  The content of the message is contained within the encrypted message.rpmsg attachment and only authorized users will be able to decrypt this attachment.
	>
	>!IMAGE[w4npbt49.jpg](\Media\w4npbt49.jpg)
	>
	>If an unauthorized recipient clicks on **Read the message** to go to the OME portal, they will be presented with the same wrapper message.  Like the external recipient from the previous task, this is not an ideal experience. So, you may want to use a mail flow rule to manage scoped labels as well.
	>
	>!IMAGE[htjesqwe.jpg](\Media\htjesqwe.jpg)

1. [] Switch to @lab.VirtualMachine(Client01).SelectLink and log in with the password +++@lab.VirtualMachine(Client01).Password+++.
1. [] Open **Microsoft Word**.
1. [] Create a new **Blank document** and type ```This is a test document``` and **save the document**.

	> [!ALERT] When you click **Save**, you will be prompted to choose a classification.  This is a result of having **None** set as the default label in the scoped policy while requiring all documents to be labeled.  This is a useful for driving **active classification decisions** by specific groups within your organization.  Notice that Outlook still has a default of **General** because of the Advanced setting we added to the scoped policy.  **This is recommended** because user send many more emails each day than they create documents. Actively forcing users to classify each email would be an unpleasant user experience whereas they are typically more understanding of having to classify each document if they are in a sensitive department or role.

1. [] Choose a classification to save the document.

---

## Testing Recommended and Automatic Classification
[:arrow_up: Top](#testing-aip-policies)

In this task, we will test the configured recommended and automatic conditions we defined in Exercise 1.  Recommended conditions can be used to help organically train your users to classify sensitive data appropriately and provides a method for testing the accuracy of your dectections prior to switching to automatic classification.  Automatic conditions should be used after thorough testing or with items you are certain need to be protected. Although the examples used here are fairly simple, in production these could be based on complex regex statements or only trigger when a specific quantity of sensitive data is present.

1. [] Switch to @lab.VirtualMachine(Client03).SelectLink and log in with the password +++@lab.VirtualMachine(Client01).Password+++.
1. [] Launch **Microsoft Word**.
1. [] In Microsoft Word, create a new **Blank document** and type ```My AMEX card number is 344047014854133. The expiration date is 09/28, and the CVV is 4368``` and **save** the document.

	> [!NOTE] This card number is a fake number that was generated using the Credit Card Generator for Testing at [https://developer.paypal.com/developer/creditCardGenerator/](https://developer.paypal.com/developer/creditCardGenerator/).  The Microsoft Classification Engine uses the Luhn Algorithm to prevent false positives so when testing, please make sure to use valid numbers.

1. [] Notice that you are prompted with a recommendation to change the classification to Confidential \ Contoso Internal. Click on **Change now** to set the classification and protect the document.

	!IMAGE[url9875r.jpg](\Media\url9875r.jpg)
	> [!Knowledge] Notice that, like the email in Task 2 of this exercise, the header value configured in the label is added to the document.
	>
	>!IMAGE[dcq31lz1.jpg](\Media\dcq31lz1.jpg)
1. [] In Microsoft Word, create a new **Blank document** and type ```my password is pass@word1``` and **save** the document.

	>[!HINT] Notice that the document is automatically classified and protected wioth the Highly Confidential \ All Employees label.
	>
	>!IMAGE[6vezzlnj.jpg](\Media\6vezzlnj.jpg)
1. [] Next, in Microsoft Outlook, click on the **New email** button.
	
	^IMAGE[Open Screenshot](\Media\ldjugk24.jpg)
	
1. [] Draft an email to Alice Anderson and Adam Smith (```Alice Anderson;Adam Smith```).  For the **Subject** and **Body** type ```Test Highly Confidential All Employees Automation```.

	^IMAGE[Open Screenshot](\Media\4v3wrrop.jpg)
1. [] Attach the **second document you created** to the email.

	!IMAGE[823tzyfd.jpg](\Media\823tzyfd.jpg)

	> [!HINT] Notice that the email was automatically classified as Highly Confidential \ All Employees.  This functionality is highly recommended because matching the email classification to attachments provides a much more cohesive user experience and helps to prevent inadvertent information disclosure in the body of sensitive emails.
	>
	>!IMAGE[yv0afeow.jpg](\Media\yv0afeow.jpg)

1. [] In the email, click **Send**.
   
---

===
# Base Configuration Exercise Complete

In this exercise, we walked through the configuration of Global and Scoped policies and labels.  We demonstrated the use of these and showed how recommended and automatic conditions function within the Office applications.  Choose one of the exercises below or click the Next button to continue sequentially.

- [AIP Scanner Discovery](#aip-scanner-discovery)
- [Bulk Classification](#bulk-classification)
- [AIP Scanner CLP](#aip-scanner-classification-labeling-and-protection)
- [Security and Compliance Center](#security-and-compliance-center)
- [AIP Analytics Dashboards](#aip-analytics-dashboards)
- [Exchange IRM](#exchange-online-irm-capabilities)

---

===
# Bulk Classification
[:arrow_left: Home](#azure-information-protection)

In this task, we will perform bulk classification using the built-in functionality of the AIP client.  This can be useful for users that want to classify/protect many documents that exist in a central location or locations identified by scanner discovery.  Because this is done manually, it is an AIP P1 feature.

1. [] On @lab.VirtualMachine(Scanner01).SelectLink, log in with the password +++@lab.VirtualMachine(Scanner01).Password+++.
2. [] Browse to the **C:\\**.
3. [] Right-click on the PII folder and select **Classify and Protect**.
   
   !IMAGE[CandP.png](\Media\CandP.png)
4. [] When prompted, click use another account and use the credentials below to authenticate:

	```AIPScanner@@lab.CloudCredential(134).TenantName```

	```Somepass1```

1. [] In the AIP client Classify and protect interface, select **Highly Confidential\\All Employees** and press **Apply**. 

	!IMAGE[CandP2.png](\Media\CandP2.png)

> [!Alert] If you are unable to see the **Apply** button due to screen resolution, click **Alt+A** and **Enter** to apply the label to the content.

> [!NOTE] You may review the results in a text file by clicking show results, or simply close the window.

---

===
# Bulk Classifiation Exercise Complete

In this exercise, we performed bulk classification using the built-in functionality of the AIP client.  This can be useful for users that want to classify/protect many documents that exist in a central location or locations identified by scanner discovery.  Choose one of the exercises below or click the Next button to continue sequentially.

- [AIP Scanner Discovery](#aip-scanner-discovery)
- [Base Configuration](#configuring-azure-information-protection-policy)
- [AIP Scanner CLP](#aip-scanner-classification-labeling-and-protection)
- [Security and Compliance Center](#security-and-compliance-center)
- [AIP Analytics Dashboards](#aip-analytics-dashboards)
- [Exchange IRM](#exchange-online-irm-capabilities)

---

===
# AIP Scanner Classification, Labeling, and Protection
[:arrow_left: Home](#azure-information-protection)

The Azure Information Protection scanner allows you to  classify and protect sensitive information stored in on-premises CIFS file shares and SharePoint sites.  

In this exercise, you will configure conditions for automatic classification.  After that, we will run the AIP Scanner in enforce mode to classify and protect the identified sensitive data. This Exercise will walk you through the items below.

- [Defining Automatic Conditions](#defining-automatic-conditions)
- [Enforcing Configured Rules](#enforcing-configured-rules)
- [Reviewing Protected Documents](#reviewing-protected-documents)

> [!Alert] This exercise requires completion of the previous [AIP Scanner Discovery](#aip-scanner-discovery) exercise.  If you did not already complete that exercise, please do so prior to continuing.

---



## Enforcing Configured Rules
[:arrow_up: Top](#aip-scanner-classification-labeling-and-protection)

In this task, we will set the AIP scanner to enforce the conditions we set up in the previous task and have it rerun on all files using the Start-AIPScan command.

1. [] Switch to @lab.VirtualMachine(Scanner01).SelectLink and log in with the password +++@lab.VirtualMachine(Scanner01).Password+++.
1. [] In an **Administrative PowerShell** window, type ```C:\Users\LabUser\Desktop\StartEnforce.ps1``` and press **Enter**. 

    > [!KNOWLEDGE] The script runs the code below. This script is available online at https://aka.ms/labscripts
	>
	> Set-AIPScannerConfiguration -Enforce On -DiscoverInformationTypes PolicyOnly
	>
	>Start-AIPScan
    
	> [!HINT] Note that this time we used the DiscoverInformationTypes -PolicyOnly switch before starting the scan. This will have the scanner only evaluate the conditions we have explicitly defined in conditions.  This increases the effeciency of the scanner and thus is much faster.  After reviewing the event log we will see the result of the enforced scan.
	>
	>!IMAGE[k3rox8ew.jpg](\Media\k3rox8ew.jpg)
		

1. [] Switch back to @lab.VirtualMachine(Client01).SelectLink and log in using the password +++@lab.VirtualMachine(Client01).Password+++.
1. [] Look in the reports directory we opened previously at ```\\Scanner01.contoso.azure\c$\users\aipscanner\AppData\Local\Microsoft\MSIP\Scanner\Reports```, you will notice that the old scan reports are zipped in the directory and only the most recent results are showing.  
	
	> If needed, use the credentials below:
	>
	>```Contoso\LabUser```
	>
	>```Pa$$w0rd```

	>[!NOTE] The DetailedReport.csv now shows the files that were protected.
	>
	>
	>!IMAGE[6waou5x3.jpg](\Media\6waou5x3.jpg)


---

## Reviewing Protected Documents
[:arrow_up: Top](#aip-scanner-classification-labeling-and-protection)

Now that we have Classified and Protected documents using the scanner, we can review the documents to see their change in status.

1. [] Switch to @lab.VirtualMachine(Client01).SelectLink and log in with the password +++@lab.VirtualMachine(Client01).Password+++.

2. [] Navigate to ```\\Scanner01.contoso.azure\documents```. 

	> If needed, use the credentials below:
	>
	>```Contoso\LabUser```
	>
    >```Pa$$w0rd```

	^IMAGE[Open Screenshot](\Media\hipavcx6.jpg)
3. [] Open one of the Contoso Purchasing Permissions documents.

	> [!NOTE] If asked to log in, use the credentials below.
	>
	> ```AdamS@@lab.CloudCredential(134).TenantName```
	>
	> ```pass@word1```

    > [!NOTE] Observe that the document is classified as Highly Confidential \ All Employees. 
    >
    > !IMAGE[s1okfpwu.jpg](\Media\HCAE.jpg)

4. [] Next, in the same documents folder, open one of the pdf files.
5. [] When prompted by Adobe, enter ```AdamS@@lab.CloudCredential(17).TenantName``` and press OK.
6. [] Check the box to save credentials and press OK.

	> [!NOTE] The PDF will now open and display the sensitivity across the top of the document.

	> [!Knowledge] The latest version of Acrobat Reader DC and the MIP Plugin have been installed on this system prior to the lab. Additionally, the sensitivity does not display by default in Adobe Acrobat Reader DC.  You must make the modifications below to the registry to make this bar display.
	>
	> In **HKEY_CURRENT_USER\Software\Adobe\Acrobat Reader\DC\MicrosoftAIP**, create a new **DWORD** value of **bShowDMB** and set the **Value** to **1**.
	>
	> !IMAGE[1547416250228](\Media\1547416250228.png)
---

===
# AIP Scanner CLP Exercise Complete

In this exercise, we configured the AIP scanner to use automatic conditions to classify, label, and protect documents in our defined repositories.  Choose one of the exercises below or click the Next button to continue sequentially.

- [AIP Scanner Discovery](#aip-scanner-discovery)
- [Base Configuration](#configuring-azure-information-protection-policy)
- [Bulk Classification](#bulk-classification)
- [Security and Compliance Center](#security-and-compliance-center)
- [AIP Analytics Dashboards](#aip-analytics-dashboards)
- [Exchange IRM](#exchange-online-irm-capabilities)

---

===

# Security and Compliance Center
[:arrow_left: Home](#azure-information-protection)

In this exercise, we will migrate your AIP Labels and activate them in the Security and Compliance Center.  This will allow you to see the labels in Microsoft Information Protection based clients such as Office 365 for Mac and Mobile Devices.

We will demonstrate these capabilities using Adobe PDF integration with AIP.

---
## Activating Unified Labeling

In this task, we will activate the labels from the Azure Portal for use in the Security and Compliance Center.

1. [] On @lab.VirtualMachine(Client01).SelectLink, log in with the password +++@lab.VirtualMachine(Client01).Password+++.
1. [] Navigate to ```https://portal.azure.com/?ActivateMigration=true#blade/Microsoft_Azure_InformationProtection/DataClassGroupEditBlade/migrationActivationBlade```

1. [] Click **Activate** and **Yes**.

	!IMAGE[o0ahpimw.jpg](\Media\o0ahpimw.jpg)

	>[!NOTE] You should see a message similar to the one below.
	>
	> !IMAGE[SCCMigration.png](\Media\SCCMigration.png) 

1. [] In a new tab, browse to ```https://protection.office.com/``` and click on **Classifications** and **Labels** to review the migrated labels. 

	>[!NOTE] Keep in mind that now the SCC Sensitivity Labels have been activated, so any modifications, additions, or deletions will be syncronised to Azure Information Protection in the Azure Portal. There are some functional differences between the two sections (DLP in SCC, HYOK & Custom Permissions in AIP), so please be aware of this when modifying policies to ensure a consistent experience on clients. 

---
## Deploying Policy in SCC

The previous step enabled the AIP labels for use in the Security and Compliance Center.  However, this did not also recreate the policies from the AIP portal. In this step we will publish a Global policy like the one we used in the AIP portal for use with unified clients.

1. [] In the Security and Compliance Center, under **Classifications**, click on **Label policies**.

2. [] In the Label policies pane, click **Publish labels**.

   ^IMAGE[Open Screenshot](\Media\SCC01.png)

3. [] On the Choose labels to publish page, click the **Choose labels to publish** link.

   ^IMAGE[Open Screenshot](\Media\SCC02.png)

4. [] In the Choose labels pane, click the **+ Add** button.

   ^IMAGE[Open Screenshot](\Media\SCC03.png)

5. [] Click the Box next to **Display name** to select all labels, then click the **Add** button.

   ^IMAGE[Open Screenshot](\Media\SCC04.png)

6. [] Click the **Done** button.

   ^IMAGE[Open Screenshot](\Media\SCC05.png)

7. [] Back on the Choose labels to publish page, click the **Next** button.

   ^IMAGE[Open Screenshot](\Media\SCC06.png)

8. [] On the Publish to users and groups page, notice that **All users are included by default**. If you were creating a scoped policy, you would choose specific users or groups to publish to. Click **Next**.

   ^IMAGE[Open Screenshot](\Media\SCC07.png)

9. [] On the Policy settings page, select the **General** label from the drop-down next to **Apply this label by default to documents and email**.

10. [] Check the Box next to **Users must provide justification to remove a label or lower classification label** and click the **Next** button.

    !IMAGE[Open Screenshot](\Media\SCC08.png)

11. [] In the Name textBox, type ```Global Policy``` and for the Description type ```This is the default global policy for all users.``` and click the **Next** button.

    ^IMAGE[Open Screenshot](\Media\SCC09.png)

12. [] Finally, on the Review your settings page, click the **Publish** button.

    !IMAGE[Open Screenshot](\Media\SCC10.png)

---
## Reviewing Adobe PDF Integration

1. [] To review a protected PDF, navigate to ```\\Scanner01.contoso.azure\documents```. 

	> If needed, use the credentials below:
	>
	>```Contoso\LabUser```
	>
    >```Pa$$w0rd```

4. [] In the documents folder, open one of the pdf files.
5. [] When prompted by Adobe, enter ```AdamS@@lab.CloudCredential(134).TenantName``` and press **OK** or **Next**.

	> [!NOTE] If prompted, provide the password ```pass@word1```.

6. [] Click **Yes** to save credentials.
7. [] In the **Permissions requested** dialog, click **Accept**.

	> [!NOTE] The PDF will now open and display the sensitivity across the top of the document.
	>
	> !IMAGE[PDF](\Media\PDF.png)

	> [!Knowledge] The latest version of Acrobat Reader DC and the MIP Plugin have been installed on this system prior to the lab. Additionally, the sensitivity does not display by default in Adobe Acrobat Reader DC.  You must make the modifications below to the registry to make this bar display.
	>
	> In **HKEY_CURRENT_USER\Software\Adobe\Acrobat Reader\DC\MicrosoftAIP**, create a new **DWORD** value of **bShowDMB** and set the **Value** to **1**.
	>
	> !IMAGE[1547416250228](\Media\1547416250228.png)

---

===
# Security and Compliance Center Exercise Complete

In this exercise, we enabled and published labels and policies in the Security and Compliance Center for use with clients based on the MIP SDK.  We demonstrated this using Adobe PDF integration.  Choose one of the exercises below or click the Next button to continue sequentially.

- [AIP Scanner Discovery](#aip-scanner-discovery)
- [Base Configuration](#configuring-azure-information-protection-policy)
- [Bulk Classification](#bulk-classification)
- [AIP Scanner CLP](#aip-scanner-classification-labeling-and-protection)
- [AIP Analytics Dashboards](#aip-analytics-dashboards)
- [Exchange IRM](#exchange-online-irm-capabilities)

---

===
# AIP Analytics Dashboards
[:arrow_left: Home](#azure-information-protection)

In this exercise, we will go to the AIP Analytics dashboards and observe them after completing all of the steps in the various exercises.  These dashboards give actionable data to AIP admins and their management related to how users are classifying and protecting data and where sensitive data is located throughout the environment.  Depending on which exercises you completed these may look different from the screenshots.

1. [] On @lab.VirtualMachine(Client01).SelectLink, open the browser that is logged into the Azure Portal.

1. [] In the AIP Portal, under **Dashboards**, click on **Usage report (Preview)**.

	> [!NOTE] Observe that there are now entries from the AIP scanner, File Explorer, Microsoft Outlook, and Microsoft Word based on our activities in this lab. 
	>
	> !IMAGE[Usage.png](\Media\newusage.png)

2. [] Next, under dashboards, click on **Activity logs (preview)**.
   
    > [!NOTE] We can now see activity from various users and clients including the AIP Scanner and specific users. 
	>
	> !IMAGE[activity.png](\Media\activity.png)
	>
	> You can also very quickly filter to just the **Highly Confidential** documents and identify the repositories and devices that contain this sensitive information.
	>
	> !IMAGE[activity2.png](\Media\activity2.png)

3. [] Finally, click on **Data discovery (Preview)**.

	> [!NOTE] In the Data discovery dashboard, you can see a breakdown of how files are being protected and locations that have sensitive content.
	>
	> !IMAGE[Discovery.png](\Media\Discovery.png)
	> 
	> If you click on one of the locations, you can drill down and see the content that has been protected on that specific device or repository.
	>
	> !IMAGE[discovery2.png](\Media\discovery2.png)
	
---

===
# AIP Analytics Dashboard Exercise Complete

In this exercise, we reviewed the new AIP Azure Log Analytics dashboards.  Choose one of the exercises below or click the Next button to continue sequentially.

- [AIP Scanner Discovery](#aip-scanner-discovery)
- [Base Configuration](#configuring-azure-information-protection-policy)
- [Bulk Classification](#bulk-classification)
- [AIP Scanner CLP](#aip-scanner-classification-labeling-and-protection)
- [Security and Compliance Center](#security-and-compliance-center)
- [Exchange IRM](#exchange-online-irm-capabilities)

---

===
# Exchange Online IRM Capabilities
[:arrow_left: Home](#azure-information-protection)

Exchange Online can work in conjunction with Azure Information Protection to provide advanced capabilities for protecting sensitive data being sent over email.  You can also manage the flow of classified content to ensure that it is not sent to unintended recipients. This Exercise will walk you through the items below.

- [Configuring Exchange Online Mail Flow Rules](#configuring-exchange-online-mail-flow-rules)
- [Demonstrating Exchange Online Mail Flow Rules](#demonstrating-exchange-online-mail-flow-rules)

## Configuring Exchange Online Mail Flow Rules

In this task, we will configure a mail flow rule to detect sensitive information traversing the network in the clear and encrypt it using the Encrypt Only RMS Template.  We will also create a mail flow rule to prevent messages classified as Confidential \ All Employees from being sent to external recipients.

1. [] Switch to @lab.VirtualMachine(Client01).SelectLink and open an **Admin PowerShell Prompt**.

1. [] In an **Administrative PowerShell** window, type ```C:\Users\LabUser\Desktop\EncryptSensitiveMFR.ps1``` and press **Enter**. 
1. [] When prompted, provide the credentials below:

	```@lab.CloudCredential(134).Username```

	```@lab.CloudCredential(134).Password```

	> [!NOTE] If prompted to remove a transport rule, hit **Enter**.

	>[!KNOWLEDGE] This mail flow rule can be used to encrypt sensitive data leaving via email.  This can be customized to add additional sensitive data types. A breakdown of the command is listed below.
	>
	>New-TransportRule 
	>
	>-Name "Encrypt external mails with sensitive content" 
	>
	>-SentToScope NotInOrganization 
	>
	>-ApplyRightsProtectionTemplate "Encrypt" 
	>
	>-MessageContainsDataClassifications @(@{Name="ABA Routing Number"; minCount="1"},@{Name="Credit Card Number"; minCount="1"},@{Name="Drug Enforcement Agency (DEA) Number"; minCount="1"},@{Name="International Classification of Diseases (ICD-10-CM)"; minCount="1"},@{Name="International Classification of Diseases (ICD-9-CM)"; minCount="1"},@{Name="U.S. / U.K. Passport Number"; minCount="1"},@{Name="U.S. Bank Account Number"; minCount="1"},@{Name="U.S. Individual Taxpayer Identification Number (ITIN)"; minCount="1"},@{Name="U.S. Social Security Number (SSN)"; minCount="1"})

    > [!KNOWLEDGE] The script runs the code below. This script is available online at https://aka.ms/labscripts
	>
	> $UserCredential = Get-Credential
	>
	>$Session = New-PSSession -ConfigurationName Microsoft.Exchange -ConnectionUri https://outlook.office365.com/powershell-liveid/ -Credential $UserCredential -Authentication Basic -AllowRedirection
	Import-PSSession $Session
	>
	> If(Get-TransportRule *Delete*){Remove-TransportRule *Delete*}
	>
	> New-TransportRule -Name "Encrypt external mails with sensitive content" -SentToScope NotInOrganization -ApplyRightsProtectionTemplate "Encrypt" -MessageContainsDataClassifications @(@{Name="ABA Routing Number"; minCount="1"},@{Name="Credit Card Number"; minCount="1"},@{Name="Drug Enforcement Agency (DEA) Number"; minCount="1"},@{Name="International Classification of Diseases (ICD-10-CM)"; minCount="1"},@{Name="International Classification of Diseases (ICD-9-CM)"; minCount="1"},@{Name="U.S. / U.K. Passport Number"; minCount="1"},@{Name="U.S. Bank Account Number"; minCount="1"},@{Name="U.S. Individual Taxpayer Identification Number (ITIN)"; minCount="1"},@{Name="U.S. Social Security Number (SSN)"; minCount="1"})

	
	> [!HINT] Next, we need to capture the **Label ID** for the **Confidential \ All Employees** label. 

1. [] Switch to the Azure Portal and under **Classifications** click on Labels, then expand **Confidential** and click on **All Employees**.

	!IMAGE[w2w5c7xc.jpg](\Media\w2w5c7xc.jpg)

	> [!HINT] If you closed the azure portal, open an Edge InPrivate window and navigate to ```https://portal.azure.com```.

1. [] In the Label: All Employees blade, scroll down to the Label ID and **copy** the value.

	!IMAGE[lypurcn5.jpg](\Media\lypurcn5.jpg)

	> [!ALERT] Make sure that there are no spaces before or after the Label ID as this will cause the mail flow rule to be ineffective.

1. [] Next, paste the copied value into a new txt file to use in the next step.
1. [] In an **Administrative PowerShell** window, type ```C:\Users\LabUser\Desktop\BlockInternal.ps1``` and press **Enter**. 
1. [] When prompted, provide the credentials below:

	```@lab.CloudCredential(134).Username```

	```@lab.CloudCredential(134).Password```

	> [!NOTE] If prompted to remove a transport rule, hit **Enter**.

	> [!KNOWLEDGE] This mail flow rule can be used to prevent internal only communications from being sent to an external audience.
	>
	>New-TransportRule 
	>
	>-name "Block Confidential Contoso All Employees" 
	>
	>-SentToScope notinorganization 
	>
	>-HeaderContainsMessageHeader "msip_labels" 
	>
	>-HeaderContainsWord $labeltext 
	>
	>-RejectMessageReasonText “Contoso internal messages cannot be sent to external recipients.”

	> [!KNOWLEDGE] The script runs the code below. This script is available online at https://aka.ms/labscripts
	>
	> $UserCredential = Get-Credential
	>
	>$Session = New-PSSession -ConfigurationName Microsoft.Exchange -ConnectionUri https://outlook.office365.com/powershell-liveid/ -Credential $UserCredential -Authentication Basic -AllowRedirection
	Import-PSSession $Session
	>
	> Add-Type -AssemblyName Microsoft.VisualBasic
	>
	> $labelid = [Microsoft.VisualBasic.Interaction]::InputBox('Enter the LabelId for your All Employees Label', 'LabelId')
	> 
	> If(Get-TransportRule *Delete*){Remove-TransportRule *Delete*}
	> 
	> $labeltext = "MSIP_Label_"+$labelid+"_enabled=true"
	New-TransportRule -name "Block Confidential Contoso All Employees" -SentToScope notinorganization -HeaderContainsMessageHeader  "msip_labels" -HeaderContainsWord $labeltext -RejectMessageReasonText “Contoso internal messages cannot be sent to external recipients.”

	> [!NOTE] In a production environment, customers would want to create a rule like this for each of their labels that they did not want going externally.

---

## Demonstrating Exchange Online Mail Flow Rules
[:arrow_up: Top](#exercise-6-exchange-online-irm-capabilities)

In this task, we will send emails to demonstrate the results of the Exchange Online mail flow rules we configured in the previous task.  This will demonstrate some ways to protect your sensitive data and ensure a positive user experience with the product.

1. [] Switch to @lab.VirtualMachine(Client03).SelectLink and log in with the password +++@lab.VirtualMachine(Client01).Password+++.

2. [] Open an InPrivate browsing session and browse to ```https://outlook.office365.com/owa/```.

1. [] Log in using the credentials below.

	> ```EvanG@@lab.CloudCredential(134).TenantName```
	>
	> ```pass@word1```

3. [] Send an email to Adam Smith, Alice Anderson, and yourself (```Adam Smith;Alice Anderson;@lab.User.Email```).  For the **Subject**, type ```Test Credit Card Email``` and for the **Body**, type ```My AMEX card number is 344047014854133. The expiration date is 09/28, and the CVV is 4368```, then click **Send**.

4. [] Switch to @lab.VirtualMachine(Client01).SelectLink and log in with the password +++@lab.VirtualMachine(Client01).Password+++.
5. [] Review the received email.

  !IMAGE[pidqfaa1.jpg](\Media\pidqfaa1.jpg)

	> [!Knowledge] Note that there is no encryption applied to the message.  That is because we set up the rule to only apply to external recipients.  If you were to leave that condition out of the mail flow rule, internal recipients would also receive an encrypted copy of the message.  The image below shows the encrypted message that was received externally.
	>
	>!IMAGE[c5foyeji.jpg](\Media\c5foyeji.jpg)
	>
	>Below is another view of the same message received in Outlook Mobile on an iOS device.
	>
	>!IMAGE[599ljwfy.jpg](\Media\599ljwfy.jpg)

6. [] Next, in Microsoft Outlook, click on the **New email** button.

  ^IMAGE[Open Screenshot](\Media\6wan9me1.jpg)
. [] Send an email to Evan Green, Alice Anderson, and yourself (```Evan Green;Alice Anderson;@lab.User.Email```).  For the **Subject** and **Body** type ```Another Test Contoso Internal Email```.

  ^IMAGE[Open Screenshot](\Media\d476fmpg.jpg)

8. [] In the Sensitivity Toolbar, click on **Confidential** and then **All Employees** and click **Send**.

  ^IMAGE[Open Screenshot](\Media\yhokhtkv.jpg)

9. [] In about a minute, you should receive an **Undeliverable** message from Exchange with the users that the message did not reach and the message you defined in the previous task.

  !IMAGE[kgjvy7ul.jpg](\Media\kgjvy7ul.jpg)

	> [!NOTE] This rule may take a few minutes to take effect, so if you do not get the undeliverable message, try again in a few minutes.

	> [!HINT] There are many other use cases for Exchange Online mail flow rules but this should give you a quick view into what is possible and how easy it is to improve the security of your sensitive data through the use of Exchange Online mail flow rules and Azure Information Protection.

---

===
# Exchange IRM Exercise Complete

In this exercise, we created several Exchange Online Mail Flow Rules to protect sensitive data or improve user experience.  Choose one of the exercises below or click the Next button to complete the Lab.

- [AIP Scanner Discovery](#aip-scanner-discovery)
- [Base Configuration](#configuring-azure-information-protection-policy)
- [Bulk Classification](#bulk-classification)
- [AIP Scanner CLP](#aip-scanner-classification-labeling-and-protection)
- [Security and Compliance Center](#security-and-compliance-center)
- [AIP Analytics Dashboards](#aip-analytics-dashboards)

---

===
# AIP Lab Complete
[:arrow_left: Home](#azure-information-protection)

Congratulations! You have completed the Azure Information Protection Hands on Lab. 

===

# Microsoft Cloud App Security
[:arrow_left: Home](#introduction)

## Introduction

Microsoft Cloud App Security is Microsoft **CASB** (Cloud Access Security Broker) and is a critical component of the Microsoft Cloud Security stack.
It's a comprehensive solution that can help your organization as you move to take full advantage of the promise of cloud applications, but keeps you in control through improved visibility into activity. It also helps increase the protection of critical data across cloud applications (Microsoft **and** 3rd parties).
With tools that help uncover shadow IT, assess risk, enforce policies, investigate activities, and stop threats, your organization can more safely move to the cloud while maintaining control of critical data.

The diagram below describe typical use cases for CASB's.

!IMAGE[MCAS intro](\Media\mcasintro-1.png "MCAS intro")

This lab will guide you through some of Microsoft Cloud App Security (MCAS) capabilities and top use cases.

===

## Lab environment
[:arrow_left: MCAS Home](#microsoft-cloud-app-security)

!IMAGE[Lab environment](\Media\mcaslabenvironment.png "Lab environment")

* **Client01** is a Windows 10 VM that will be used to access Office 365 and Cloud app Security management consoles and configure the log collector running on LinuxVM, using Putty.
* **LinuxVM** is an Ubuntu 18.04 computer on which we install Docker to run the Cloud App Security Discovery log collector.
* Office 365 and Cloud App Security are test tenants for the labs.



### Portals URLs

* Office 365:```https://portal.office.com```
* Cloud App Security: ```https://portal.cloudappsecurity.com```
* Security & Compliance Center: ```https://protection.office.com```
* Windows Defender ATP: ```https://securitycenter.windows.com```

===

## Labs
[:arrow_left: MCAS Home](#microsoft-cloud-app-security)

> [!ALERT] Before going to the different labs section, please be sure to complete the **[environment preparation](#mcas-environment-preparation)**.

The different Cloud App Security capabilities covered in the labs are:

* [Management](#manage-admin-access)
* [Cloud apps Discovery](#cloud-app-security-discovery-lab)
* [Information Protection](#information-protection)
* [Conditional Access App Control with Office 365](#conditional-access-app-control-with-office-365)
* [Threat Detection](#cloud-app-security-threat-detection-lab)

### Optional/Follow on labs

* [Management with PowerShell](#management-with-powershell)
* [Log collector troubleshooting](#log-collector-troubleshooting)
* [Conditional Access App Control with 3rd party apps](#conditional-access-app-control-with-3rd-party-apps)

>:question: If you have questions or want to go further in your Cloud App Security journey, join our **[Tech community](https://techcommunity.microsoft.com/t5/Microsoft-Cloud-App-Security/bd-p/MicrosoftCloudAppSecurity)** !

===

# Manage Admin Access
[:arrow_left: MCAS Home](#microsoft-cloud-app-security)

[Manage admin access:](#manage-admin-access) :clock10: 5 min

For this task, you are asked to delegate admin access to monitor a dedicated group of users for a specific region, without adding them to the Global Admin management role.


> [!KNOWLEDGE] Cloud App Security Global admin role is not the same as the Office 365 Global admin role.
> [!KNOWLEDGE] Although the Office 365 Global admins are automatically granted the Cloud App Security Global admin role, you can grant users           MCAS Global Admin role without adding them to the Office 365 Global admins.



## Delegate User Group Administration

In this lab, we are going to delegate the management of US employees to a new administrator (**mcasadminUS**). This administrator will only see those users alerts and activities for the users bieing members of the group **US employees**.

1. [] On @lab.VirtualMachine(Client01).SelectLink log in with the password +++@lab.VirtualMachine(Client01).Password+++.

1. [] In a in Private Browsing session navigate to the ```https://portal.cloudappsecurity.com```

1. [] Click on the **Gear** and under the Data Enrichement section, select **User groups**
    
	> [!HINT] The import can take **up to one hour**. Cloud App Security has to synchronize the Azure AD groups before importing them.
	> [!HINT] If the import takes too long, don't hesitate to move to the next section and come back to complete this task.

	    ^IMAGE[Open Screenshot](\Media\mgmt-import1.png "Import group")

    1. Click on **Import user group**

        !IMAGE[Import group](\Media\mgmt-import2.png "Import group")

    1. Select an app to import group from: **Office 365(AzureAD)**

        ^IMAGE[Open Screenshot](\Media\mgmt-import3.png "Import group")

	1. The user group will be seen as **Importing**.

        !IMAGE[Import group](\Media\mgmt-import4.png "Import group")	


		> [!NOTE] **While the import is happening, head over to *Delegate MCAS administration to an external admin* and come back to this portion of the lab once you've completed that section section. The import will be completed and you continue with the steps below.**

1. [] In the Cloud App Security portal```https://portal.cloudappsecurity.com```, add **mcasAdminUS** as **User group admin** for the **US employees** group.

    1. Click on the **Gear** under System, and select **Manage admin access**

        ^IMAGE[Open Screenshot](\Media\mgmt-admin1.png "New admin")

	1. Click on the **+** icon.

        ^IMAGE[Open Screenshot](\Media\mgmt-admin2.png "New admin")

    1. Type the admin user name: ```mcasadminus@@lab.CloudCredential(134).TenantName```

    	> [!HINT] The tenant name is the one provided for your lab environment. This information is available in your **Resources** section, at the top of the instructions.
    
    	 ^IMAGE[Open Screenshot](\Media\tenantcredentials.JPG)
    
        >[!KNOWLEDGE] **Group admin** has permissions to all of the data in Microsoft Cloud App Security that deals exclusively with the specific group selected here. The admin can view and modify information in Microsoft Cloud App Security only for that user group.
    	>[!KNOWLEDGE] For more details about admin roles```https://docs.microsoft.com/en-us/cloud-app-security/manage-admins```

	1. Select Role: **User group admin**

        ^IMAGE[Open Screenshot](\Media\mgmt-admin3.png "New admin")

	1. Select groups for this admin**: **US employees**

	1. Click on **Add admin**

        !IMAGE[New admin](\Media\mgmt-admin4.png "New admin")

	1. The user should be populated with the **role** and **scope** you selected.

        !IMAGE[New admin](\Media\mgmt-admin5.png "New admin")


1. [] Open a new **private** tab and connect to the [Cloud App Security portal]```https://portal.cloudappsecurity.com``` with **mcasAdminUS** and compare the activities, alerts and actions that this scoped admin can perform compared to your regular Global admin account.
     ```mcasadminus@@lab.CloudCredential(134).TenantName```
    
     ```pass@word1```

    > [!NOTE] As you can see, this administrator can not see **Discovery** data, **Files** or **Alerts** that are not related to the users from the group he/she manages.

    !IMAGE[New admin](\Media\mgmt-admin6.png "New admin")

---

## Delegate MCAS administration to an external admin
[:arrow_up: Top](#manage-admin-access)

As a **Managed Security Service Providers** (MSSPs), you are asked by your customer how you could access their environment to manage their alerts in the Cloud App Security portal.
In this lab, we will see how to answer to that question.

>[!ALERT] For this task, work with the person next to you to configure an external access for the Managed Security Service Provider to another MCAS tenant. You will need that person's tenant name that can be found in the **Resources** section, at the top of the instructions.
>[!ALERT] Both attendees need to perform the operation in parralel.
>[!ALERT]^IMAGE[Open Screenshot](\Media\tenantcredentials.JPG)

1. [] Browse to ```https://portal.cloudappsecurity.com``` and add the external MCAS admin (your neighboor's admin username) as **Security reader** in your MCAS tenant.

    1. Click on the **Gear** under System and select **Manage admin access**.

        ^IMAGE[Open Screenshot](\Media\mgmt-admin1.png "External admin")

	1. Click on the **+** icon.

        ^IMAGE[Open Screenshot](\Media\mgmt-admin2.png "External admin")

    1. Type **Admin user name**:  *Use your neighbor's credentials provided in their tenant as the external admin.

	1. Select type of role for this admin:**Security Reader**

	    ^IMAGE[Open Screenshot](\Media\tenantcredentials.JPG)

	1. Click on **Add Admin**

        ^IMAGE[Open Screenshot](\Media\mgmt-externaladmin1.png "External admin")

        >[!HINT] Note here that the icon next to the newly added admin shows that the user is external to the company.
        !IMAGE[External admin](\Media\mgmt-externaladmin2.png "External admin")

1. [] After adding the external admin, **log off** and log on again from Cloud App Security.

    ^IMAGE[Open Screenshot](\Media\mgmt-switch1.png "Log off")

3. [] Switch to the external Cloud App Security tenant where you have been added as an external admin and compare at the actions you can perform using this security role as external admin.

    !IMAGE[Switch](\Media\mgmt-switch2.png "Switch")

===

# Cloud App Security Discovery
[:arrow_left: MCAS Home](#microsoft-cloud-app-security)

On average, more than 1,100 cloud applications are used by enterprises today, of which 61% are **not** sanctioned by IT. This results in duplicate capabilities, apps not meeting compliance standards or posing a security risk to the organization without any IT oversight.
**Discovery** identifies current cloud apps, provides risk assessments and ongoing analytics and lifecycle management capabilities to control the use.


!IMAGE[Discovery](\Media\discovery3.JPG)


To provide this visibility on Shadow IT and cloud apps usage, Cloud App Security ingest and analyze network logs from proxy, firewall but also from **Windows 10** clients within or **ouside** the corporate network, using the native integration with **Windows Defender ATP**.

!IMAGE[Discovery intro](\Media\dis-intro1.png "Discovery intro")


Once the logs have been analyzed, Cloud App Security provides the visibility on **Shadow IT** and alerts you when it detects risky apps or anomalous usage.

!IMAGE[Discovery intro](\Media\dis-intro2.png "Discovery intro")


> [!NOTE] In this lab, we will simulate the upload of network logs from a SQUID proxy to analyze the apps used withing your company. We will not test the Windows Defender ATP integration at it can take up to **2h** before the logs are parsed and the results are visible in the console.

===
# Cloud App Security Discovery Lab
[:arrow_left: Discovery lab](#cloud-app-security-discovery)

* [Cloud Discovery Snapshot Report](#cloud-discovery-snapshot-report)
* [Configure and Test Continuous Reports](#configure-and-test-continuous-reports)

===

# Cloud Discovery Snapshot Report
[:arrow_left: Discovery lab](#cloud-app-security-discovery-lab)

In this lab, we will create a Discovery **Snapshot report**.
Snapshot Reports is the manual method of uploading files into Cloud App Security. This process is a great and easy way to validate your logs format of have a quick look at the Cloud App Security Discovery capability.
You can upload batches of 20 logs of 1 GB max at a time and they will parse into their own separate report. Any discovery policies you create **will not** apply to these types of reports.

To create snapshot reports:

1. [] Go to the **Discover** section and click on **Create snapshot report**.

    ^IMAGE[Open Screenshot](\Media\dis-newsnaphsot.png "Create snapshot")

1. [] In the Add data source window, use the settings below (do not close the window yet) and click on **View and verify...**.

    >|||
    >|---------|---------|
    >|Report Name| ```**Demo report**```|
    >|Description| 
    >|Data Source| **SQUID (Common)**|
    >|Anonymize private information |**Check the Box**|

1. [] Click on **View and Verify** in the window and then click on the **Download sample log** button and save it to your desktop. 

	!IMAGE[Open Screenshot](\Media\dis-createsnapshot.png "New snapshot")

    ^IMAGE[Open Screenshot](Media\dis-squiddownload.png "Download log")

1. [] Close that window.

1. [] Click on the **Browse** button and in the new window, select the log you downloaded and click **Open**.

    ^IMAGE[Open Screenshot](\Media\dis-browse.png "Browse logs")

    ^IMAGE[Open Screenshot](\Media\dis-squidselect.png "Select logs")

1. [] Now that the log has been selected, click on the **Create** button to create your report.

    ^IMAGE[Open Screenshot](\Media\dis-squidcreate.png "Create snapshot")

1. [] Your report will then be processed. It will take a couple of minutes before it's marked as **Ready**.

    !IMAGE[Report processing](\Media\dis-processing.png "Report processing")

    !IMAGE[Report processing](\Media\dis-reportready.png "Report processing")

1. [] Once your report is marked as ready, click on the text **Ready**. You will be redirected to your snapshot report where you can start exploring the discovered apps, users, IPs. Your instructor can give you more details on how to use the dashboard.

    ^IMAGE[Open Screenshot](\Media\dis-reportready2.png "Report ready")

    !IMAGE[Report dashboard](\Media\dis-dashboard.png "Report dashboard")

    !IMAGE[Report dashboard -risk](\Media\dis-risk.png "Report dashboard - risk")

===

## Configure and Test Continuous Reports
[:arrow_left: Discovery lab](#cloud-app-security-discovery-lab)

In this lab, we will implement Continuous reports.
Continuous reports analyze all logs that are forwarded from your network using Cloud App Security. They provide improved visibility over all data, and **automatically** identify anomalous use using either the Machine Learning anomaly detection engine or by using custom policies that you define.

These reports can be created by connecting in the following ways:

* **Windows Defender ATP Integration**: Cloud App Security integrates with Windows Defender Advanced Threat Protection (ATP) natively, to simplify rollout of Cloud Discovery, extend Cloud Discovery capabilities beyond your corporate network, and enable machine-based investigation.

* **Log Collector**: Log collectors enable you to easily automate log upload from your network. The log collector runs on your network and receives logs over Syslog or FTP.

* **Zscaler Integration**: If you work with both Cloud App Security and Zscaler, you can integrate the two products to enhance your security Cloud Discovery experience. Together, Cloud App Security and Zscaler provide seamless deployment of Cloud Discovery, automatic blocking of unsanctioned apps, and risk assessment directly in the Zscaler portal.

!IMAGE[Zscaler](\Media\SWG.JPG)

In this lab you will perform the configuration of a **Log collector**.


> [!ALERT] After completing this portion of the lab and validating that your logs have been successfully uploaded and processed by MCAS, you will not immediately see a loaded Discovery Dashboard. Due to Cloud Discovery logs being  parsed **twice a day**.


> [!HINT] For this lab, the Docker engine has been pre-installed on **LinuxVM** in your lab environment, but normally you have to configure it on your host using the command below.
> [!HINT] The command downloads a script installing the Docker engine on your host computer (Ubuntu in this case) and pull the latest Cloud App Security collector image from the Docker library.

> [!KNOWLEDGE] curl -o /tmp/MCASInstallDocker.sh https://adaprodconsole.blob.core.windows.net/public-files/MCASInstallDocker.sh && chmod +x /tmp/MCASInstallDocker.sh; /tmp/MCASInstallDocker.sh 

---

### Create a data source and a log collector in the Cloud App Security Portal

1. [] Switch to @lab.VirtualMachine(Client01).SelectLink and log in with the password +++@lab.VirtualMachine(Client01).Password+++.


1. [] Create a new tab in the in a  Private window and browse to ```https://portal.cloudappsecurity.com```.

       >```@lab.CloudCredential(134).Username```
       >
       >```@lab.CloudCredential(134).Password```

1. [] In the Cloud App Security dashboard, click on the **Gear** icon and select **Log collectors**.

       ^IMAGE[Open Screenshot](\Media\dis-settings.png "Settings")

1. [] On the **Data sources tab**, click the **Add data source...** button.

       ^IMAGE[Open Screenshot](\Media\dis-newsource.png "New data source")

1. [] In the Add data source window, use the settings below (do not close the window yet):

    >|||
    >|---------|---------|
    >|Name| ```**SquidLogs**```|
    >|Source| **SQUID (Common)**|
    >|Receiver type| **FTP**|
    >|Anonymize private information |**Check the Box**|
    >
    >^IMAGE[Open Screenshot](\Media\dis-squidsource.png)

    > [!KNOWLEDGE] In this lab we use FTP as the receiver type but usually companies will use Syslog.

1. []Click **Add** in the **Add data source** dialog.

	^IMAGE[Open Screenshot](\Media\dis-addsource.png "Add source")

    > [!HINT] We just created a data source which is the logical representation of the network appliance data source type the log collector will receive.

1. [] Next, click on the **Log collectors tab** and click the **Add log collector...** button.

    !IMAGE[Add log collector](\Media\dis-addlogcollector.png "Add log collector")

1. [] In the Create log collector dialog, provide the settings below and click the **Update** button.

    |||
    |-----|-----|
    |Name|```LogCollector```
    |Host IP address|```192.168.141.125```
    |Data source(s)|**SquidLogs**

    ^IMAGE[Open Screenshot](\Media\dis-createlogcollector.png "Create log collector")

1. [] After clicking on the **Update** button, follow the required steps to create your log collector instance which will be hosted within the **LinuxVM**. You'll have to use the provided command line in **Putty** in the following steps to configure your log collector.
    > [!ALERT] Do not close this window until your log collector is up and running!

    !IMAGE[Create log collector command](\Media\dis-addlogcollectortoken.png "Create log collector")

    > [!KNOWLEDGE] This command line contains the different parameters to instantiate a new log collector on the Linux host:
    > [!KNOWLEDGE] An API token to connect to Cloud App Security for uploading the logs: *1f5b5fb2a0d778e3d57f26ca5ab11574db0751166477940528ccf19a7c4*
    > [!KNOWLEDGE] The docker parameters to configure the log collector container: *docker run ...*

1. [] **Copy the command line** provided at the end of the previous step and **minimize** the browser. 

1. [] Open **Putty (64-bit)**. You should have the icon on your desktop.

    ^IMAGE[Open Screenshot](\Media\dis-putty.png "Putty")

1. [] In the PuTTY Configuration window, enter ```**192.168.141.125**``` and click **Open**.

    ^IMAGE[Open Screenshot](\Media\dis-puttyconfig.png "Putty config")

1. [] At the Putty warning message, click **Yes**.

    > [!KNOWLEDGE] This warning is due to the ssh certificate provided by LinuxVM not being trusted by your computer. You can safely ignore this warning in this lab.

    ^IMAGE[Open Screenshot](\Media\dis-puttywarning.png "Putty warning")

1. [] Log in using the credentials below.
    >|Username|Password|
    >|---|---|
    >|user01|Passw0rd1|

    > [!HINT] The password doesn't appear in the command prompt when you type it. You can safely press enter to validate the credentials.

    !IMAGE[Open Screenshot](\Media\dis-puttylogin.png)

1. [] Type the command below and press **Enter**. Provide the user password when prompted.

    ***sudo -i***

    ***Passw0rd1***

     ^IMAGE[Open Screenshot](\Media\dis-sudo.png)

    > [!KNOWLEDGE] The previous command elevates your permissions in the Linux environment like the UAC prompt would do on a Windows machine.

1. [] Return to the **Create log collector** dialog.

1. [] Copy the **collector configuration** command from step 2 and run it in the PuTTY window.

     ^IMAGE[Open Screenshot](\Media\dis-addlogcollectorcopy.png "Copy token")
     !IMAGE[New container](\Media\dis-newcontainer.png "New container")

    > [!KNOWLEDGE] The output of this command is the id of the newly created container/log collector.

1. [] Launch the **WinSCP** application from **Client01** start menu.

    ^IMAGE[Open Screenshot](\Media\dis-winscp.png "WinSCP")

1. [] Use the details below in the WinSCP window to connect to the log collector **FTP** service.

    |File Protocol|Host name|User name|Password|
    |-----|-----|-----|-----|
    |**FTP**|```192.168.141.125```|```discovery```|```BP98Jw4Ns*zpTFrH```|

    ^IMAGE[Open Screenshot](\Media\dis-winscpconnection.png "WinSCP connection")

    > [!HINT] this information was provided during the log collector creation.

1. [] You should then be able to see a folder with your data source name (**SquidLogs**).

    !IMAGE[WinSCP connection](\Media\dis-winscpfolder.png "WinSCP connection")

    > [!ALERT] If you are **not** able to connect to the log collector FTP service, verify that you successfully created the new log collector instance within Putty in previous steps.

1. [] On the left pane, move to the **Desktop** folder and drag your example Squid log (From Step 7) into the folder named for your data source (**SquidLogs**). After some minutes, the log collector will upload your logs.

    !IMAGE[Log upload](\Media\dis-winscplogupload.png "Log upload")
    !IMAGE[Log upload](\Media\dis-winscplogupload2.png "Log upload")
    !IMAGE[Log upload](\Media\dis-winscplogupload3.png "Log upload")

1. [] Return to the Cloud App Security portal and click on the **Gear** icon and select **Governance log**.

    ^IMAGE[Open Screenshot](\Media\dis-governancelog.png "Settings Governance log")

1. [] Verify the status of the uploaded logs.

    ^IMAGE[Open Screenshot](\Media\dis-loguploaded.png "Log uploaded")

    > [!NOTE] The status you see is the parsing status of the logs. Parsing status can be successful, pending or failed.

1. [] You can also verify the **Last data received** date on the **Data sources** tab under **Automatic log upload** in the settings.

    !IMAGE[Last data received](\Media\dis-lastreceived.png "Last data received")

1. [] Go to the **Cloud Discovery Dashboard** to verify the discovered apps. As explained at the beginning of the lab, the data you uploaded is probably not already available as the data is analyzed **twice a day**.

    !IMAGE[Open Screenshot](\Media\dis-discoverydashboard.png "Discovery dashboard")

    !IMAGE[Open Screenshot](\Media\dis-discoverydata.png "Discovery data")

===

# Information Protection
[:arrow_left: MCAS Home](#microsoft-cloud-app-security)

In a perfect world, all your employees understand the importance of information protection and work within your policies. But in a real world, it's probable that a partner who works with accounting uploads a document to your Box repository with the wrong permissions, and a week later you realize that your enterprise's confidential information was leaked to your competition.
Microsoft Cloud App Security helps you prevent this kind of disaster before it happens.

!IMAGE[IP](\Media\IPCAS.JPG)

===
## Information Protection Lab

File policies are a great tool for finding threats to your information protection policies, for instance finding places where users stored sensitive information, credit card numbers and third-party ICAP files in your cloud. With Cloud App Security, not only can you detect these unwanted files stored in your cloud that leave you vulnerable, but you can take im/mediate action to stop them in their tracks and lock down the files that pose a threat.
Using Admin quarantine, you can protect your files in the cloud and remediate problems, as well as prevent future leaks from occurring.

* [Apply AIP classification to SSN documents:](#apply-aip-classification-to-ssn-documents)
* [Quarantine sensitive PDF for review:](#quarantine-sensitive-pdf-for-review)
* [Test our policies:](#test-our-policies)

## Apply AIP classification to SSN documents
[:arrow_up: Top](#information-protection)

In this lab, we are going to configure a file policy to apply an **Azure Information Protection** template on documents containing social security numbers. This method could be compared to the **Azure Information Protection Scanner** for documents that are stored on file servers.

1. [] In the Cloud App Security portal, go to **Control** and then click on **Policies.**

    !IMAGE[Open Screenshot](\Media\info-policies.png "Policies")

1. [] Create a Policy click on  **File policy**.

    ^IMAGE[Open Screenshot](\Media\info-newpolicy.png "New policy")

1. [] Provide the following settings to that policy:

    >|||
    >|---------|---------|
    >|Policy Name| ```Protect SSN documents in sensitive site```|
    >|Files matching all of the following| **App equals Box** |
    >|Apply to| **All Files** |


    ^IMAGE[Open Screenshot](\Media\allfilesBox1.png)  

1. [] In the inspection method, select **Data Classification Service**.

    ^IMAGE[Open Screenshot](\Media\info-dcs.png "DCS")

    > [!KNOWLEDGE] **Microsoft Data Classification Service** provides a **unified** information protection experience across Office 365, Azure Information Protection, and Microsoft Cloud App Security.
    > [!KNOWLEDGE]
    > [!KNOWLEDGE] The classification service allows you to extend your data classification efforts to the third-party cloud apps protected by Cloud App Security, using the decisions you already made across an even greater number of apps.

1. [] Click on **Choose inspection type** and then on **sensitive information type**.

    ^IMAGE[Open Screenshot](\Media\info-type.png "SSN type")


1. Search and select the **all** the information types matching ```SSN``` and click on **Done**.

    > [!HINT] Be sure to select the checkboxes as clicking on the name do not select the information type.

    !IMAGE[SSN type](\Media\info-ssn.png "SSN type")

1. [] Click on the **Unmask the last 4 characters of a match** and the **Create an alert for each matching file** checkboxes.

    ^IMAGE[Open Screenshot](\Media\info-unmask.png "Unmask")

    > [!KNOWLEDGE] In production scenarios, as you will probably have thousands of matches, you will **not** create alerts but use the **policy matches** approach instead.

1. [] In the Governance actions, click on **Box** and select **Apply classification label**. Select the **Highly Confidental - All Employees** label. 

    > [!ALERT] If you are not able to select Azure Information Protection templates, verify that you configured the integration in the prerequisites section or that you waited the 1h for the classifications to sync. In addition log out of the Cloud App Security Portal and log back in and see and if you're able to apply the label. 

  !IMAGE[gov](\Media\Boxgovssn.JPG)


1. [] Click **Create** to finish the policy creation.

---

## Quarantine sensitive PDF for review
[:arrow_up: Top](#information-protection)

In this lab, we are going to configure a file policy to quarantine sensitive PDF files that are shared externally, so an admin can review those files and validate if they could or not be shared externally. **Admin quarantine** can also be used to isolate files that should not have been uploaded to cloud storage apps.

1. [] In the Cloud App Security portal, go to **Control** and then click on **Policies.**

    !IMAGE[Open Screenshot](\Media\info-policies.png "Policies")

1. [] Create a Polick and click on **File policy** that will determine which files should be placed in quarantine.

    ^IMAGE[Open Screenshot](\Media\info-newpolicy.png "New policy")

1. [] Provide the following settings to that policy:

    >|Policy name|Files matching all of the following|
    >|---------|---------|
    >|```Quarantine sensitive pdf```| Extension equals pdf **and** Access level equals Public, External|

    ^IMAGE[Open Screenshot](\Media\info-policy3.png "New policy")

1. [] Check the **Create an alert for each matching file** checkBox. 

1. [] In Governance actions of the policy, select **Put in admin quarantine** for Box and click on the **Create** button.

    !IMAGE[Unmask](\Media\Boxgovadmin.JPG)

---

## Test our Policies
[:arrow_up: Top](#information-protection)

We are now going to test our files policies by performing the following actions.

1. [] On Client01, unzip the content of the **Demo files.zip**.

1. [] Go to the **Box** files ```https://app.box.com/folder/0```

1. [] Upload the unzipped files to the site.

    ^IMAGE[Open Screenshot](\Media\info-uploadbox.png "Upload")

1. [] After upload is complete, **share** the PDF document named **Protect with Microsoft Cloud App Security proxy.pdf**

    ^IMAGE[Open Screenshot](\Media\info-share1.png "Upload")

    ^IMAGE[Open Screenshot](\Media\info-share2.png "Upload")

1. [] Cloud App Security will now scan those documents and search for matches to your created policies.

    > [!HINT] The scan can take **several minutes** before completion.

1. [] To monitor the evolution of the scan, go back to Cloud App Security, select **Investigate** and open the **Files** page.

    ^IMAGE[Open Screenshot](\Media\info-files1.png "Search files")

1. [] You can search for the files you uploaded using different criteria, like **file name**, **type**, ... or just look at all the files discovered by Cloud App Security. When a policy match is discovered, you will notice it on this page by looking at the icones next to the file name. You will also have icons related to the applied **Governance action** (AIP label or Admin Quarantine in our lab).

    ^IMAGE[Open Screenshot](\Media\Boxfilesmatch.png)

1. [] To open the details of the file, click on its name. You can see there the matched policies and the scan status of the files.

    ^IMAGE[Scan status](\Media\info-files5.png "Scan status")

1. [] You can also view the related governance actions, like applying the Azure Information classification or moving the file to the quarantine folder, at the file level or in the **Governance log**.

    !IMAGE[Governance log](\Media\Boxgovlog.jpg)

1. [] As we configured **Alerts** in our lab, you can also review the related alerts in the **Alerts page**.

    ^IMAGE[Alert](\Media\Boxalert1.png)

    ^IMAGE[Alert](\Media\Boxalert2.png)

1. [] If you go back to **Box**, you will also notice that the quarantined files will be replaced by **placeholders**. The original file will be moved to the **Quarantine**.

    > [!KNOWLEDGE]  For Box, the quarantine folder location and user message **can't be customized**. The folder location is the drive of the admin who connected Box to Cloud App Security. For **SharePoint and OneDrive**, the location and the message can be customized in Cloud App Security settings.

    ^IMAGE[Open Screenshot](\Media\boxquarantine2.png)

    ^IMAGE[Open Screenshot](\Media\boxquarantine1.png)

1. [] The other way to review the policy matches is to go back to the **Policies page** and look at the **matches number**.

    !IMAGE[Matches](\Media\info=matches.png)

===

# Conditional Access App Control with Office 365
[:arrow_left: MCAS Home](#microsoft-cloud-app-security) :clock10: 15 min

## Introduction

Conditional Access App Control utilizes a reverse proxy architecture and is uniquely integrated with Azure AD conditional access.
Azure AD conditional access allows you to enforce access controls on your organization’s apps based on certain conditions. The conditions define who (for example a user, or group of users) and what (which cloud apps) and where (which locations and networks) a conditional access policy is applied to. After you’ve determined the conditions, you can route users to the Microsoft Cloud App Security where you can protect data with Conditional Access App Control by applying access and session controls.

Conditional Access App Control enables user app access and sessions to be **monitored and controlled in real time** based on access and session policies.

!IMAGE[AAD portal](\Media\caac1.jpg)

> [!NOTE] **App Control Access and Session policies give you the capability to the following:**
* **Block on download**: You can block the download of sensitive documents. For example, on unmanaged devices.
* **Protect on download**: Instead of blocking the download of sensitive documents, you can require documents to be protected via encryption on download. This ensures that the document is protected, and user access is authenticated, if the data is downloaded to an untrusted device.
* **Monitor low-trust user sessions**: Risky users are monitored when they sign into apps and their actions are logged from within the session. You can investigate and analyze user behavior to understand where, and under what conditions, session policies should be applied in the future.
* **Block access**: You can completely block access to specific apps for users coming from unmanaged devices or from non-corporate networks.
* **Create read-only mode**: By monitoring and blocking custom in-app activities you can create a read-only mode to specific apps for specific users.
* **Restrict user sessions from non-corporate networks**: Users accessing a protected app from a location that is not part of your corporate network, are allowed restricted access and the download of sensitive materials is blocked or protected.

===
# App Control Lab
[:arrow_left: Discovery lab](#cloud-app-security-discovery)

* [App Control Configuration](#app-control-configuration)
* [Testing the Session Policy](#testing-the-session-policy)

===

## App Control Configuration

1. [] Go to the Azure portal ```https://portal.azure.com``` and open the **Azure Active Directory** blade.

   ^IMAGE[Open Screenshot](\Media\aad-1.png)

1. [] Scroll down to **Security** and click on **Conditional Access**.

   ^IMAGE[Open Screenshot](\Media\aad-2.png)

1. [] Create a new conditional access policy with the following settings:

   |Name|Assignments|Apps|
   |-----|-----|-----|
   |Office365 AppControl|All users|Exchange, SharePoint|

    1. Click on **New Policy**

	    ^IMAGE[Open Screenshot](\Media\cond-policy-1.png)

    1. Name it ```Office365 App Control```

    1. Under assignments: Click on **All users** and then **Done**

       ^IMAGE[Open Screenshot](\Media\cond-policy-2.png)

    1. Go to the next section: Cloud Apps: Select Apps and choose Office 365 Exchange Online and Office 365 SharePoint Online and **Done**

       ^IMAGE[Open Screenshot](\Media\cond-policy-3.png)

    1. Under **Acess Controls**, click on **Session** and check off **Use Conditional Access App Control** 

       ^IMAGE[Open Screenshot](\Media\cond-policy-4.png)

    1. Click on **ON** in *Enable the policy* and click **Create**

       ^IMAGE[Open Screenshot](\Media\cond-policy-5.png)

1. [] Sign out of the Azure Portal and close you browser.

1. [] Open your browser and go to the Exchange Web App ```https://outlook.office.com```.

1. [] Connect using :

    >```@lab.CloudCredential(134).Username```
    >
    >```@lab.CloudCredential(134).Password```

    > [!HINT] This is done to force the use of conditional access. Once a session has been redirected to Cloud App Security, you will be able to configure the application for App Control in Cloud App Security.

1. [] Go back to Cloud App Security ```https://portal.cloudappsecurity.com```, click on the **Gear** icon and click on **Conditional Access App Control**.
  
   ^IMAGE[Open Screenshot](\Media\appc-office-1.png)

    > [!HINT] You will see that **Exchange Online** has now appeared as an application and can now be configured.

   ^IMAGE[Open Screenshot](\Media\appc-office-2.png)

1. [] Click on **Continue setup** to enable session control and click on **Add**.

   ^IMAGE[Open Screenshot](\Media\appc-office-3.png)

   ^IMAGE[Open Screenshot](\Media\appc-office-4.png)
  
1. [] You are now notified that the application is under Session Control.

   !IMAGE[Setup](\Media\appc-office-5.png)

1. [] On the left hand side click on **Control** and then **Policies**.

   !IMAGE[Open Screenshot](\Media\appc-office-6.png)

1. [] Click on **Create Polic** and click on **Session policy**.

        ^IMAGE[Open Screenshot](\Media\appc-office-7.png)

    1. **Name**: ```Proxy - Block sensitive files download```

    1. Under Session Control Type choose **Control filedownload (with DLP)**

        ^IMAGE[Open Screenshot](\Media\appc-office-8.png)

    1. Add Activity Filters: **Device Tag** does not equal **Compliant, Domain joined**

    1. **App** equals **Office 365 Exchange Online and Office 365 SharePoint Online**

       !IMAGE[Session policy](\Media\appc-office-9.png)

   1. Content inspection check **Enabled**
   Include files that match a preset expression anc choose US: **PII: Social Security Number**

       !IMAGE[Session policy](\Media\appc-office-10.png)

   1. Under Actions: go to **Block**

   1. Click: **Customize block message**: ```This file containes SSN information and cannot be downloaded on non-coporate devices.```

   1. Click: Verify that  **Create an alert for each matching event with the policy's severity** is checked. 

   1. Click: **Create**

       !IMAGE[Session policy](\Media\appc-office-11.png)

====

# Testing the Session Policy
[:arrow_left: Home](#app-control-labs)

Now is time to test our configuration. We will here simulate the userexperience while accessing company apps protected by Cloud App Security from an unmanaged device

1. [] Sign out, close you browser and open the Exchange Web App ```https://outlook.office.com```. Use the following credentials to connect:
  
   >```@lab.CloudCredential(134).Username```
   >
   >```@lab.CloudCredential(134).Password```

1. You should receive the following message, as you are redirected through Cloud App Security before accessing the application.
  
  Click **Continue to Exchange Online**.

   !IMAGE[Warning](\Media\appc-office-12.png)

1. [] You are now directed to Exchange Online and your session is now passing **through** Cloud App Security.

    > [!HINT] By taking a look at the **URL**, you can verify that your session is actually being redirected to **Cloud App Security**.

   !IMAGE[Session](\Media\appc-office-13.png)

1. [] To test our policy, perform the following:

    1. Create a new mail and attach the Word document named **Personal employees information.docx** and the Excel spreadsheet named **Workplace Innovation.xlsx** stored on **Client01** desktop. Send the mail to your user, ```@lab.CloudCredential(134).Username```

       !IMAGE[Test](\Media\appc-office-14.png)

    1. [] Wait until you receive your email in the webmail. 

    1. Once the message is received, click on the attached document **Personal employees information.docx**. This will open the file preview.
    As you can see, the user can access the document using the Office Online app.

        !IMAGE[Warning](\Media\appc-office-15.png)

    1. [] Try now to download the **Personal employees information.docx** document. As this file contains social security numbers, the download will be blocked and will trigger an alert in Cloud App Security.

       !IMAGE[Test](\Media\appc-office-16.png)
    
       !IMAGE[Test](\Media\appc-office-17.png)

    1. [] Now let's try to download the **Workplace Innovation.xlsx** spreadsheet. As this file **do not** contain social security numbers, the download will be allowed.

       !IMAGE[Test](\Media\appc-office-18.png)
    
       !IMAGE[Test](\Media\appc-office-19.png)
    
       !IMAGE[Test](\Media\appc-office-20.png)
    
       !IMAGE[Test](\Media\appc-office-21.png)

> [!KNOWLEDGE] We just demonstrated App Control capabilities to go further than just allow/block scenarios, based on session risks. This capability can open many scenarios, like BYOD access for example.

====

# Reviewing the Alerts
[:arrow_left: Home](#labs)

Now that we validated our configuration, let's go back to the admin view.

1. [] Go back to the Cloud App Security console ```https://portal.cloudappsecurity.com```

1. [] Go to the **Alerts** page.

   ^IMAGE[Open Screenshot](\Media\appc-admin-1.png)

1. [] Click on the alert generated by our policy.

   !IMAGE[Menu](\Media\appc-admin-2.png)

1. [] On the alert page, you can see that the **admin** user tried to download a file named **Personal employees information.docx** but **Session control** blocked the download. You also see the name of the policy that triggered the alert.

   !IMAGE[Menu](\Media\appc-admin-3.png)

1. [] To go further in the investigation, click on  **View all user activity**. This will redirect you to the Activity log where you can see all the user activities.

   !IMAGE[Menu](\Media\appc-admin-4.png)

1. [] By looking at the user activities, you can follow her/his trace:

    1. Below, you can see that the user was **redirected** to Cloud App Security

        !IMAGE[Menu](\Media\appc-admin-5.png)

    1. Here, you can see that during her/his session, the user **successfuly downloaded** a file named **Worplace Innovation.xlsx**, as this file didn't match any blocking policy.

        !IMAGE[Menu](\Media\appc-admin-6.png)

===

# Cloud App Security: Threat Detection Lab
[:arrow_left: MCAS Home](#microsoft-cloud-app-security)

Cloud App Security provides several threats detection policies using machine learning and **user behavior analytics** to detect suspicious activities across your different applications.
Those policies are enabled by default and after an initial learning period, Cloud App Security will start alerting you when suspicious actions like activity from anonymous IP addresses, infrequent country, suspicious IP addresses, impossible travel, ransomware activity, suspicious inBox forwarding configuration or unusual file download are detected.

!IMAGE[Thret protection](\Media\tp-intro.png)

===

> [!ALERT] It can take **up to 24 hours** for the auditing in Office 365 to be configured, meaning that Cloud App Security will not receive the activities events. As many alerts relies on activities events to work, we will be using a pre-populated tenant for this portion of the lab so we can see alerts and have the ability to investigate them.

> [!HINT] LOG OUT OF YOUR CURRENT CLOUD APP SECURITY TENANT AND LOG BACK IN USING THE CREDENTIALS BELOW.

|Portal               |Username                   |Password
|----------------- |----------------------------------- |---------------------
| ```https://portal.cloudappsecurity.com``` | viewer@emslab.tech |EventP@ssword

**URLs**

* Office 365: ```https://portal.office.com```
* Cloud App Security: ```https://portal.cloudappsecurity.com```

## Labs

* [Anonymous access:](#anonymous-access)
* [Impossible travel:](#impossible-travel)
* [Activity from infrequent country:](#activity-from-infrequent-country)
* [Malware detection:](#malware-detection)
* [Email exfiltration using suspicious inBox forwarding:](#email-exfiltration-using-suspicious-inbox-forwarding)
* [Ransomware activity:](#ransomware-activity)
* [Suspicious application consent:](#suspicious-application-consent)

===

## Anonymous access

[:arrow_up: Top](#cloud-app-security-threat-detection-lab)

This detection identifies that users were active from an IP address that has been identified as an anonymous proxy IP address. These proxies are used by people who want to hide their device’s IP address, and may be used for malicious intent. This detection uses a machine learning algorithm that reduces "false positives", such as mis-tagged IP addresses that are widely used by users in the organization.

### Investigate

As your authentication during the previous steps came from an anonymous IP address, it will be detected as suspicious by Cloud App Security.

1. [] Go back to the Cloud App Security portal and review the alerts.

   !IMAGE[MCAS alerts menu](\Media\td-alerts.png "Security Alerts")

   You will see an alert similar  to this one:

   !IMAGE[TOR alert](\Media\td-toralert.png "TOR alert")

2. [] Click on the alert to open it.
   You see in this page more information on the alert and the related activities:

   !IMAGE[TOR alert](\Media\td-toralert-details.png "TOR alert details")

3. [] Click on the activities to get more information on the specific activity, the user and the IP address:

   !IMAGE[TOR alert](\Media\td-toralert-details-user.png "TOR alert user")
   !IMAGE[TOR alert](\Media\td-toralert-details-ip.png "TOR alert IP address")

4. [] You can go further in your investigation by looking at the related actions performed during that session by clicking on the “investigate in activity log" button:

   !IMAGE[TOR alert](\Media\td-toralert-details-activities.png "TOR alert activities")

5. [] You will then be redirected to the activity log where you will be able to investigate on the actions performed during that session, like configuration changes or data exfiltration.

---

## Impossible travel

[:arrow_up: Top](#cloud-app-security-threat-detection-lab)

This detection identifies two user activities (is a single or multiple sessions) originating from geographically distant locations within a time period shorter than the time it would have taken the user to travel from the first location to the second, indicating that a different user is using the same credentials. This detection uses a machine learning algorithm that ignores obvious "false positives" contributing to the impossible travel condition, such as VPNs and locations regularly used by other users in the organization. The detection has an initial learning period of seven days during which it learns a new user’s activity pattern.

### Investigate

As the first and the second authentication came from distinct locations, Cloud App Security will detect that those time to travel between those two locations was to short and will then alert you.

1. [] Go back to the Cloud App Security portal and review the alerts.

   !IMAGE[MCAS alerts menu](\Media\td-alerts.png "Security Alerts")

   You will see an alert similar  to this one:

   !IMAGE[Impossible travel alert](\Media\td-impossibletravelalert.png "Impossible travel alert")

2. [] The investigation steps are similar to the anonymous access but by looking at the IP address details and the **ISP**, you will be able to determine the possible risk:

   !IMAGE[Impossible travel alert](\Media\td-impossibletravelalert-details.png "Impossible travel alert details")

	> [!KNOWLEDGE] To reduce the amount of false positives, edit the impossible travel policy. 
	>  
	> **You can adjust the policy by clicking on the alert and *Resolve* and click on *Adjust policy.**
	> 
	> Each anomaly detection policy can be independently scoped so that it applies only to the users and 
	groups you want to include and exclude in the policy. For example, you can set the Activity from infrequent county detection to ignore a specific user who travels frequently.
	> 
	> **To scope an anomaly detection policy:**
	> 
    > 1. Click Control and then Policies, and set the Type filter to Anomaly detection policy.
	> 2. Click on the policy you want to scope. 
    > 3. Under Scope, change the drop-down from the default setting of All users and groups, to Specific users and groups. 
	> 4. Select Include to specify the users and groups for whom this policy will apply. Any user or group not selected here won't be    considered a threat and won't generate an alert.
	> 5. Select Exclude to specify users for whom this policy won't apply. Any user selected here won't be considered a threat and won't generate an alert, even if they're members of groups selected under Include.
	> 
	> **Sensitivity Slider:**
	> 
    > You can set the sensitivity slider to determine the level of anomalous behavior needed before an alert is triggered. For example, if you set it to low, it will suppress Impossible Travel alerts from a user’s common locations, and if you set it to high, it will surface such alerts.
	> 
	>  !IMAGE[Impossible Travel Sensitivity Bar](\Media\updatedimpossibletravel.JPG)  

---

## Activity from infrequent country

[:arrow_up: Top](#cloud-app-security-threat-detection-lab)

This detection considers past activity locations to determine new and infrequent locations. The anomaly detection engine stores information about previous locations used by users in the organization. An alert is triggered when an activity occurs from a location that wasn't recently or never visited by any user in the organization.

### Investigate

After an initial learning period, Cloud App Security will detect that this location was not used before by your user or other people within the organization and will then alert you.

1. [] Go back to the Cloud App Security portal and review the alerts.

   !IMAGE[MCAS alerts menu](\Media\td-alerts.png "Security Alerts")

   You will see an alert similar  to this one:

   !IMAGE[Infrequent country alert](\Media\td-infrequentcountryalert.png "Infrequent country alert")

2. [] The investigation steps are similar to the anonymous access but by looking at the IP address details and the ISP, you will be able to determine the possible risk. In this specific example, we see it’s coming from a TOR IP, so this authentication is suspicious:

   !IMAGE[Infrequent country alert](\Media\td-infrequentcountryalert-details.png "Infrequent country alert details")

	> [!NOTE] Possible resolution options are available on the top bar for single click remediation. 

	!IMAGE[Infrequent country alert](\Media\infrequentcountry1.JPG)


---

## Malware detection

[:arrow_up: Top](#cloud-app-security-threat-detection-lab)

This detection identifies malicious files in your cloud storage, whether they're from your Microsoft apps or third-party apps. Microsoft Cloud App Security uses Microsoft's threat intelligence to recognize whether certain files are associated with known malware attacks and are potentially malicious. This built-in policy is disabled by default. Not every file is scanned, but heuristics are used to look for files that are potentially risky. After files are detected, you can then see a list of **Infected files**. Click on the malware file name in the file drawer to open a malware report that provides you with information about that type of malware the file is infected with.

### Investigate

1. [] Go back to the Cloud App Security portal and review the alerts.

   !IMAGE[MCAS alerts menu](\Media\td-alerts.png "Security Alerts")

   You will see an alert similar  to this one:

   !IMAGE[Malware detected alert](\Media\td-malwarealert.png "Malware detected alert")

2. [] Click on the alert to open it. You see in this page more information on the alert and the related activities:

   !IMAGE[Malware detected alert](\Media\td-malwarealert-details.png "Malware detected alert")

3. [] In the alert, you have more information on the file and its location, but also the malware that we identified:

   !IMAGE[Malware family](\Media\td-malwarefamily.png "Malware family")

4. [] Click on the malware type link to have access to the Microsoft Threat Intelligence report regarding this file:

   !IMAGE[Malware family](\Media\td-malwarefamilymti.png "Malware family")

5. [] Back in the alert, you can scroll down to the related activities. There, you will have more information on how the file was uploaded to OneDrive and possibly who downloaded it:

   !IMAGE[Malware family](\Media\td-malwarealert-activities.png "Malware family")

---

## Email exfiltration using suspicious inBox forwarding

[:arrow_up: Top](#cloud-app-security-threat-detection-lab)

This detection looks for suspicious email forwarding rules, for example, if a user created an inBox rule that forwards a copy of all emails to an external address.

### Investigate

As the rules redirects your user’s emails to a suspicious external address, Cloud App Security will detect this rule creation and will then alert you.

1. [] Go back to the Cloud App Security portal and review the alerts.

   !IMAGE[MCAS alerts menu](\Media\td-alerts.png "Security Alerts")

   You will see an alert similar  to this one:

   !IMAGE[Suspicious forwarding alert](\Media\td-suspiciousforwardingalert.png "Suspicious forwarding alert")

2. [] Click on the alert to open it. You see in this page more information on the alert, like the **destination address** and the related activities:

   !IMAGE[Suspicious forwarding alert](\Media\td-suspiciousforwardingalert-details.png "Suspicious forwarding alert")

3. [] With this information, you can now go back to the user to remove this rule but also investigate in Exchange trace logs which emails were sent to that destination address.

---

## Ransomware activity

Cloud App Security extended its ransomware detection capabilities with anomaly detection to ensure a more comprehensive coverage against sophisticated Ransomware attacks. Using our security research expertise to identify behavioral patterns that reflect ransomware activity,Cloud App Security ensures holistic and robust protection. If Cloud App Security identifies, for example, a high rate of file uploads or file deletion activities it may represent an adverse encryption process. This data is collected in the logs received from connected APIs and is then combined with learned behavioral patterns and threat intelligence, for example, known ransomware extensions. For more information about how Cloud App Security detects ransomware, see Protecting your organization against ransomware.

### Investigate

As the rules redirects your user’s emails to a suspicious external address, Cloud App Security will detect this rule creation and will then alert you.

1. [] Go back to the Cloud App Security portal and review the alerts.

   !IMAGE[MCAS alerts menu](\Media\td-alerts.png "Security Alerts")

   You will see an alert similar  to this one:

   !IMAGE[Ransomware alert](\Media\td-ransomwarealert.png "Ransomware alert")

2. [] Click on the alert to open it. You see in this page more information on the impacted user, the number of encrypted files, the location of the files and the related activities:

   !IMAGE[Ransomware alert](\Media\td-ransomwarealert-details.png "Ransomware alert")

3. [] Now that we’ve seen the alert, let’s go back to the policies:

   !IMAGE[Policies](\Media\td-policies.png "Policies")

4. [] Search for the “Ransomware activity” policy and open it:

   !IMAGE[Ransomware policy](\Media\td-policiesransomware.png "Ransomware policies")

5. [] At the bottom of the policy, review the possible alerts and governance actions:

   !IMAGE[Ransomware policy](\Media\td-policiesransomware-governance.png "Ransomware policies")

---

## Suspicious application consent

[:arrow_up: Top](#cloud-app-security-threat-detection-lab)

Many third-party productivity apps that might be installed by business users in your organization request permission to access user information and data and sign in on behalf of the user in other cloud apps, such as Office 365, G Suite and Salesforce. 
When users install these apps, they often click accept without closely reviewing the details in the prompt, including granting permissions to the app. This problem is compounded by the fact that IT may not have enough insight to weigh the security risk of an application against the productivity benefit that it provides.
Because accepting third-party app permissions is a potential security risk to your organization, monitoring the app permissions your users grant gives you the necessary visibility and control to protect your users and your applications. The Microsoft Cloud App Security app permissions enable you to see which user-installed applications have access to Office 365 data, G Suite data and Salesforce data, what permissions the apps have, and which users granted these apps access to their Office 365, G Suite and Salesforce accounts. 

Here is an example of such user consent:

!IMAGE[App consent](\Media\td-appconsent.png "App consent")

### Investigate

1. [] Without even creating policies, Cloud App Security shows you the applications that received permissions from your users:

   !IMAGE[App permissions](\Media\td-oauth.png "App permissions")

2. [] From this page, you can easily see who granted permissions to those apps, if they are commonly used or their permissions level:

   !IMAGE[App commodity](\Media\td-zapiercommodity.png "App commodity")

3. [] If you detect that an application should not be granted access to your environment, you can revoke the app access.
   > **IMPORTANT:** This operation will apply to the **entire** organization:

   !IMAGE[App revoke](\Media\td-apprevoke.png "App revoke")

4. [] When investigating, you can search for apps rarely used in Office 365 which were granted high privileges and create a **policy** to be automatically alerted when such action is performed:

   !IMAGE[App filter](\Media\td-appfilter.png "App filter")

5. [] After clicking on the “New policy from search” button, you can see that your filter will be used to create a new policy:

   !IMAGE[App policy](\Media\td-apppolicy.png "App policy")

6. [] Go down on that page and review the possible alerts and governance automatic actions that you can configure:

   !IMAGE[App policy](\Media\td-apppolicy-governance.png "App policy")

7. [] To go further in your investigation, let’s now pivot to the “Activity log”:

   !IMAGE[Activity log](\Media\td-activitylog.png "Activity log")

8. [] In the activity log, search for "**Consent to application**" activities:

   !IMAGE[Activity log](\Media\td-activitylog-consent01.png "Activity log")

9. [] You will then be able to investigate on who, when and from where your users granted access to applications:

   !IMAGE[Activity log](\Media\td-activitylog-consent02.png "Activity log")

---

## Create your own policies

[:arrow_up: Top](#cloud-app-security-threat-detection-lab)

Now that we reviewed some of the default detection capabilities of Cloud App Security, you should start creating your own policies.
Cloud App Security provides by default many has policies templates to start creating your custom policies.

1. [] To create your policies, go to “Policies”:

   !IMAGE[Policies](\Media\td-policies.png "Policies")

2. [] Click on “Create policy” and select the type of policy you want to create:

   !IMAGE[Policies types](\Media\td-policiestypes.png "Policies types")

3. [] In the policy screen, choose the policy template you want to use:

   !IMAGE[Policies templates](\Media\td-policiestemplates.png "Policies templates")

4. [] Apply the template:

   !IMAGE[Apply template](\Media\td-applytemplate.png "Apply template")

5. [] Cloud App Security will then populate the different properties of the policy:

   !IMAGE[Policy template filter](\Media\td-policytemplatefilter.png "Policy template filter")

6. [] Review those properties and customize them if needed.

7. [] Explore other types of policies and review the proposed templates.

===

# Management with PowerShell
[:arrow_left: MCAS Home](#microsoft-cloud-app-security)) :clock10: 15 min

To help administrators interact with MCAS in a programmatic way, two
Microsoft employees created a non-official PowerShell module for Cloud
App Security. For this lab, you will install this module and discover
the available cmdlets.

Note: the module relies on the Cloud App Security API. You can find its
documentation in the MCAS portal.

!IMAGE[f847xhzx.jpg](\Media\f847xhzx.jpg)

The module is available in the PowerShell gallery and can be installed
using the ```*Install-Module mcas*``` command.

!IMAGE[6j16dgs2.jpg](\Media\6j16dgs2.jpg)

More information on the module is available on GitHub:
[https://github.com/powershellshock/MCAS-Powershell](https://github.com/powershellshock/MCAS-Powershell)

After installing the module, read how to connect to MCAS in the
PowerShell help and start exploring the cmdlets.

Hint: you'll have to create an API token in Cloud App Security.

!IMAGE[0x2tzeqd.jpg](\Media\0x2tzeqd.jpg)

Using PowerShell:

1.  [] Review the list of MCAS administrators and when they were granted
    those permissions

2.  [] Review your security alerts and close them in bulk

3.   [] Download a sample SQUID log and send it to MCAS as a snapshot
    report.

4.   [] In the portal, in Discovery, tag some apps as unsanctioned and
    generate a blocking script for your appliance to block access to
    those apps.

5.   [] You are asked to define corporate IP's in MCAS. Subnets go from
    10.50.50.0/24 to 10.50.80.0/24

===

# Log Collector Troubleshooting
[:arrow_left: Home](#labs) :clock10: 15 min

In this task, you will review possible troubleshooting steps to identify issues in automatic logs upload from the log collector.
There are several things to test at different locations: in the log collector, in MCAS, at the network level.

## Useful commands

* `cd` : *Used to navigate in the directories*
    >**Examples:**
    >
    >`cd /var/adallom` : *to go to the specified directory*
    >
    >`cd /` : *to go to the root directory*
    >
    >`cd ..` : *to go to the parent directory*

* `more` or `cat` : *Used to display the content of the logs*
    > **Examples:**
    >
    >`more trace.log` : *to display the content of the trace.log file*

* `tail` : *Used for outputting the last part of files*

* `ll` : *Used to display the content of the directory as a list*. This command is an alias for `ls -l`

* `clear` : *Used to clear the screen*

* `tab key` : Used to perform autocompletion

## Verify the log collector (container) status

1. [] On **Client01**, open a session on PuTTY to **192.168.141.125** and use the credentials below.
    In the PuTTY Configuration window, enter **192.168.141.125** and click **Open**.

    ^IMAGE[Open Screenshot](\Media\dis-puttyconfig.png "Putty config")

    Log in using the credentials below.
    >|Username|Password|
    >|---|---|
    >|user01|Passw0rd1|
    >
    >:warning:The password doesn't appear in the command prompt, you can safely press enter to validate the credentials.

2. [] Run the following commands:

    ``` 
    sudo -i
    docker stats
    ```
    !IMAGE[Docker stats](\Media\dis-dockerstats.png "Docker stats")

     > [!HINT] This command will show you the status of the log collector instance.

3. [] Press `Ctrl-C` to end the command.

4. [] Next, run the command below:

    ``` 
    docker logs --details LogCollector
    ```
    !IMAGE[Docker log](\Media\dis-dockerlog.png "Docker log")

     > [!HINT] This command will show you the container logs to verify if it encountered errors when initiating.

### Verify the log collector logs

1. [] Type the following command:

    ``` 
    docker exec -it LogCollector bash
    ```
     > [!HINT] This command will execute the container's bash. You will then be able to execute commands *from inside* of the log collector.

2. [] You can now explore the container filesystem and inspect the **/var/adallom** directory. This directory is where you will investigate most of the issues with the syslog or ftp logs being sent to the log collector.

    ``` 
    cd /var/adallom
    ll
    ```

    !IMAGE[adallom folder](\Media\dis-dockerll.png "adallom folder")

    Go to the following folders and review their log files using `more`:
    * **/adallom/ftp/discovery**: this folder contains the data source folders where you send the log files for automated upload. This is also the default folder when logging into the collector with FTP credentials.
    * **/adallom/syslog/discovery**: if you setup the log collector to receive syslog messages, this is where the flat file of aggregated messages will reside until it is uploaded.
    * **/adallom/discoverylogsbackup**: this folder contains the last file that was sent to MCAS. This is useful for looking at the raw log in case there are parsing issues.

3. [] To validate that logs are correctly received from the network appliance, you can also verify the **/var/log/pure-ftpd** directory and check the transfer log:

    ``` 
    tail transfer.log
    ```

    !IMAGE[FTP logs](\Media\dis-pureftp.png "FTP logs")

4. [] Now, move to the **/var/log/adallom** directory.

    !IMAGE[var log](\Media\dis-varlog.png "var log")

    Go to the following folders and review their content and log files using `ll` and `more` or `tail`:
    * **/var/log/adallom/columbus**: this folder is where you will find log files useful for troubleshooting issues with the collector sending files to Cloud App Security. In the **log-archive** folder you can find previous logs compressed as *.tar.gz* files that could be used to send to support for example.
    * **/var/log/adallom/columbusInstaller**: this is where you will investigate issues with the log collector itself. You will find here logs related to the configuration and bootstrapping of the collector. For example, **trace.log** will show you the bootstrapping process:

    !IMAGE[Bootstrapping log](\Media\dis-bootstrapping.png "bootstrapping log")

## Verify the connectivity between the log collector and Cloud App Security

An easy way to test the connectivity after configuring the log collector is to download a sample of your appliance logs from and use WinSCP to connect to the log collector to upload it and see if it gets uploaded to Cloud App Security, as you did in the previous exercise

!IMAGE[Pending log](\Media\dis-pending.png "Log pending")

> [!HINT] If the log stays in the source folder for too long, then you know you probably have a connection issue between the log collector and Cloud App Security and should go investigate the logs reviewed previously.

===

# Conditional Access App Control with 3rd party apps
[:arrow_left: MCAS Home](#microsoft-cloud-app-security) :clock10: 45 min

## Introduction

Conditional Access App Control utilizes a reverse proxy architecture and is uniquely integrated with Azure AD conditional access. Azure AD conditional access allows you to enforce access controls on your organization’s apps based on certain conditions. The conditions define who (for example a user, or group of users) and what (which cloud apps) and where (which locations and networks) a conditional access policy is applied to. After you’ve determined the conditions, you can route users to the Microsoft Cloud App Security where you can protect data with Conditional Access App Control by applying access and session controls.

Conditional Access App Control enables user app access and sessions to be monitored and controlled in real time based on access and session policies. Access and session policies are utilized within the Cloud App Security portal to further refine filters and set actions to be taken on a user.

With the access and session policies, you can:

* **Block on download**: You can block the download of sensitive documents. For example, on unmanaged devices.
* **Protect on download**: Instead of blocking the download of sensitive documents, you can require documents to be protected via encryption on download. This ensures that the document is protected, and user access is authenticated, if the data is downloaded to an untrusted device.
* **Monitor low-trust user sessions**: Risky users are monitored when they sign into apps and their actions are logged from within the session. You can investigate and analyze user behavior to understand where, and under what conditions, session policies should be applied in the future.
* **Block access**: You can completely block access to specific apps for users coming from unmanaged devices or from non-corporate networks.
* **Create read-only mode**: By monitoring and blocking custom in-app activities you can create a read-only mode to specific apps for specific users.
* **Restrict user sessions from non-corporate networks**: Users accessing a protected app from a location that is not part of your corporate network, are allowed restricted access and the download of sensitive materials is blocked or protected.

>:memo: In this lab, we will cover only some scenarios.

===

## Federate Salesforce with Azure AD

[:arrow_up: Top](#conditional-Access-App-Control)

>:warning: As Conditional Access App Control requires the protected app to be federated with your IdP (Azure AD in our case), we will first federate Saleforce with our tenant before moving to the controls configuration. Please go through all the steps exactly as described to avoid any complications further in the lab.

1. [] Create a Salesforce developer account.

    * On Client01, launch a browser and create a Salesforce developer org at this address: [https://developer.salesforce.com/signup](https://developer.salesforce.com/signup).
      This org (or tenant) will allow you to create a test environment to federate with our Azure AD tenant.

      >:memo: Dev Salesforce orgs are available for free but are deleted after extended periods of inactivity.
      >
      > :warning: Use your lab tenant admin user as the Email and Username

      !IMAGE[Dev sign-up](\Media\appc-signup.png "Salesforce sign-up")

    * Fill in the rest of details, click **Sign me up**, accept the **verification email** in your mailBox, and choose a new password. Use the admin password provided in the lab environment if possible.

        !IMAGE[Dev sign-up](\Media\appc-signup2.png "Salesforce sign-up")

        !IMAGE[Dev sign-up](\Media\appc-signup3.png "Salesforce sign-up")

2. [] Configure Salesforce in Azure AD for single sign-on.

    * In Salesforce, go to **Setup**, search for **My Domain** and register a new domain matching your Office 365 lab domain, e.g., **ems123456-dev-ed.salesforce.com**

        !IMAGE[My domain](\Media\sf-mydomain.png "My domain")

        !IMAGE[My domain](\Media\sf-registerdomain.png "My domain")

        !IMAGE[My domain](\Media\sf-registerdomain2.png "My domain")

    * Save **full Salesforce domain name**, including **https://** for the next step, e.g., **https://ems123456-dev-ed.salesforce.com**

        >:warning: Do not close this page !

        !IMAGE[My domain](\Media\sf-registerdomain3.png "My domain")

    * Go to **https://portal.azure.com** were we will add Salesforce as an Enterprise application and configure **single sign-on**, which is a requirement for using App Control.

3. [] Go to **Azure Active Directory**, click on **Enterprise applications**, and add the **Salesforce** application. Call it **SalesforceCAS**, and click on **Add**.

    !IMAGE[Add SF](\Media\appc-app1.png "Add SF")

    !IMAGE[Add SF](\Media\appc-app2.png "Add SF")

    !IMAGE[Add SF](\Media\appc-app3.png "Add SF")

    !IMAGE[Add SF](\Media\appc-app4.png "Add SF")

4. [] Now that Salesforce has been added as an Enterprise application, we have to configure **single sign-on**.

    !IMAGE[Add SF](\Media\appc-app5.png "Add SF")

5. [] Select **SAML** as the SSO method.

    !IMAGE[Add SF](\Media\appc-app6.png "Add SF")

6. [] We will now configure the SAML single sign-on using the information provided by Salesforce when we added our domain.

    >:warning: Do not forget to add **https://** in front of the domain name provided by Salesforce.

    !IMAGE[Add SF](\Media\appc-app7.png "Add SF")

    !IMAGE[Add SF](\Media\appc-app8.png "Add SF")

7. [] Close the pane and go to the **Step 4** of the SSO wizard and click on **View step-by-step instructions**. This page will give you all the required information for configuring Salesforce SSO.

    !IMAGE[Add SF](\Media\appc-app9.png "Add SF")

    !IMAGE[Add SF](\Media\appc-app10.png "Add SF")

8. [] Go back to the **Salesforce admin page** and go to the **Signle sign-on** settings. There, click on the **Edit** button.

    !IMAGE[Add SF](\Media\appc-app11.png "Add SF")

9. [] Enable single sign-on using SAML and click on the **Save** button.

    !IMAGE[Add SF](\Media\appc-app12.png "Add SF")

10. [] Back on the configuration page, under **SAML Single Sign-On Settings**, click on **New**.

    !IMAGE[Add SF](\Media\appc-app13.png "Add SF")

    We will now configure those settings using the information provided in the **Azure AD portal**.

    !IMAGE[Add SF](\Media\appc-app14.png "Add SF")

11. [] For the configuration, you will have to use the information provided at the bottom of the Azure AD **Configure sign-on** pane. Scroll down until you reach the **Quick reference** section.

    !IMAGE[Add SF](\Media\appc-app15.png "Add SF")

12. [] Copy/paste the information from the **Quick reference** to the Salesforce **single sign-on settings** page.

    a.In the Name textBox, type the name of the configuration: **AzureAD**.

    b. Paste **Azure AD SAML Entity ID** value into the **Issuer** textBox.

    c. In the **Entity Id** textBox, type in the **Sign On URL** that you entered in **Step 1**, which should be in this format: **http://company.my.salesforce.com**

    d. Download the **Azure AD Signing Certificate** in the Azure portal and then click **Browse** to upload the downloaded certificate Azure AD Signing Certificate in the **Salesforce setting page**.

    e. As **SAML Identity Type**, select **Assertion contains the Federation ID** from the User object.

    f. As **SAML Identity Location**, select **Identity is in the NameIdentifier element of the Subject statement**.

    g. Paste **Azure AD Single Sign-On Service URL** into the **Identity Provider Login URL** textBox.

    h. Salesforce does not support SAML logout. As a workaround, paste **Azure AD Sign Out URL** into the **Identity Provider Logout URL** textBox.

    i. As **Service Provider Initiated Request Binding**, select **HTTP POST**.

    j. Click Save.

    !IMAGE[Add SF](\Media\appc-app16.png "Add SF")

    !IMAGE[Add SF](\Media\appc-app17.png "Add SF")

13. [] Go back to **My Domain** in Salesforce and in **Authentication Configuration**, click on **Edit**. You will be redirected to another page.

    !IMAGE[Add SF](\Media\appc-app18.png "Add SF")

14. [] In **Authentication Configuration**, un-check the **Login Page** checkBox and check the **Azure AD** checkBox. Click on Save. When back on the configuration page, click on the **login** button to complete the configuration.

    !IMAGE[Add SF](\Media\appc-app19.png "Add SF")

    !IMAGE[Add SF](\Media\appc-app20.png "Add SF")

    !IMAGE[Add SF](\Media\appc-app20a.png "Add SF")

    !IMAGE[Add SF](\Media\appc-app20b.png "Add SF")

---

## Deploy Salesforce to your users

[:arrow_up: Top](#conditional-Access-App-Control)

We will now provide access to our users and validate the SSO experience.

1. [] Go back to the Azure AD portal, within the **SalesforceCAS** app and choose **Users and groups**

    !IMAGE[Assign users](\Media\appc-app21.png "Assign users")

2. [] Click on **+ Add user**. Choose your admin account as the user (e.g.,admin@ems123456.onmicrosoft.com) and select **System Administrator** as the Role. Click on **Assign**

    !IMAGE[Assign users](\Media\appc-app22.png "Assign users")

    !IMAGE[Assign users](\Media\appc-app23.png "Assign users")

    !IMAGE[Assign users](\Media\appc-app24.png "Assign users")

    > [!ALERT] If you want to assign Salesforce to more users, you must create a user for them in Salesforce as we didn't configured **provisionning**. Our admin account already has an an account matching his UPN, created during the Salesforce configuration.

3. [] Test the setup by going to [https://myapps.microsoft.com](https://myapps.microsoft.com) with your admin account and click on the **SalesforceCAS** app. You should then experience SSO to Salesforce.

    !IMAGE[Test SSO](\Media\appc-app25.png "Test SSO")

    !IMAGE[Test SSO](\Media\appc-app26.png "Test SSO")

    :warning: If you receive an error message, verify that you validated the SSO configuration by clicking on the **Log in** button in **step 14**.

===

## Deploy the reverse proxy capability for Salesforce

[:arrow_up: Top](#conditional-Access-App-Control)

The next step of the configuration is to create a Conditional access policy in Azure AD and then complete the configuration in Cloud App Security.

> [!HINT] Soon, you will be able to perform the full configuration from the Azure AD conditional access policy configuration pane.

1. [] In Azure Active Directory, under **Security**, click on **Conditional access**.

    !IMAGE[Configure policy](\Media\appc-policy1.png "Configure policy")

2. [] Click on **New policy** and create a new policy.

    !IMAGE[Configure policy](\Media\appc-policy2.png "Configure policy")

3. [] Use the following settings in your policy.

    **Name**: Test Cloud App Security proxy
    **Assignment**: choose your admin
    **Cloud apps**: SalesforceCAS
    **Access control / Session**: Use Conditional Access App Control **checkBox**

    !IMAGE[Configure policy](\Media\appc-policy3.png "Configure policy")

    !IMAGE[Configure policy](\Media\appc-policy4.png "Configure policy")

    !IMAGE[Configure policy](\Media\appc-policy5.png "Configure policy")

4. [] Enable your policy and click on **Create**

    !IMAGE[Configure policy](\Media\appc-policy6.png "Configure policy")

    !IMAGE[Configure policy](\Media\appc-policy7.png "Configure policy")

5. [] After the policy was created successfully, open a new browser, ***make sure you are logged out***, go to ```https://myapps.microsoft.com```, connect with the admin user and click on the SalesforceCAS tile.

6. [] Go back to the Cloud App Security portal, and under the settings cog choose **Conditional Access App Control**.

    !IMAGE[Configure policy](\Media\appc-policy8.png "Configure policy")

    You should see know that Salesforce has been discovered and need to continue the setup.

    !IMAGE[Configure policy](\Media\appc-policy9.png "Configure policy")

    > [!HINT] If the message does not appear, go back to step 5. (After the policy was created...) this time, close the browser and open a new browser in Incognito mode.

    In the dialog that opens, click on **Add**.

    !IMAGE[Configure policy](\Media\appc-policy10.png "Configure policy")

    !IMAGE[Configure policy](\Media\appc-policy11.png "Configure policy")

    The deployment is now **complete**!

---

### Configure device authentication

[:arrow_up: Top](#conditional-Access-App-Control)

Conditional Access App Control is capable to identify company devices using either Azure AD, Intune or certificates (provided by 3rd party MDM for example). We will here simulate the 3rd party MDM scenario, using client certificates.

1. [] Go to the settings and at the bottom of the page, choose **Device identification**.

    !IMAGE[Device authentication](\Media\appc-device1.png "Device authentication")

2. [] Click on the **Browse** button and upload the **CASTestCA.crt** certificate from the **Client Certificate** folder within the **Demofiles.zip** file you've received as the certificate authority root certificate.

    !IMAGE[Device authentication](\Media\appc-device2.png "Device authentication")

    !IMAGE[Device authentication](\Media\appc-device3.png "Device authentication")

    !IMAGE[Device authentication](\Media\appc-device4.png "Device authentication")

---

### Create a session policy

[:arrow_up: Top](#conditional-Access-App-Control)

To control our users sessions to Salesforce, we have now to create a **policy**.

1. [] In the Cloud App Security portal, select **Control** followed by **Policies**.

    !IMAGE[Session policy](\Media\appc-session1.png "Session policy")

2. [] In the **Policies** page, click **Create policy** and select **Session policy**.

    !IMAGE[Session policy](\Media\appc-session2.png "Session policy")

3. [] In the **Session policy** window, assign a name for your policy, such as **Block download of sensitive documents to unmanaged devices** and in the **Session control type** field, select **Control file download (with DLP)**

     !IMAGE[Session policy](\Media\appc-session3.png "Session policy")

4. [] Under **Activity source** in the **Activities matching all of the following** section, select the following activity filters to apply to the policy:

    **Device tags** does not equal **Valid client certificate**
    **App** equals **Salesforce**
    Check the **Enabled** checkBox near **Content inspection**

    !IMAGE[Session policy](\Media\appc-session4.png "Session policy")

5. [] Check the **Include files that match a preset expression** radio button. In the dropdown menu just below the radio button, scroll all the way to the end to choose **US: PII: Social security number** and check the **Don't require relevant context** checkBox, just below the dropdown menu.

    !IMAGE[Session policy](\Media\appc-session5.png "Session policy")

6. [] Under **Actions**, select **Block**. Check the **Customize block message** checkBox, and add a custom message in the textBox that has opened, e.g.: "This file is sensitive"

    !IMAGE[Session policy](\Media\appc-session6.png "Session policy")

7. [] Click on **Create**

8. [] Create a second **Session policy** called **Protect download to unmanaged devices**. In the **Session control type** field Select **Control file download (with DLP)**.

    !IMAGE[Session policy](\Media\appc-session7.png "Session policy")

9. [] Under **Activity source** in the **Activities matching all of the following** section, select the following activity filters to apply to the policy:

    **Device tags** does not equal **Valid client certificate**
    **App** equals **Salesforce**

    !IMAGE[Session policy](\Media\appc-session8.png "Session policy")

10. [] Check the **Enabled** checkBox near **Content inspection**. Under **Actions**, select **Protect**

    !IMAGE[Session policy](\Media\appc-session9.png "Session policy")

11. [] Click on **Create**

12. [] Disable this policy

---

### Test the user experience

[:arrow_up: Top](#conditional-access-app-control)

1. [] Extract the file **silvia.pfx** from the **Client Certificate** folder in **Demo files.zip** file you've received

2. [] Double click on the **silvia.pfx** file, click **Next**, **Next**, enter the password **acme**, click **Next**, **Next**, **Finish**.

3. [] Open a new browser in an Incognito mode

4. [] Go to [https://myapps.microsoft.com](https://myapps.microsoft.com) and login with the admin user.

5. [] Click on the **SalesforceCAS** icon.

6. [] You should now see a certificate prompt. Click on **Cancel**.

    > [!HINT] This will simulate a connection from an unmannaged device. **In a real demo**, you can open two different browsers, side by side, and show the user experience from a managed and unmanaged device by clicking on **OK** in one browser and **Cancel** in the other.

   !IMAGE[Session policy](\Media\appc-session10.png "Session policy")

7. [] You should then see a Monitored access message, click on **Continue to Salesforce** to continue.

    !IMAGE[Session policy](\Media\appc-session11.png "Session policy")

8. [] Now you are logged in to Salesforce. Click on + and go to Files

    !IMAGE[d0ik67yl.jpg](\Media\d0ik67yl.jpg)

9. [] Upload the files **Personal employees information.docx** and **Protect with Microsoft Cloud App Security proxy.pdf** from the **Demo files.zip** file to the Files page in Salesforce

10. [] Download the **Protect with Microsoft Cloud App Security proxy.pdf** files and see that it is downloaded, and you can open it.

11. [] Download the **Personal employees information.docx** file and see that you get a blocking message and instead of the file, you get a **Blocked...txt** file.

   !IMAGE[wvk16zl2.jpg](\Media\wvk16zl2.jpg)

---

### Test the admin experience

[:arrow_up: Top](#conditional-access-app-control)

1. [] Go back to the Cloud App Security portal, and under **Investigate**  choose **Activity log**

2. [] See the login activity that was redirected to the session control, the file download that was not blocked, and the file download that was blocked because it matched the policy.

    !IMAGE[j0vuo06k.jpg](\Media\j0vuo06k.jpg)


=== 

## Azure ATP Immersion Lab 
[:arrow_left: Home](#introduction)

The rest of the lab will be instructor led via PowerPoint. 
