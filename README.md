# jamfbot - Querying JAMF Pro via a Slackbot

## Overview
This page will explain how to configure a Jamf bot for slack to query your JAMF Pro server.


## Requirements
* Administrator permissions on a Jamf Pro server
* Administrator permissions for a Slack instance
* AWS Access (can be an Administrator or have permissions to these services:
	- Lambda
	- IAM Policy
	- SNS Topics
	- AWS Secret Manager


## Table of Contents
* Slack App Setup
* Create Jamf Pro API User
* Setup AWS Secret Manager in AWS
* Setup AWS SNS Topics
* AWS Lambda Function Setup

## Slack App Setup
This step will have us setup out Slackbot/Application that will communicate with AWS Lambda for our Jamf Searches.

1. https://api.slack.com/apps
2. Click Create new App
3. On the next popup select the workspace and name your app.
4. Once created, select Slash Commands from the left column, under Features.
5. Click create new command and fill out the following information.  When done, hit Save.
	- Command: `/jamf`
	- request URL: We will fill this in later, but this is where a request gets sent when we run the command.  In this case, a AWS Lambda function.
	- Short Description: query Jamf Pro
6. Under the Settings category on the left, select Basic Information.  On the page that opens, copy the Signing Secret value for use later in Secret Manager.
7. We will add an interactivity button that sends another request to AWS Lambda, but we have not set that up yet, so we will return here later.
8. Lastly create a channel for the Bot to use in Slack (it can be private or public).  Visit the channel in a web-browswer and grab the channel ID from the URL.  The channel ID is the last set of characters at the end of the URL.  Save this for later.

## Create Jamf Pro API User
The first step we need to take is to create a user for the Slackbot to query Jamf Pro with.  

1. Go to https://JSSURL/accounts.html
2. Create a new Standard local user account with the following permissions:
	* Read ONLY Access to Computers, Computer Extesnion Attributes, Users, User Extension Attributes, Advanced Computer Searches
3. Save the username and password information for use later

## Setup Secret Manager in AWS
The Lambda functions require certain API keys and signing secrets to communicate properly with Slack and Jamf Pro.  We will use AWS Secret Manager to store these secrets properly so that the functions can use them securely.

1. Login into the AWS Console using your administrator account.
2. Go to Secret Manager
3. Create 2 Secrets by following these steps
	- click Store a new secret
	- for Secret Type, select "Other type of secrets"
	- We are going to add 3 keys/values to this secret
		* `username`:Jamf Pro username you created above
		* `password`:Jamf Pro password you created above
		* `secret`:Slack Signing secret you saved when you created the Slack App.
	- Hit Next when you are done
	- Enter secret name : `jamf_slack_bot` and add any tags for easy identification.
	- Hit Next again and make sure automatic rotation is disabled.
	- Finally review your settings.  At the bottom, there is sample code, copy the keys `secretName` and `region` and save them for later.  Hit ``Store`` at the bottom when ready.
4. In Secrets manager, click into the Secret you just created and copy the value for `Secret Arn` for later use.

## Setup AWS SNS Topics
Once our endpoints recieve data, they need to know where to send that data.  That's where AWS' Simple Notification service comes in handy.  We will setup 2 topics.

1. Go to Amazon SNS.  Once there, select `Topics` from the left column and then `Create topic`.
2. Enter the name: `slackJamfInitialTopic` and `slackJamfIdSearch`.
3. You can leave all settings default.  Optionally you can add tags for easy identification.
4. After you create each topic, make sure you copy the `ARN` from each and save them for later use.

## AWS Lambda Functions Setup
Now we get to create our functions that are going to be doing all the work for the Slackbot.

**Go to the AWS Lambda dashboard to begin**

### Function #1
1. click Create Function
2. Select Author from scratch
3. name the function `slackApp-01-initialJamfCommand`
4. select `Python 3.7` as the runtime.
5. Expand the `Choose or create execution role` section and select `New role with basic Lambda permissions`
6. Once the function has been created, go to IAM and find the role that was just created.  Click the role.
7. Under permissions, select `attach policy`.
8. Click `Create Policy`.  
9. In the new window that opens up, select the `JSON` tab.  Paste in the below JSON
**You need to input the ARN of the secret you created above into the policy**

```
{
	"Version": "2012-10-17",
	"Statement": {
		"Effect": "Allow",
		"Action": "secretsmanager:GetSecretValue",
		"Resource": "<arn-of-the-secret-the-app-needs-to-access>"
	}
}
```

10. click `Review Policy` and give it a name and description.  Then click `Create policy`.
11. Go back to the Attach policy tab and hit the reload button in the top right and search for the policy you just created.  Check the box to left and hit `Attach policy`.
12. Go back to Lambda and to the function you just created.
13. Paste in the [code for the first function](https://gist.githubusercontent.com/taniacomputer/ef9d87ec5f0895058a1dfbd5f4d185bb/raw/296d05878275b689ace823ea814aaddd460d8e11/slackapp_initial_slash_command_invoked.py) into the space provided in the `Function Code` section.
14. Once pasted in hit `Save`.
15. We are now going to edit lines 43, 45, 46, 48.
	- Line 43 is is the SNS Topic ARN for `slackJamfInitialTopic` topic.
	- Line 45 is the Secret Manager ARN.  It will look like this initially: `"https://secretsmanager.xx-yyyy-z.amazonaws.com"`.  Please replace xx-yyyy-z with your region.  For example, `us-east-2`.
	- Line 46 is the region, which would be same as the previous step.
	- Line 48 is the Slack Channel ID from the Slackbot creation section.
16. Once edited, hit `Save`.
17. Scroll up to the `Designer` section of Lambda.  Click `Add Trigger`.  From the drop down select `API Gateway`.  You will see more options now.
	- Choose to create an API
	- set `Security` as open
	- `API Type`: REST API
	Leave all other settings defaults and hit save.
18. Click `Add destination` and choose `SNS topic` as a `Destinition Type`.  Configure the following options:
	- choose `slackJamfInitialTopic` as the destintion topic
	- set `Source` as `Asychronous invocation`
	- set `Condition` as `Success`
	- hit `Save`

<a href="Prestage Package Configuration"><img src="https://github.com/zacharysfisher/connect-login-notify/blob/master/images/package_prestage.png" align="right" height="350" width="450" ></a>
First we need to build our Pre-Stage package to look like the image to the right:
<br>
<br> 
As you can see, we are installing both Sync and Login to a temporary location which we will then all to install using the `installer` binary later in the process.  We are also installing out image files and the notify script location which will also be called later on in this provisioning process.Post Install Script should look like below:
The package also needs a post-install script that will install Login, Sync and activate the Notify and RunScript Mechanisms for us.  Please see below.
<br>
<br>
<br>


```
#!/bin/sh
## postinstall

# Install JCL + Sync
installer -pkg /tmp/Jamf\ Connect\ Login-1.9.0.pkg -target / 
installer -pkg /tmp/Jamf\ Connect\ Sync-1.1.0.pkg -target /

# Enable Notify - Run Script
/usr/local/bin/authchanger -reset -Okta -postAuth JamfConnectLogin:Notify JamfConnectLogin:RunScript,privileged


exit 0		## Success
exit 1		## Failure
```

Once this package is ready for building, make sure that you `sign` the package and upload it to your distribution points for deployment.

## Plist Configuration for Jamf Connect Login
Below is an example Plist that we can use with a Custom Settings payload Configuration Profile:

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>AuthServer</key>
	<string>acme.okta.com</string>
	<key>HelpURL</key>
	<string>https://acme.okta.com</string>
	<key>LocalFallback</key>
	<true/>
	<key>LoginLogo</key>
	<string>/usr/local/images/company_logo.png</string>
	<key>OIDCIgnoreCookies</key>
	<true/>
	<key>OIDCRedirectURI</key>
	<string>https://127.0.0.1/jamfconnect</string>
	<key>AllowNetworkSelection</key>
	<true/>
	<key>CreateSyncPasswords</key>
	<true/>
	<key>ScriptPath</key>
	<string>/usr/local/bin/enrollment_script.sh</string>
	<key>EnableFDE</key>
	<true/>
</dict>
</plist>
```

Below is an explanation of keys being used

| Key                    | Description                                                            | Example         |
|------------------------|------------------------------------------------------------------------|-----------------|
| AuthServer  | Your Okta base URL for your organization | `<key>AuthServer</key>` `<string>acme.okta.com</string>` |
| HelpURL     | URL users are directed to when hitting help at login.  This can be a custom page or your okta homepage    | `<key>HelpURL</key>` `<string>https://acme.okta.com</string>` |
| LocalFallback       | Allows local authentication in case Okta is not reachable    | `<key>LocalFallback</key>` `<true/>` |
| LoginLogo     | Login Logo to be displayed at Jamf Connect Login screen | `<key>LoginLogo</key>` `<string>/path/to/image.png</string>` |
| OIDCIgnoreCookies     | Ignores any cookies stored by the loginwindow | `<key>OIDCIgnoreCookies</key>` `<true/>` |
| OIDCRedirectURI     | The redirect URI used by your Jamf Connect app in Okta.  "jamfconnect://127.0.0.1/jamfconnect" is recommended by default. | `<key>OIDCRedirectURI</key>` `<string>https://127.0.0.1/jamfconnect</string>` |
| AllowNetworkSelection     | Allows user to select Wi-Fi network at login window. | `<key>AllowNetworkSelection</key>` `<true/>` |
| CreateSyncPasswords     | Creates a keychain entry for Jamf Connect Sync (requires Sync to be installed already at time of login for this to function | `<key>CreateSyncPasswords</key>` `<true/>` |
| ScriptPath     | Specifies the path to the script or other executable run by the RunScript mechanism. | `<key>ScriptPath</key>` `<string>/path/to/script.sh</string>` |
| EnableFDE     | Enables Filevault and stores the FV Recovery key locally for Escrow to JAMF Pro (Requires Escrow Configuration Profile to send to JAMF Pro). | `<key>EnableFDE</key>` `<true/>` |

**Please note**
For 10.15 and on, a PCCC Configuration profile is needed to enable FileVault on login.  ![PCCC Payload to allow EnableFDE](https://github.com/zacharysfisher/connect-login-notify/blob/master/images/PCCC_FDE.png)

## RunScript Configuration
JAMF has good instrutions on how to enable the RunScript mechanism for JAMF Login.  [RunScript Mechanism Documentation](https://www.jamf.com/resources/product-documentation/jamf-connect-administrators-guide/)

You can also follow these instructions using the nano editor.
1. We have actually already enabled our workflow to enable this mechanism by using the `authchanger` command and to include `JamfConnectLogin:RunScript,privileged` in our postInstall script.
2. In this script we will tell Notify what to display and what JAMF Policies to run.  See the script in this repo for an example script that is modelled after information that JAMF provides.  I have also attached a script that renames computers based on a users first name and last name from their Okta Profile.



This script displays different images, display text and runs the on-boarding JAMF Policies.  A better explaination of commands that can be run can be found here on JAMF's documentation page. [Notify Screen Mechanism](https://www.jamf.com/resources/product-documentation/jamf-connect-administrators-guide/)


## Prestage Settings
When creating a Prestage Enrollment to work with there are a few settings and configuration profiles that need to be enabled so that Jamf Connect Login gets configured properly.

1. Configuration Profiles (PPPC for Filevault and Jamf Connect Login Settings) should be scoped to these new computers in the `Configuration Profiles` section of JAMF Pro as well as scoped to the Prestage Enrollment. <a href="Configuration Profiles in Prestage Enrollment"><img src="https://github.com/zacharysfisher/connect-login-notify/blob/master/images/Config_Prestage_profiles.png" height="350" width="250" ></a>
2. Attach your Prestage package to the prestage enrollment.
3. For Account Settings, in Prestage Enrollment.  Make you select to create an Admin Account.  Here you can hoose tho keep the account hidden as well as skipping user creaiton, which is something we want Jamf Connect Login to handle.  ![Account Settings](https://github.com/zacharysfisher/connect-login-notify/blob/master/images/prestage_account_settings.png)
4. The last step is to setup the General Settings tab with information specific to your deployment and select which Setup Assistant options you want to display to the user.

Once done, scope this enrollment to your devices.

## Okta Configurations for Standard / Admin Users
If you are using the Plist linked above, no additionaly configuration is needed to allow users to log into computers using their Okta accounts.  However if you wish to segregate users and allow certain users to become Admins and others Standard, you will need to add some keys to your Plist and take some additional configuration steps in Okta.  Below are the keys that need to be added to your plist:

| Key                    | Description                                                            | Example         |
|------------------------|------------------------------------------------------------------------|-----------------|
| OIDCAdminClientID  | OIDC ClientID for Okta Application that makes the user an admin user upon logging in. | `<key>OIDCAdminClientID</key>` `<string>0oa3qmcgyywWj1JR52p7</string>` |
| OIDCAccessClientID  | OIDC ClientID for Okta Application that makes the user a standard user upon logging in.    | `<key>OIDCAccessClientID</key>` `<string>0oa3qmdmdqJGOB1iG2p7</string>` |


## Deployment

1) Upload your prestage package to your specified Distribution point.  As of a recent relase of JAMF Pro this no longer needs to be a Cloud Distribution point but it does require HTTPS.  In addition to this, make sure that your PKG is signed.
2) Double check that your configuration profiles are scopes to proper machines in the Configuration Profiles section of JAMF Pro and the same configuration profiles are selected for prestage enrollment deployment.  
3) If you are going to use the rename script that is in this Repo, make sure it is being called during notify.  Make sure that the Okta API key it uses is a Read Only key.
4) Double check your Prestage settings to make sure that account creation is skipped and that the Prestage is assigned to the proper devices.
