---
title: "AWS Resources Lifecycle Management Via ServiceNow and AWS Service Management Connector"
url: "https://aws.amazon.com/blogs/mt/aws-resources-lifecycle-management-via-servicenow-and-aws-systems-management-connector/"
date: "Tue, 06 Sep 2022 17:03:19 +0000"
author: "Ayo Omosebi"
feed_url: "https://aws.amazon.com/blogs/mt/tag/aws-service-catalog/feed/"
---
<p>Customers deploy series of AWS resources to support their workloads in the cloud. These organizations, as part of their maturity journey, must help managing the lifecycle of their AWS Resources using existing IT Service Management tool, such as ServiceNow. Manually executing these tasks via both consoles (ServiceNow and <a href="https://aws.amazon.com/console/">AWS Console</a>) is inefficient and time-tasking. With the recent release of the<a href="https://docs.aws.amazon.com/servicecatalog/latest/smcguide/overview.html"> AWS Service Management Connector (AWS SMC)</a> 4.5.x, this integration has become much easier.</p> 
<p>In this post, I’ll show you how to cover common use cases, including resources provisioning, incident/request management, cost optimization, and asset management audits in the CMDB. The use case will be the lifecycle management of <a href="https://docs.aws.amazon.com/workspaces/latest/adminguide/amazon-workspaces.html">Amazon WorkSpace</a>.</p> 
<p>The following high-level architecture diagram shows core solution components:</p> 
<div class="wp-caption aligncenter" id="attachment_31716" style="width: 1434px;">
 <img alt="Fig 1.0: High-level architecture diagram of core components" class="wp-image-31716" height="532" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/09/06/Screen-Shot-2022-09-06-at-12.17.48-PM.png" width="1424" />
 <p class="wp-caption-text" id="caption-attachment-31716">Fig 1.0: High-level architecture diagram of core components</p>
</div> 
<p>All of the steps to setup the solution are broken down into four major sections:</p> 
<ol> 
 <li>Configure AWS</li> 
</ol> 
<ul> 
 <li> 
  <ul> 
   <li>Make sure that the Support Plan is at a minimum of Business Plan</li> 
  </ul> </li> 
</ul> 
<ol start="2"> 
 <li>Complete ServiceNow instance set up with required plugin (Discovery and Service Mapping Patterns)</li> 
 <li>Establish integration between AWS and ServiceNow via Systems Management Connector v4.5.x</li> 
 <li>Perform lifecycle management of an Amazon WorkSpaces</li> 
</ol> 
<ul> 
 <li> 
  <ul> 
   <li>Provision new WorkSpaces</li> 
   <li>Incident/Event Management on WorkSpaces</li> 
   <li>Termination of a WorkSpaces via Change Request</li> 
  </ul> </li> 
</ul> 
<h2>Background</h2> 
<p><a href="https://docs.aws.amazon.com/servicecatalog/latest/smcguide/overview.html">AWS SMC for ServiceNow</a> enables ServiceNow end users to provision, manage and operate AWS resources natively through ServiceNow. The connector provides different features which minimize direct AWS console access, simplifying AWS product request and operational actions for ServiceNow end users. The same applies to ServiceNow administrators by streamlining Service Management governance and oversight over AWS resources and services.</p> 
<p>ServiceNow is an enterprise service management tool that places a service-oriented lens on the activities, tasks, and processes that enable day-to-day work life and a modern work environment.</p> 
<h2>Getting started</h2> 
<p>To deploy this solution, make sure that the following prerequisites have been completed.</p> 
<h2>AWS prerequisites</h2> 
<ol> 
 <li>An AWS Account with administrative access</li> 
 <li>Support Plan: Business (Minimum)</li> 
 <li>Make sure that Amazon WorkSpaces is available in your region (For this Blog’s Use case)</li> 
</ol> 
<h2>AWS SMC and ServiceNow prerequisites</h2> 
<ol> 
 <li>Admin level access to a ServiceNow Personal Developer instance (PDI) or Organization ServiceNow Instance</li> 
 <li>Complete integration between the AWS SMC for ServiceNow and ServiceNow by following this&nbsp;<a href="https://docs.aws.amazon.com/servicecatalog/latest/adminguide/integrations-servicenow.html">guide</a></li> 
</ol> 
<h2>Set up Amazon WorkSpaces directory</h2> 
<p>Amazon WorkSpaces use a directory to store and manage information, such as users, groups, and devices data for your WorkSpaces. If you have an existing directory (<a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/directory_microsoft_ad.html">AWS Managed Microsoft AD</a>, Simple AD, AD Connector, or <a href="https://aws.amazon.com/cognito/">Amazon Cognito</a> user Pools), then note the “Directory ID” which is required in the upcoming <a href="https://aws.amazon.com/cloudformation/">AWS CloudFormation</a> template.</p> 
<p>If you don’t have any directory, then follow the steps in this&nbsp;<a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/setting_up.html">guide</a>, and then note the “Directory ID”.&nbsp;This is a requirement before proceeding with this post’s content.</p> 
<h2>Installing a Workspace portfolio stack</h2> 
<p>In this section, you’ll launch the Amazon WorkSpaces portfolio stack, which requires you having a ‘Directory ID’ as stated earlier in the post.</p> 
<p>Select “<a href="https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=SC-RA-Workspaces-Portfolio&amp;templateURL=https://aws-service-catalog-reference-architectures.s3.amazonaws.com/workspaces/sc-portfolio-workspaces.json">Launch Stack</a>” to launch the WorkSpaces portfolio stack.</p> 
<ol type="a"> 
 <li>Accept default parameters, including stack name “<strong>SC-RA-Workspaces-Portfolio</strong>”</li> 
 <li>Within the Stack properties form, under <strong>Product Settings</strong>, enter your ‘DirectoryId’ as shown in the following</li> 
</ol> 
<div class="wp-caption aligncenter" id="attachment_31717" style="width: 710px;">
 <img alt="Fig 1.1: Product Settings during Stack creation." class="wp-image-31717" height="172" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/23/cloudops_911_1.1.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31717">Fig 1.1: Product Settings during Stack creation.</p>
</div> 
<p style="padding-left: 40px;">c. Accept both acknowledgement under “Capabilities”</p> 
<ol start="2"> 
 <li>Select “<strong>Create Stack</strong>”<br /> a. This creates a new “<strong>AWS WorkSpaces portfolio</strong>” with a single product and a launch constraint</li> 
 <li>Grant users access to the portfolio and allow them to view and launch the product from ServiceNow. This means adding the ‘SCEndUser’ as an allowed user to launch the request from ServiceNow service catalog.<br /> a. From the new service catalog portfolio, select “<strong>Groups</strong>, <strong>roles</strong>, and <strong>users</strong>” tab.</li> 
</ol> 
<div class="wp-caption aligncenter" id="attachment_31727" style="width: 710px;">
 <img alt="Fig 1.2: Grant users’ access to deployed AWS Service Catalog Portfolio" class="wp-image-31727" height="371" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/23/cloudops_911_1.2.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31727">Fig 1.2: Grant users’ access to deployed AWS Service Catalog Portfolio</p>
</div> 
<p style="padding-left: 40px;">b. Select “Add groups, roles, users” tab &gt; Users and select ‘<strong>SCEndUser</strong>’ user name<br /> c. Select “<strong>Add Access</strong>”</p> 
<div class="wp-caption aligncenter" id="attachment_31728" style="width: 710px;">
 <img alt="Fig 1.3: Confirmation of granted IAM User in specific Portfolio" class="wp-image-31728" height="173" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/23/cloudops_911_1.3.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31728">Fig 1.3: Confirmation of granted IAM User in specific Portfolio</p>
</div> 
<h2>Sync between AWS SMC and ServiceNow</h2> 
<p>Prior to starting with the following use case, run the sync job for the AWS Accounts in ServiceNow:</p> 
<ol> 
 <li>From ServiceNow, go to Scheduled Jobs and manually execute the “S<strong>ynchronize changes to all AWS accounts</strong>”.</li> 
</ol> 
<ol> 
 <li> 
  <ol type="a"> 
   <li>By default, it runs daily at midnight. This will make sure that all Service Catalog changes are synced with ServiceNow for the next use cases.</li> 
  </ol> </li> 
</ol> 
<h2>Authorize ServiceNow user to request an AWS Resource</h2> 
<ol> 
 <li>From your ServiceNow instance, browse to <strong>AWS Service Catalog</strong> &gt; Portfolio</li> 
 <li>Select portfolio from list: “<strong>Service Catalog – AWS Workspaces Reference Architecture</strong>”</li> 
 <li>Select Allowed Groups<br /> a. Click New.<br /> b. Type in the user group that will be provisioning the Amazon WorkSpaces.</li> 
</ol> 
<div class="wp-caption aligncenter" id="attachment_31729" style="width: 710px;">
 <img alt="Fig 1.4: Sample of authorized group “Order_AWS_Products” used by the end user in this blog" class="wp-image-31729" height="127" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/23/cloudops_911_1.4.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31729">Fig 1.4: Sample of authorized group “Order_AWS_Products” used by the end user in this blog</p>
</div> 
<h2>Step #1: Provisioning an Amazon WorkSpaces</h2> 
<ol> 
 <li>From your ServiceNow instance as an end user, launch Service Catalog and choose <strong>AWS Service Catalog</strong></li> 
 <li>Select from the product list: “<strong>AWS Workspaces application</strong>”</li> 
 <li>Fill in the required fields as follows:</li> 
</ol> 
<p style="padding-left: 40px;">a. Product Name:<br /> b. Product Version: Select “Easy Launch v1.0<br /> c. Parameters:</p> 
<p style="padding-left: 80px;">i. User Name: specify known user name: e.g., smctest<br /> ii. Workstation Type: choose from choice list; e.g., Standard-Win10-Desktop</p> 
<ol start="4"> 
 <li>Select “<strong>Order Now</strong>” to submit the request. This starts the provisioning of your WorkSpaces in the Console.</li> 
 <li>Select ‘Home’ to launch “My Assets” which lists your asset requests and additional information.</li> 
 <li>Select the configuration item “value= name of requested workspaces” and wait till product status=Provisioned and Status= Available</li> 
</ol> 
<div class="wp-caption aligncenter" id="attachment_31730" style="width: 710px;">
 <img alt="Fig 1.5: Results of a successfully provisioned Amazon WorkSpaces from ServiceNow Console" class="wp-image-31730" height="333" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/23/cloudops_911_1.5.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31730">Fig 1.5: Results of a successfully provisioned Amazon WorkSpaces from ServiceNow Console</p>
</div> 
<ol start="7"> 
 <li>Optionally, you can log in to the Amazon WorkSpaces console to view the status.</li> 
</ol> 
<h2>Step #2: Incident management on a WorkSpaces</h2> 
<p>Process an incident created from an event that occurred on the WorkSpaces.</p> 
<ol> 
 <li>Stop the listed services to trigger RDP disconnection (triggers Unhealthy Status in Amazon WorkSpaces console) for the logged in user:<br /> a. WorkSpaces required services:</li> 
</ol> 
<p style="padding-left: 80px;">i. SkyLightWorkspaceConfigService</p> 
<div class="wp-caption aligncenter" id="attachment_31733" style="width: 710px;">
 <img alt="Fig 1.7: Sample of SkyLightWorkspaceConfigService Service in Amazon WorkSpaces control services panel" class="wp-image-31733" height="154" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/23/cloudops_911_1.7.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31733">Fig 1.7: Sample of SkyLightWorkspaceConfigService Service in Amazon WorkSpaces control services panel</p>
</div> 
<p style="padding-left: 80px;">ii. PCoIP Standard Agent for Windows: Stop this last as it instantly drops the RDP Session.</p> 
<div class="wp-caption aligncenter" id="attachment_31734" style="width: 710px;">
 <img alt="Fig 1.8: Sample of PCoIP Standard Agent Service in Amazon WorkSpaces control services panel" class="wp-image-31734" height="123" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/23/cloudops_911_1.8.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31734">Fig 1.8: Sample of PCoIP Standard Agent Service in Amazon WorkSpaces control services panel</p>
</div> 
<p>NOTE: It will take approximately 25-30mins before the status shows “Unhealthy”. Amazon WorkSpace reachability checks are performed every 30mins.</p> 
<div class="wp-caption aligncenter" id="attachment_31735" style="width: 710px;">
 <img alt="Fig 1.9: Status of Amazon WorkSpaces as ‘Unhealthy” due to stopped services" class="wp-image-31735" height="242" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/23/cloudops_911_1.9.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31735">Fig 1.9: Status of Amazon WorkSpaces as ‘Unhealthy” due to stopped services</p>
</div> 
<p style="padding-left: 40px;">2. Create an incident in ServiceNow.</p> 
<div class="wp-caption aligncenter" id="attachment_31736" style="width: 710px;">
 <img alt="Fig 1.10: Incident record stating Amazon WorkSpaces is unreachable" class="wp-image-31736" height="326" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/23/cloudops_911_1.10.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31736">Fig 1.10: Incident record stating Amazon WorkSpaces is unreachable</p>
</div> 
<ol start="3"> 
 <li>Within incident form, select or search applicable automation document to fix incident “<strong>AWSSupport-RecoverWorkSpace</strong>”.</li> 
</ol> 
<div class="wp-caption aligncenter" id="attachment_31737" style="width: 710px;">
 <img alt="Fig 1.11: Related Search Results to recover Amazon WorkSpaces within incident record" class="wp-image-31737" height="87" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/23/cloudops_911_1.11.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31737">Fig 1.11: Related Search Results to recover Amazon WorkSpaces within incident record</p>
</div> 
<p style="padding-left: 40px;">a. Reboots affected WorkSpaces using the specified WorkSpaceID.</p> 
<div class="wp-caption aligncenter" id="attachment_31738" style="width: 710px;">
 <img alt="Fig 1.12: AWS Systems Manager Automation Catalog item “AWSSupport-RecoverWorkSpace” order form" class="wp-image-31738" height="335" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/23/cloudops_911_1.12.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31738">Fig 1.12: AWS Systems Manager Automation Catalog item “AWSSupport-RecoverWorkSpace” order form</p>
</div> 
<p style="padding-left: 40px;">b. View the Automation Execution status in ServiceNow.</p> 
<div class="wp-caption aligncenter" id="attachment_31739" style="width: 710px;">
 <img alt="Fig 1.13: Automation Document Execution status in ServiceNow" class="wp-image-31739" height="263" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/23/cloudops_911_1.13.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31739">Fig 1.13: Automation Document Execution status in ServiceNow</p>
</div> 
<p style="padding-left: 40px;">c. View the status in Amazon WorkSpaces console.</p> 
<div class="wp-caption aligncenter" id="attachment_31740" style="width: 710px;">
 <img alt="Fig 1.14: Rebooting Status based on Automation Document request from ServiceNow" class="wp-image-31740" height="159" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/23/cloudops_911_1.14.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31740">Fig 1.14: Rebooting Status based on Automation Document request from ServiceNow</p>
</div> 
<p style="padding-left: 40px;">d. The WorkSpaces becomes available once it is fully rebooted.</p> 
<div class="wp-caption aligncenter" id="attachment_31742" style="width: 710px;">
 <img alt="Fig 1.15: Status after full recovery in Amazon WorkSpaces Console" class="wp-image-31742" height="316" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/24/cloudops_911_1.15.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31742">Fig 1.15: Status after full recovery in Amazon WorkSpaces Console</p>
</div> 
<p style="padding-left: 40px;">e. Resolve the incident once the workspace Status =Available in CMDB, or in Amazon WorkSpaces console</p> 
<div class="wp-caption aligncenter" id="attachment_31743" style="width: 710px;">
 <img alt="Fig 1.16: Final Incident record status and Automation Execution in ServiceNow" class="wp-image-31743" height="264" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/24/cloudops_911_1.16.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31743">Fig 1.16: Final Incident record status and Automation Execution in ServiceNow</p>
</div> 
<h2>Step #3: Change request to “Terminate/Remove WorkSpaces”</h2> 
<p>For Cost Optimization purpose, any AWS resources that are underutilized or no longer needed must be terminated once the required data is backed up. For this use case, a request for termination is triggered as follows from ServiceNow:</p> 
<ol> 
 <li>From “<strong>My Assets</strong>” dashboard in ServiceNow, select specific CI “AWS_Workspaces_application-0701125856”<br /> a. Make sure that the WorkSpaceID matches the ID of the target WorkSpaces in the Console</li> 
 <li>Scroll down to “<strong>Related Links</strong>” and select “<strong>Request Termination</strong>”<br /> a. This triggers the termination workflow of the specific WorkSpaces in the Console</li> 
</ol> 
<div class="wp-caption aligncenter" id="attachment_31744" style="width: 710px;">
 <img alt="Fig 1.17: Status in Amazon WorkSpaces Console based on Termination Request from ServiceNow" class="wp-image-31744" height="277" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/24/cloudops_911_1.17.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31744">Fig 1.17: Status in Amazon WorkSpaces Console based on Termination Request from ServiceNow</p>
</div> 
<p style="padding-left: 40px;">b. The field ‘Product Status’=&gt; Terminated once the termination process is completed, thereby removing it completely from the Amazon WorkSpaces console.</p> 
<div class="wp-caption aligncenter" id="attachment_31745" style="width: 710px;">
 <img alt="Fig 1.18: Amazon WorkSpaces product status in ServiceNow CMDB as Terminated" class="wp-image-31745" height="294" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/24/cloudops_911_1.18.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31745">Fig 1.18: Amazon WorkSpaces product status in ServiceNow CMDB as Terminated</p>
</div> 
<p style="padding-left: 40px;">c. The change request can be closed per process once WorkSpace is fully terminated.</p> 
<div class="wp-caption aligncenter" id="attachment_31746" style="width: 710px;">
 <img alt="Fig 1.19: Closed Change Request ticket for the Amazon WorkSpaces termination in ServiceNow" class="wp-image-31746" height="316" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/24/cloudops_911_1.19.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31746">Fig 1.19: Closed Change Request ticket for the Amazon WorkSpaces termination in ServiceNow</p>
</div> 
<h2>Step #4: Configuration audits/inventory management</h2> 
<p>For audits trail purposes, the record for the terminated resource (WorkSpaces) will remain in ServiceNow CMDB until archived/deleted based on the organization retention policy. This allows the CI owner or business unit to audit the history of any provisioned AWS Resources via ServiceNow integration.</p> 
<ol> 
 <li>From “AWS Service Catalog” in ServiceNow, select Provisioned Products</li> 
 <li>Filter using specific CI “AWS_Workspaces_application-0701125856”</li> 
</ol> 
<div class="wp-caption aligncenter" id="attachment_31747" style="width: 710px;">
 <img alt="Fig 1.20: Amazon WorkSpaces Configuration Item (CI) inventory record in ServiceNow CMDB for audit purposes" class="wp-image-31747" height="84" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/24/cloudops_911_1.20.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31747">Fig 1.20: Amazon WorkSpaces Configuration Item (CI) inventory record in ServiceNow CMDB for audit purposes</p>
</div> 
<h2>Conclusion</h2> 
<p>In closing this post, I have showed you a typical lifecycle management of an AWS Resource (<a href="https://docs.aws.amazon.com/workspaces/latest/adminguide/amazon-workspaces.html">Amazon WorkSpaces</a>) using <a href="https://aws.amazon.com/cloudformation/">AWS CloudFormation</a>, <a href="https://aws.amazon.com/servicecatalog/">AWS Service Catalog</a>, <a href="https://aws.amazon.com/directoryservice/?nc2=type_a">AWS Directory Services</a> and ServiceNow. These services can be used to provision AWS Resources in your portfolio, manage incident and change requests, reduce cost on underutilized resources, and conduct inventory audits in the CMDB at scale.</p> 
<h2>Cleanup</h2> 
<p>NOTE: The Amazon WorkSpace was terminated in the step #3.</p> 
<p>This step removes all resources deployed during creation of the stack. This includes the AWS Service Catalog portfolio, product and launch constraint role.</p> 
<ol> 
 <li>Delete the created Stack “<strong>SC-RA-Workspaces-Portfolio</strong>”<br /> a. Go to <strong>CloudFormation</strong> Service in your AWS Console<br /> b. Click on Stacks<br /> c. Select the stack name “<strong>SC-RA-Workspaces-Portfolio</strong>”; created at the beginning of this blog.<br /> d. Click <strong>Delete</strong> from the available option.</li> 
</ol> 
<h2>Next Steps</h2> 
<p>I encourage you to follow the provided steps in your environment as a workshop.</p> 
<p>You can reach out to the team for support by signing up for SMC Activation Day via</p> 
<p><a href="mailto:aws-servicemanagement-connector@amazon.com">aws-servicemanagement-connector@amazon.com</a></p> 
<p><strong>About the author:</strong></p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="aligncenter size-full wp-image-11636" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/24/AyoBioPic.png" />
  </div> 
  <h3 class="lb-h4">Ayo Omosebi</h3> 
  <p>Ayo is a Sr. Business Services SDM at Amazon Web Services. He is passionate about building and promoting integrations between AWS services and customers business platforms. Outside of work, he enjoys spending time with his family outdoors, running and mountain hiking.</p> 
 </div> 
</footer>
