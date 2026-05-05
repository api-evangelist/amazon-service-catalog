---
title: "How to enable bulk self-service provisioning of Amazon WorkSpaces by using AWS Service Management Connector, AWS Service Catalog and ServiceNow Import sets"
url: "https://aws.amazon.com/blogs/mt/how-to-enable-bulk-self-service-provisioning-of-amazon-workspaces-by-using-aws-service-management-connector-aws-service-catalog-and-servicenow-import-sets/"
date: "Thu, 01 Sep 2022 16:35:54 +0000"
author: "Chandra Sekhar Chappa"
feed_url: "https://aws.amazon.com/blogs/mt/tag/aws-service-catalog/feed/"
---
<p><a href="https://aws.amazon.com/workspaces/">Amazon WorkSpaces</a> is a fully-managed, secure Desktop-as-a-Service (DaaS) solution that runs on AWS. AWS provides several choices to deploy desktops to users. Some organizations need help integrating this process into their existing automation and Information Technology Service Management (ITSM) tools. Many customers that we talk to want to have a bulk provisioning process, approval process, and a tracking mechanism for their <a href="https://aws.amazon.com/workspaces/">Amazon WorkSpaces</a> process.</p> 
<p>In this post, we’ll show you how to setup <a href="https://aws.amazon.com/servicecatalog/">AWS Service Catalog</a> and ServiceNow Data Source to deploy bulk WorkSpaces using the <a href="https://docs.aws.amazon.com/servicecatalog/latest/smcguide/sn-what-is.html">AWS Service Management Connector</a> for ServiceNow.</p> 
<p>The following high-level architecture diagram shows core solution components.</p> 
<div class="wp-caption aligncenter" id="attachment_31310" style="width: 710px;">
 <img alt="The following diagram summarizes end-user interactions. This flow shows AWS Service Catalog portfolios, API calls from ServiceNow and end user interactions" class="wp-image-31310" height="303" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/09/01/cloudops_767_New_1.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31310">Figure 1. The following diagram summarizes end-user interactions. This flow shows AWS Service Catalog portfolios, API calls from ServiceNow and end user interactions.</p>
</div> 
<p><img alt="The following diagram summarizes end-user interactions. This flow shows AWS Service Catalog portfolios, API calls from ServiceNow and end user interactions" class="aligncenter wp-image-31311" height="326" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/09/01/cloudops_767_New_2.png" width="700" /></p> 
<h2>Prerequisites</h2> 
<p>This post requires the following:</p> 
<ul> 
 <li>An AWS Account with administrative access</li> 
 <li>Access to an enterprise or a <a href="https://developer.servicenow.com/app.do#!/training/article/app_store_learnv2_buildmyfirstapp_jakarta_servicenow_basics/app_store_learnv2_buildmyfirstapp_jakarta_personal_developer_instances?v=jakarta">ServiceNow Personal Developer (PDI) instance</a></li> 
 <li>Administrator access to ServiceNow PDI or Enterprise Instance</li> 
 <li>Configure the AWS Service Catalog Connector for ServiceNow by following <a href="https://aws.amazon.com/blogs/mt/how-to-install-and-configure-the-aws-service-catalog-connector-for-servicenow/">this post</a>.</li> 
 <li>Download the code package from <a href="https://servicecatalogconnector.s3.amazonaws.com/Bulk-Import-Set-for-Amazon-WorkSpaces-main.zip">Bulk-Import-Set-for-Amazon-WorkSpaces-main.zip</a></li> 
</ul> 
<p>The overall&nbsp;steps to setup a solution&nbsp;can be broken down into three major categories:</p> 
<ol> 
 <li>Configure AWS (to set up a WorkSpace using Amazon WorkSpaces as an AWS Service Catalog product)</li> 
 <li>Install and configure ServiceNow (to setup the integration between AWS and ServiceNow)</li> 
 <li>Validate the WorkSpaces product in the ServiceNow Service Catalog</li> 
 <li>Configure Import Set process in ServiceNow, and upload the excel template for bulk provisioning</li> 
 <li>Run the Import Set using Transform Map logic</li> 
 <li>Validate the bulk request process</li> 
</ol> 
<h2>Setup a directory</h2> 
<p>Amazon WorkSpaces requires the use of a directory to store and manage information for your WorkSpaces and users. See the <a href="https://docs.aws.amazon.com/workspaces/latest/adminguide/manage-workspaces-directory.html">WorkSpaces Administration Guide on Managing Directories</a> for more information about directories. If you already have a directory (Simple Active Directory (AD), Microsoft AD, or AD Connector) deployed on AWS, then you can skip this section. If not, then you can follow the detailed steps from Appendix A to set up a directory that will be used to store user accounts for your WorkSpaces users.</p> 
<h2>Set up an AWS CloudFormation Template</h2> 
<p>In this section, you will set up an <a href="https://aws.amazon.com/cloudformation/">AWS CloudFormation</a> template that deploys WorkSpaces on your behalf. You can learn more about this step in the <a href="https://docs.aws.amazon.com/servicecatalog/latest/adminguide/getstarted-template.html">AWS Service Catalog Administrators Guide</a>.</p> 
<ol> 
 <li>From the downloaded package, open a text editor or your favorite code editor and copy the content of <strong>Workspaces yaml.yml</strong> and paste it into a new file.</li> 
 <li>In the Mappings section of the CloudFormation template, locate the three occurrences of the text “d-XXXXXXXXXX”, and replace all of these with the directory ID that you captured when you set up the directory (Appendix A).</li> 
 <li>Save the file on your computer as <strong>deploy-workspaces.template</strong> and note where you’re saving it.</li> 
</ol> 
<h2>Setup a new portfolio</h2> 
<p>To provide users with products, begin by creating a portfolio for those products. To create a portfolio, follow the detailed instructions in the <a href="https://docs.aws.amazon.com/servicecatalog/latest/adminguide/getstarted-portfolio.html">AWS Service Catalog documentation</a>.</p> 
<p>On the AWS Service Catalog console – Create Portfolio page, use the following values for creating the portfolio:</p> 
<ul> 
 <li><strong>Portfolio name</strong> – End-User-Compute</li> 
 <li><strong>Description</strong> – Portfolio for EUC products such as desktops</li> 
 <li><strong>Owner</strong> – IT (<a href="mailto:it@example.com">it@example.com</a>)</li> 
</ul> 
<h2>Set up a new product</h2> 
<p>After you’ve created a portfolio, add a new product using detailed instructions in the <a href="https://docs.aws.amazon.com/servicecatalog/latest/adminguide/getstarted-product.html">AWS Service Catalog documentation</a>.</p> 
<p>On the AWS Service Catalog console – Upload New Product page, use the following values for creating the product:</p> 
<ul> 
 <li><strong>Product name</strong> – Virtual Windows Desktop</li> 
 <li><strong>Description</strong> – Virtual windows desktop powered by Amazon WorkSpaces</li> 
 <li><strong>Provided by</strong> – IT</li> 
 <li><strong>Vendor</strong> (optional) – Amazon Web Services</li> 
</ul> 
<p>On the <strong>Enter support details</strong> page, type the following, and then choose NEXT:</p> 
<ul> 
 <li><strong>Email contact</strong> – ITSupport@example.com</li> 
 <li><strong>Support link</strong> – Link to your IT team’s contact us or support page (e.g.,https://aws.amazon.com/contact-us/)</li> 
 <li><strong>Support description</strong> – Contact IT department for further help</li> 
</ul> 
<p>On the <strong>Version details</strong> page, choose<strong> Upload a template file</strong>, select <strong>Choose File</strong>, locate the <strong>deploy-workspaces.template</strong> file that you saved when you set up the CloudFormation template, and then choose NEXT:</p> 
<ul> 
 <li><strong>Version title</strong> – 1.0.0</li> 
 <li><strong>Description</strong> – Initial Version</li> 
</ul> 
<p>On the <strong>Review</strong> page, choose <strong>CREATE</strong>.</p> 
<div class="wp-caption aligncenter" id="attachment_31312" style="width: 710px;">
 <img alt="upload a new product" class="wp-image-31312" height="391" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/09/01/cloudops_767_New_3.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-31312">Figure 2. Upload a new product</p>
</div> 
<h2>Enable AWS Service Catalog to launch Amazon WorkSpaces</h2> 
<p>To enable the AWS Service Catalog to launch WorkSpaces, you must grant additional security privileges. You achieve that through additional <a href="https://aws.amazon.com/iam/">AWS Identity and Access Management (IAM)</a> permissions and a <a href="https://docs.aws.amazon.com/servicecatalog/latest/adminguide/constraints-launch.html">launch constraint</a>. A launch constraint specifies the IAM role that AWS Service Catalog assumes when an end user launches a product.</p> 
<h2>Configure IAM permissions</h2> 
<p>In this step, we’ll set up an IAM policy and modify an existing role. Make sure that you have followed the steps for integration prerequisites, discussed earlier, prior to starting this section.</p> 
<h2>Create IAM policy</h2> 
<p>In this step, you’ll create an IAM policy ‘SCWorkSpacesLaunchPermissions’ to match the following permissions. To create an IAM policy, follow the detailed instructions in the<a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_create.html#access_policies_create-json-editor"> IAM User Guide</a>.</p> 
<pre class="unlimited-height-code"><code class="lang-json">{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "workspaces:*",
            "Resource": "*"
        }
    ]
}</code></pre> 
<p>In the IAM console, on the Review policy page, fill in the form as follows:</p> 
<ol> 
 <li><strong>Name</strong> – SCWorkSpacesLaunchPermissions</li> 
 <li><strong>Description</strong> – Allows the ability to launch WorkSpaces</li> 
</ol> 
<h2>Modify IAM role</h2> 
<p>Modify the existing SCConnectLaunch role and attach the SCWorkSpacesLaunchPermissions policy to it. Refer to Appendix B for detailed instructions.</p> 
<h2>Launch constraints</h2> 
<p>A launch constraint specifies the IAM role that AWS Service Catalog assumes when an end user launches a product. For the new Virtual Windows Desktop product, assign the launch constraint ‘SCConnectLaunch’ before it can be launched. Refer to Appendix C for detailed instructions.</p> 
<h2>Validate Service Catalog Product in ServiceNow</h2> 
<p>You’re now ready to validate that the new product appears in ServiceNow, and that you can order a product through the ServiceNow Service Catalog.</p> 
<ol> 
 <li>Log in to your ServiceNow instance as the end user (e.g., Abel Tuter). If you’re logged into a developer instance as the administrator, then you can do this by choosing Impersonate User from the user menu in the upper-right corner of your screen.</li> 
 <li>Type <strong>Service Catalog</strong> in the navigation filter, and choose <strong>Service Catalog</strong>.</li> 
 <li>Choose <strong>AWS Service Catalog</strong>.</li> 
</ol> 
<p><img alt="" class="aligncenter wp-image-31314" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/09/01/cloudops_767_New_4.png" /></p> 
<ol start="4"> 
 <li>You should now see the AWS Service Catalog product:</li> 
</ol> 
<p><img alt="ServiceNow Catalog page with launch role and parameter information from AWS Service Catalog product" class="aligncenter wp-image-31315" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/09/01/cloudops_767_New_5.png" /></p> 
<ol start="5"> 
 <li>Select Virtual Windows Desktop.</li> 
 <li>Fill in the order form as follows:</li> 
</ol> 
<ol> 
 <li> 
  <ol start="1" type="a"> 
   <li><strong>Product Name</strong> – Type any meaningful name, such as MyCloudDesktop.</li> 
   <li><strong>WorkStationType</strong> – Choose your type of workstation, Value, Standard, or Performance. If you modified your CloudFormation template to include different bundle names, then they should appear here.</li> 
   <li><strong>UserName</strong> – Type the Windows user name that you specified when you created the user (Appendix A). If you’re unable to provision a WorkSpace using the user ID you enter here, then ask your Active Directory administrator for your SAMAccountName.</li> 
  </ol> </li> 
</ol> 
<p><img alt="" class="aligncenter wp-image-31317" height="506" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/09/01/cloudops_767_New_6.png" width="700" /></p> 
<ol start="7"> 
 <li>This confirms that the AWS Service Catalog product is successfully synchronized into ServiceNow and we can view the parameters of the WorkSpace product.</li> 
</ol> 
<h3>Create an Excel Template with provisioning parameters</h3> 
<ol> 
 <li>For bulk provisioning, you’ll create an excel sheet for the number of WorkSpaces required, and enter the parameter details <strong>WorkstationType</strong>, and <strong>Username</strong>. In the following example, you’ll provision eight WorkSpaces with username starting from awssmc1 – awssmc8. In an enterprise setup, you may have additional parameters and all of them will go into your excel spreadsheet columns.</li> 
</ol> 
<p>Workstation parameters screen</p> 
<table border="2" cellpadding="2" cellspacing="3" style="border-color: #000000; background-color: #ebe8e8;"> 
 <tbody> 
  <tr> 
   <td style="text-align: center;"><strong>User Name</strong></td> 
   <td style="text-align: center;"><strong>Workstation Type</strong></td> 
   <td style="text-align: center;"><strong>Workstation Name</strong></td> 
  </tr> 
  <tr> 
   <td><strong>awssmc1</strong></td> 
   <td><strong>Value_Win10_Desktop</strong></td> 
   <td><strong>Value_Windows_Desktop-01</strong></td> 
  </tr> 
  <tr> 
   <td><strong>awssmc2</strong></td> 
   <td><strong>Value_Win10_Desktop</strong></td> 
   <td><strong>Value_Windows_Desktop-02</strong></td> 
  </tr> 
  <tr> 
   <td><strong>awssmc3</strong></td> 
   <td><strong>Value_Win10_Desktop</strong></td> 
   <td><strong>Value_Windows_Desktop-03</strong></td> 
  </tr> 
  <tr> 
   <td><strong>awssmc4</strong></td> 
   <td><strong>Value_Win10_Desktop</strong></td> 
   <td><strong>Value_Windows_Desktop-04</strong></td> 
  </tr> 
  <tr> 
   <td><strong>awssmc5</strong></td> 
   <td><strong>Value_Win10_Desktop</strong></td> 
   <td><strong>Value_Windows_Desktop-05</strong></td> 
  </tr> 
  <tr> 
   <td><strong>awssmc6</strong></td> 
   <td><strong>Value_Win10_Desktop</strong></td> 
   <td><strong>Value_Windows_Desktop-06</strong></td> 
  </tr> 
  <tr> 
   <td><strong>awssmc7</strong></td> 
   <td><strong>Value_Win10_Desktop</strong></td> 
   <td><strong>Value_Windows_Desktop-07</strong></td> 
  </tr> 
  <tr> 
   <td><strong>awssmc8</strong></td> 
   <td><strong>Value_Win10_Desktop</strong></td> 
   <td><strong>Value_Windows_Desktop-08</strong></td> 
  </tr> 
 </tbody> 
</table> 
<ol start="2"> 
 <li>Save the excel file on your computer and name it “<strong>Worskpaces Bulk Upload Excel.xslx</strong>”.</li> 
</ol> 
<h2>Configuring import set process in ServiceNow for bulk provisioning</h2> 
<p>In this step we will create a script Include, transform target table, import set data source, and transform map in your ServiceNow instance to enable bulk provisioning mechanism for WorkSpaces.</p> 
<ol> 
 <li>Set the application as AWS Service Management Connector on your fulfillment portal from Global.</li> 
 <li>Navigate to System Definitions à Script Includes, and select New and set the 
  <ol start="1" type="a"> 
   <li>Name – “ServiceCatalogServer”</li> 
   <li>Application – AWS Service Management Connector</li> 
   <li>Accessible from – All Application Scopes</li> 
   <li>Active – true (select the checkbox)</li> 
   <li>For the script section, enter the content of <strong>ScriptInclude-ServiceCatalogServer</strong> from the downloaded package and save the form</li> 
  </ol> </li> 
</ol> 
<ol start="3"> 
 <li>Create a table in ServiceNow. Navigate to System Definition à Tables, and select New 
  <ol start="1" type="a"> 
   <li>Label – “WorkSpaces Bulk Upload Repo”</li> 
   <li>Name – Will be auto populated after entering the label column</li> 
   <li>Under Columns tab – Select “Insert a new row” and add three rows for UserName, WorkSpace Name, and Workstation Type as shown in the following, and Submit.</li> 
  </ol> </li> 
</ol> 
<p><img alt="ServiceNow transform target table creation" class="aligncenter wp-image-31349" height="341" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/09/01/cloudops_767_New_7.png" width="700" /></p> 
<ol start="4"> 
 <li>Now you’ll create the Import Set Data Source. Navigate to System Import Sets <strong>-&gt;</strong> Administration <strong>-&gt;</strong> Data Sources, select New, and enter the following field values.</li> 
</ol> 
<p style="padding-left: 40px;">Enter the following parameters to create the new Data Source:</p> 
<ol> 
 <li> 
  <ol start="1" type="a"> 
   <li><strong>Name</strong> – Bulk WorkSpaces Upload</li> 
   <li><strong>Import set table name</strong> – “x_126749_aws_sc_u_worskpace_bulk_upload”</li> 
   <li><strong>Type</strong> – File</li> 
   <li><strong>Format</strong> – Excel (xlsx.xls)</li> 
   <li><strong>Save the record</strong></li> 
  </ol> </li> 
</ol> 
<ol start="5"> 
 <li>On the page, you can see an attachment option with a clip icon. Select it and upload the spreadsheet previously created, <strong>WorkSpaces Bulk Upload Excel.xslx</strong>.</li> 
 <li>On Data Source – select “<strong>Load All Records</strong>” UI Action.</li> 
 <li>On the next page, select Create Transform map.</li> 
</ol> 
<p><img alt="Create Transform Map for Import Set process" class="aligncenter wp-image-31327" height="165" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/09/01/cloudops_767_New_8.png" width="700" /></p> 
<ol start="8"> 
 <li>Enter Name as – “TM-WorkSpaces” and select the Target Table as “WorkSpaces Bulk Upload Repo”, then right-click the hamburger icon to save the form.</li> 
 <li>Select the UI Action “Mapping Assist” and map the fields from Source Table and Target Table as shown in the following screenshot, and save.</li> 
</ol> 
<p><img alt="Mapping Assist for import set table transformation" class="aligncenter wp-image-31322" height="185" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/09/01/cloudops_767_New_9.png" width="700" /></p> 
<ol start="10"> 
 <li>Under the “Transform Scripts” tab, select New.</li> 
</ol> 
<ol> 
 <li> 
  <ol> 
   <li>Select the “When” column to “OnAfter”</li> 
   <li>On the script tab, enter the content of <strong>OnAfter – transform script.txt</strong> from the downloaded package and save the form</li> 
   <li>Update the productSysID marked in yellow to the sys_id of AWS Service Catalog Product – “Virtual Windows Desktop”.</li> 
  </ol> </li> 
</ol> 
<ul> 
 <li> 
  <ul> 
   <li>To know the sys id on your servicenow instance, navigate to AWS Service Management Connector<strong> -&gt;</strong> AWS Service Catalog <strong>-&gt;</strong> Products <strong>-&gt;</strong> Name (Search with Virtual Windows Desktop). <strong>-&gt;</strong> Right-click and select “Copy sys_id”.</li> 
  </ul> </li> 
</ul> 
<h2>Run the Import Set using Transform Map logic</h2> 
<p>In this step, we will execute the import using the field mapping logic and the onAfter transform script to execute bulk provisioning of WorkSpaces.</p> 
<ol> 
 <li>Now let’s run the Transform map to complete the import set process. On the Table Transform Map select the UI Action “Transform”.</li> 
</ol> 
<p><img alt="Transform Map with field mappings and transform scripts" class="aligncenter wp-image-31324" height="383" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/09/01/cloudops_767_New_10.png" width="700" /></p> 
<ol start="2"> 
 <li>Transform will now run the import set process to do a bulk provisioning of WorkSpaces.</li> 
</ol> 
<p><img alt="" class="aligncenter wp-image-31326" height="146" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/09/01/cloudops_767_New_11.png" width="700" /></p> 
<ol start="3"> 
 <li>Successful completion of the Transform map will result in the following screenshot.</li> 
</ol> 
<p><img alt="" class="aligncenter wp-image-31327" height="165" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/09/01/cloudops_767_New_12.png" width="700" /></p> 
<h2>Validating the bulk requests</h2> 
<p>In this step we will validate the Requested Items (RITM’s) that were created as a part of bulk-provisioning process, and the successful provisioning process that includes the outputs of provisioned WorkSpaces.</p> 
<ol> 
 <li>On the ServiceNow Navigation Filter, type Requested Items and open the list view and apply the filter “Item is Virtual Windows Desktop” and “Created on Last Minute”.</li> 
</ol> 
<p><img alt="Requested Items List generated from bulk provisioning" class="aligncenter wp-image-31328" height="257" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/09/01/cloudops_767_New_13.png" width="700" /></p> 
<ol start="2"> 
 <li>System Administrators can now bulk provision WorkSpaces and each requested item will have its own lifecycle process. They can be further modified by adding additional approvals to the workflow depending on Username, group, etc.</li> 
 <li>Navigate to AWS Service Catalog<strong> -&gt;</strong> Provisioned Products <strong>-&gt;</strong>List View to view the output parameters for any provisioned product. Select any of the recently provisioned products. The following are the outputs from the CloudFormation output parameters that users can use to start interacting with the cloud desktop.</li> 
</ol> 
<p><img alt="Provisioned Product Outputs for Workspace" class="aligncenter wp-image-31330" height="426" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/09/01/cloudops_767_New_14.png" width="700" /></p> 
<ol start="4"> 
 <li>After WorkSpaces provisioning is complete, the user should receive an email from AWS with complete instructions on how to complete the user profile and log in to the WorkSpaces instance. Make sure that you complete your user profile first.</li> 
</ol> 
<p><img alt="Workspaces email information to complete user profile" class="aligncenter wp-image-31331" height="247" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/09/01/cloudops_767_New_15.png" width="700" /></p> 
<p>The following shows at a high level how you can log in to your WorkSpaces after the user profile completion. Start by <a href="https://clients.amazonworkspaces.com/">downloading the WorkSpaces client</a> for your platform.</p> 
<p><img alt="workspaces client after logging" class="aligncenter wp-image-31333" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/09/01/cloudops_767_New_16.png" /></p> 
<ol start="5"> 
 <li>After you’ve installed the WorkSpaces client, log in by using your username and associated credentials.</li> 
</ol> 
<p><img alt="" class="aligncenter wp-image-31334" height="426" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/09/01/cloudops_767_New_17.png" width="700" /></p> 
<h2>Conclusion</h2> 
<p>In this post, we covered how you can use AWS Service Catalog, AWS Service Management Connector and ServiceNow import sets to create a fully automated, self service desktop solution for bulk deploying of WorkSpaces. This simplifies IT Service Management platform administrators, HR partners to onboard new users to WorkSpaces with self-service mechanism from ServiceNow.</p> 
<p><strong>Authors</strong>:</p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="aligncenter size-full wp-image-11636" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/15/Chandra-Chappa.jpg" />
  </div> 
  <h3 class="lb-h4">Chandra Chappa</h3> 
  <p>Chandra Chappa is a Denver based Sr. Service Management Specialist with AWS Service Management Connector. Chandra enjoys helping customers enable end-to-end IT lifecycle management to AWS Field, Customers, and Solutions Architect Partners. In his free time, he likes playing local club cricket and enjoys spending time with family and friends.</p> 
 </div> 
</footer> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="aligncenter wp-image-11636" height="160" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/15/Joe-Thomas.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Joe Thomas</h3> 
  <p>Joe Thomas is a Systems Development Engineer with the AWS Service Management Connectors team. He is passionate about driving operational excellence, troubleshooting customer issues and enjoys building product features and enhancements that drive customer adoption and satisfaction. Outside of work, Joe enjoys spending time with his family on short summer hikes or traveling across the country exploring the various food cultures.</p> 
 </div> 
</footer> 
<h3>Appendices</h3> 
<h3>Appendix A – Create directory</h3> 
<p>In the context of testing or proof-of-concept work, we recommend that you deploy Simple AD if you don’t already have a directory setup. Simple AD is a cost-effective solution to get your environment ready for deploying Amazon WorkSpaces quickly. To create a Simple AD directory, follow the steps in&nbsp;<a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/how_to_create_simple_ad.html">Create a Simple AD Directory</a>.</p> 
<h3>Add Users</h3> 
<p>Before you can use ServiceNow to deploy Amazon WorkSpaces, you must set up user accounts in the directory for the people for which you’ll create Amazon WorkSpaces.</p> 
<p>Note that if you have AD Connector set up, then users would already exist in your directory, thereby allowing you to skip this step.</p> 
<h3>To add users to Simple AD directory</h3> 
<ol> 
 <li>Open the <a href="https://console.aws.amazon.com/workspaces/">Amazon WorkSpaces console</a>. Make sure that the correct Region is selected in the upper right of the console.</li> 
 <li>Choose<strong> Launch WorkSpaces</strong>.</li> 
 <li>Select your directory from the list and choose <strong>Next Step</strong>.</li> 
 <li>Type the Username, First Name, Last Name, and valid email address for the first user that you want to add.</li> 
</ol> 
<p>Note that if you don’t specify a valid email address, then the user won’t be able to log in.</p> 
<p><img alt="" class="aligncenter wp-image-31419" height="205" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/16/cloudops_767_24_Apendix_1.png" width="700" /></p> 
<ol start="5"> 
 <li>If you want to add more than just one user, then choose <strong>+ Create Additional Users</strong>. This will add more rows to the form.</li> 
 <li>Choose <strong>Create Users</strong>.</li> 
 <li>Choose <strong>Cancel</strong> at the bottom of the form. We don’t actually want to allocate WorkSpaces to these users at this time, just create the accounts.</li> 
</ol> 
<h3>Obtain WorkSpaces Directory ID</h3> 
<p>Each directory that you set up will be provisioned with a unique directory ID. It’s necessary to acquire at least one of these directory IDs from your Amazon WorkSpaces deployment. This is needed in the next section and is used to tell your CloudFormation template under which directory to deploy Amazon WorkSpaces.</p> 
<ol> 
 <li>Open the <a href="https://console.aws.amazon.com/workspaces/">Amazon WorkSpaces console</a>. Make sure that the correct Region is selected in the upper right of the console.</li> 
 <li>In the left navigation panel, choose <strong>Directories</strong>.</li> 
 <li>Check the box next to the directory that you want, then highlight the text of the <strong>Directory ID</strong> (d-XXXXXXXXXX) to <strong>copy</strong> it to your clipboard.</li> 
</ol> 
<p><img alt="" class="aligncenter size-full wp-image-31420" height="115" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/16/cloudops_767_25_Apendix_2.png" width="626" /></p> 
<p>Paste this ID somewhere where you can get back to it easily for a later step.</p> 
<h3>Appendix B – To modify the IAM role</h3> 
<ol> 
 <li>Open the <a href="https://console.aws.amazon.com/iam/home#/roles">IAM console roles page</a>.</li> 
 <li>Locate the SCConnectLaunch role in the list or type the name in the search box, and then choose the role in the Role name column.</li> 
 <li>Choose <strong>Attach policy</strong>.</li> 
 <li>In the search box, begin typing SCWorkSpacesLaunchPermissions to locate the policy.</li> 
 <li>Select the checkbox in the first column to assign that policy.</li> 
 <li>Choose <strong>Attach policy</strong>.</li> 
 <li>On the role summary screen, and choose the copy icon next to the <strong>Role ARN</strong> field. This will copy the ARN to your clipboard.</li> 
 <li>Paste the ARN somewhere for safekeeping (e.g., Notepad). You’ll need it in the next section.</li> 
</ol> 
<h3>Appendix C – To add a launch constraint</h3> 
<ol> 
 <li>Open the A<a href="https://console.aws.amazon.com/servicecatalog/">WS Service Catalog</a> console.</li> 
 <li>Open the <strong>End-User-Compute</strong> portfolio that we previously created.</li> 
 <li>Expand <strong>Constraints</strong>.</li> 
 <li>Choose the <strong>ADD CONSTRAINTS</strong> link.</li> 
 <li>You should see the following dialog box. 
  <ol> 
   <li><strong>Product</strong> – Virtual Windows Desktop</li> 
   <li><strong>Constraint type</strong> – Launch</li> 
  </ol> </li> 
</ol> 
<p><img alt="" class="aligncenter size-full wp-image-31421" height="549" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/16/cloudops_767_25_Apendix_3.png" width="554" /></p> 
<ol start="6"> 
 <li>Choose <strong>CONTINUE</strong>.</li> 
 <li>You’ll be prompted for the IAM role and description. 
  <ol> 
   <li><strong>IAM role</strong> – There are two boxes, paste the SCConnectLaunch role ARN that you set up in <a href="https://aws.amazon.com/blogs/mt/how-to-enable-self-service-amazon-workspaces-by-using-aws-service-catalog-connector-for-servicenow/#Appendix-B">Appendix B</a> into the second box.</li> 
  </ol> </li> 
</ol> 
<ol start="8"> 
 <li>Description – Ability to only launch Amazon WorkSpaces.</li> 
</ol> 
<p><img alt="" class="aligncenter size-full wp-image-31422" height="284" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/08/16/cloudops_767_26_Apendix_4.png" width="611" /></p> 
<ol start="9"> 
 <li>Choose SUBMIT.</li> 
</ol>
