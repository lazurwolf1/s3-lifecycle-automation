# s3-lifecycle-automation
automation to manage s3 life cycle policies

First lets take a look at the cloudformation template we used to deploy this automation.  This infrastructure as code template builds out a few items used in this automation.
First I should mention this deployment does not require any existing resources before it can be setup inside your account.  However, since this tool is intended to help manage both new and existing buckets, it would make sense to deploy this tool in an account that has existing S3 buckets.
Also as we discussed, this solution uses a compliance framework to manage the S3 Life cycle polices that are being deployed.  While that frame work could be stored in many different forms from databases to to on-prem records, this deployment is storing this frame work as JSON files inside of an S3 bucket which will be referred to as the document bucket, and one of the items that is created by the deployment script is a bucket for this purpose.
Also the deployment script creates an Identify and Access Management Role or IAM Role that is both required and used by the automation for this solution to apply S3 Life Cycle polices to S3 buckets.  While this role is added to the newly created bucket by default.  After the cloud formation template finishes running you would need to modify the S3 Bucket Policies of any bucket that would be managed by this solution to add in this role.  At a minimum you would need to add permission to GET and PUT S3 life cycle policies.  This IAM role will show up as one of the resources created by the Cloud Formation template.
Finally  the cloud formation template creates a Systems Manage Automaiton Document also referred to as an SSM Automation.  This automation document is the solution and our demo is going to run this automation.
The automation runs in one of three different modes.  The initialize mode should only be run once after the solution is deployed.  It creates an example framework. If it run a second time you risk losing data.
The update  mode will be discussed later.
The Add mode is the primary way this tool is used. It allows you to add an S3 life cycle policy for the retention of data to a S3 bucket.
But before we add a Life Cycle policy lets quick look at two different things.  First let’s look at our framework document.  The primary document referred to as the GovRuleDocument for our demo already exists in the bucket that was created to hold it.  This is because we already ran this automation in initialize mode.
By going to my S3 bucket I can open this document.
As you can see in this document.  I have 3 geographical regions defined.  One includes all of the US and has a rule called Type1 that is used for that geo region.
The second region applies to Ireland and UK and has a rule called Type2
And the third applies to Germany and has a rule called Type3.
If I attempt to add a life cycle policy through this tool that violates this framework I will get an error.
We also see the rule defined retention times associated with these three rules.  For example in the Ireland UK geo region, the retention time is 870 days.  This is the region where we will be applying a new S3 life cycle policy.

The second thing we need to look at is the S3 bucket policy for the S3 bucket we are going to be adding a life cycle policy.
If I look at S3 and select my bucket - 4738-uni-bucket, then select on permissions, and scroll down I can see the bucket policy for this bucket.  I can see the IAM role used by my automation is applied to this bucket.
Also if I look at properties for this bucket I can see this bucket is in eu-west-1.  
Finally for this bucket if I look at management, I can see there are already 3 s3 life cycle polices applied to this bucket.  We will be adding a forth policy.

So back in the SSM automation.  I am going to leave most of these values as defaults, but I am going to:

* set the mode to Add
* Enter my bucket name - 4738-uni-bucket
* Set the region to eu-west-1
* set an lcName - this will be the name or ID of the life cycle policy.  This name needs to be unique.  A S3 bucket cannot have multiple life cycle policies with the same name.
* Set the goverence rule type or GoverancRule to Type 2 (which is the rule type required for this geo region)
* We are going to assume all of the data originated in S3 and know of it has a pre-age value.  So we are going to set the preAge to zero
* Set the S3 bucket prefix where this policy will apply.  We are going to use a value of new/data4.  Note:  this prefix does not need to exist yet.

Then we just scroll down and Execute the automation.

This will take a little while to run so while it does lets talk about the Update mode of execution.
As I showed you in the frame work document, each of the rules have a retention time set.  What happens if this governance rule is changed.  One of the major requirements our customer had was the ability to propagate changes to governance rules across all existing S3 life cycle policies.

So I am going to use a tool to update my framework and then upload it to my S3 bucket where those frameworks are kept.  In most cases you would want to tightly limit who has the capability of doing this type of update.

So we are going to set the retention time for the ireland/uk geo region to 500 days.  Save off those changes and as soon as the current automation is finished we will push these changes to the bucket.  The goal here is to update all of the appropriate S3 Life Cycle policies.  This will affect this new policy that is being written right now as well as previous polices that have already been written (which we saw there were 3 of those).  While this new policy has no pre-age associated with it.  Any policies that did have a pre-age will need that to be considered as the change to the rule is made.  If there was a rule change which because of pre-age would not have a negative retention time, by default that S3 life cycle policy is set to 1 day.  This effectively mean any data in this prefix should be removed ASAP.  You might want to generate further alerting in this scenario.

So lets check back in on our automation.

As we can see it has finished.  How I deployed this automation, if the last step that is run is the failure step that means the automation failed.  Please review the results of the automation to see why it failed.  Admittedly, additional or better logging might be warranted.

So lets look at our bucket to see if there is a new life cycle policy for the prefix new/data4.
<wait>. Sure enough there is and it as a retention time of 870 days.
The other thing the automation is doing is creating records so you can track when and how the tool was run.
The first record is the governance configuration record.  This tracks all the S3 buckets that have S3 life cycle policies applied to them via this tool.
As you can see we are currently managing three buckets.
The final and probably most important document is the LifeCycle Index document which shows all of the currently implemented life cycle polices.  
As you can see for the 4738-uni-buckeet, we have four policies.  This also shows the retention time used by this policy and any pre-age associated with this data set.

Lets go back and upload our changes to the governance rule document.

With those changes in place we are now going to run our automation in Update mode.  Technically the automation is not using any other values of than the defaults to run in this mode, but we cannot start the automation with any blank values - I know, we need a better software developer.  So just enter any value for the bucket field, select Update mode and execute the automation.  

Lets wait for a second for the automation to finish.

Now in the 4738-uni-bucket we can look at the life cycle policy and see that the retention time has been changed to reflect the new value we set for the rule.  When we look at the other life cycle policies in this bucket we can see that there value has changed as well.
