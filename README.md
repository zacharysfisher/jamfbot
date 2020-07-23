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

1. go to `https://api.slack.com/apps`
2. Click Create new App
3. On the next popup select the workspace and name your app.
4. Once created, select Slash Commands from the left column, under Features.
5. Click create new command and fill out the following information.  When done, hit Save.
	- Command: `/jamf`
	- request URL: We will fill this in later, but this is where a request gets sent when we run the command.  In this case, a AWS Lambda function. For the time being enter in `https://127.0.0.1/jamf`
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
18. Once created, select the newly created API Endpoint and hit the error to expand the Endpoint.  Copy this URL for later.
18. Click `Add destination` and choose `SNS topic` as a `Destinition Type`.  Configure the following options:
	- choose `slackJamfInitialTopic` as the destintion topic
	- set `Source` as `Asychronous invocation`
	- set `Condition` as `Success`
	- hit `Save`
	
### Function 2
1. click Create Function
2. Select Author from scratch
3. name the function `slackApp-02-RunMatchSearchJamf`
4. select `Python 3.7` as the runtime.
5. Expand the `Choose or create execution role` section and select `New role with basic Lambda permissions`
6. Once the function has been created, go to IAM and find the role that was just created.  Click the role.
7. Under permissions, select `attach policy`.
8. Search for the policy you created above for Secret permissions.  Check the box to left and hit `Attach policy`.
10. Go back to Lambda and to the function you just created.
12. Paste in the [code for the second function](https://gist.githubusercontent.com/taniacomputer/deece78ad7dd408d4d367245fcc08bc8/raw/b812530ce502baba5cc1a7f2c999e37f668ad5d3/slackapp_run_matchSearch.py) into the space provided in the `Function Code` section.
13. Once pasted in hit `Save`.
14. Edit lines 16, 19, 20, 22 with the below values
	- replace `jamf.com` with your Jamf Pro URL on line 16
	- SM_ENDPOINT_URL (replace xx-yyyy-z with your AWS Region (ie us-east-2)
	- SM_REGION_NAME (same as above, replace xx-yyyy-z with your AWS Region
	- SLACK_CHANNEL_ID is the slack channel ID we replaced in the previous function
15. Hit save once those lines are edited and click Add Trigger.
	- Select SNS
	- select `slackJamfInitialTopic` from list
	- make sure `Enabled Trigger` is checked and hit `Add`

### Third Function
1. click Create Function
2. Select Author from scratch
3. name the function `slackApp-03-JamfMoreInfo`
4. select `Python 3.7` as the runtime.
5. Expand the `Choose or create execution role` section and select `New role with basic Lambda permissions`
6. Once the function has been created, go to IAM and find the role that was just created.  Click the role.
7. Under permissions, select `attach policy`.
8. Search for the policy you created above for Secret permissions.  Check the box to left and hit `Attach policy`.
9. Go back to Lambda and to the function you just created.
10. Paste in the [code for the third function](https://gist.githubusercontent.com/taniacomputer/e6a8b651b10ff27a821be4c43e56e418/raw/55e26aee0596c5859e0c5ab03bd05e560bdf5e4c/slackapp_buttonClicked.py) into the space provided in the `Function Code` section.
11. Once pasted in hit `Save`.
12. Edit lines 19, 22, 23.
	- SNS_TOPIC_ARN is ARN from the `slackJamfIdSearch` topic that you saved earlier
	- SM_ENDPOINT_URL (replace xx-yyyy-z with your AWS Region (ie us-east-2)
	- SM_REGION_NAME (same as above, replace xx-yyyy-z with your AWS Region
13. Hit `Save` again to save changes.
14. Go to the `Designer` section of Lambda.  Click `Add Trigger`.  From the drop down select `API Gateway`.  You will see more options now.
	- Choose to create an API
	- set `Security` as open
	- `API Type`: REST API
	Leave all other settings defaults and hit save.
15. Once created, select the newly created API Endpoint and hit the error to expand the Endpoint.  Copy this URL for later.
16. Click `Add destination` and choose `SNS topic` as a `Destinition Type`.  Configure the following options:
	- choose `slackJamfIdSearch` as the destintion topic
	- set `Source` as `Asychronous invocation`
	- set `Condition` as `Success`
	- hit `Save`

### Fourth Function
1. click Create Function
2. Select Author from scratch
3. name the function `slackApp-03-MoreInfoResponse`
4. select `Python 3.7` as the runtime.
5. Expand the `Choose or create execution role` section and select `New role with basic Lambda permissions`
6. Once the function has been created, go to IAM and find the role that was just created.  Click the role.
7. Under permissions, select `attach policy`.
8. Search for the policy you created above for Secret permissions.  Check the box to left and hit `Attach policy`.
9. Go back to Lambda and to the function you just created.
10. Paste in the [code for the fourth function](https://gist.githubusercontent.com/taniacomputer/329c2d9ff1666b63d5ddb7d55296b9b4/raw/4db17ff176c62815fcbf013bebb1205ce418a285/slackapp_idSearch.py) into the space provided in the `Function Code` section.
11. Once pasted in hit `Save`.
12. Edit lines 19, 20, 22, 24
	- SM_ENDPOINT_URL (replace xx-yyyy-z with your AWS Region (ie us-east-2)
	- SM_REGION_NAME (same as above, replace xx-yyyy-z with your AWS Region
	- SLACK_CHANNEL_ID is the slack channel ID we replaced in the previous function
	- API_URL is the url of your Jamf Pro server, replace `jamf.com` with your base url.
13. Hit save once those lines are edited and click Add Trigger.
	- Select SNS
	- select `slackJamfIdSearch` from list
	- make sure `Enabled Trigger` is checked and hit `Add`.
	
	
## Update URLs in Slack &Kappa;
We will now add our API Endpoint URLs in our Slack app so it knows where it send requests.

1. go to `https://api.slack.com/apps`
2. Select the app you created earlier.
3. Select `Slash Commands` on the left and hit the pencil to edit. 
4. For request URL, enter in the `API Endpoint URL` from the first Lambda function and hit `Save`.
5. On the left, select `Interactivity & Shortcuts` and toggle the on switch.  Under `Request URL` enter the `API Endpoint URL` from the third Lambda function.  Hit `Save` at the bottom.
6. Select `Install App` on the left and then the green button to install the app to your workspace.


## Usage
You can now test the bot in your slack channel by typing `/jamf` and hitting return.  It should give you a help page.



