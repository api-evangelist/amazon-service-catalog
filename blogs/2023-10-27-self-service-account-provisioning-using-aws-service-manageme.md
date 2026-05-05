---
title: "Self-service Account Provisioning Using AWS Service Management Connector for ServiceNow"
url: "https://aws.amazon.com/blogs/mt/self-service-account-provisioning-using-aws-service-management-connector-for-servicenow/"
date: "Fri, 27 Oct 2023 14:08:18 +0000"
author: "Siddhartha Angara"
feed_url: "https://aws.amazon.com/blogs/mt/tag/aws-service-catalog/feed/"
---
<p>Many customers are looking to adopt a multi-account strategy within their AWS environment. This allows customers to isolate their workloads into different environments including test, dev, and production in addition to separating workloads based on regulatory requirements. As customers scale their multi-account environments, one strategy to increase agility is to offer business units their own sandbox accounts for experimental and testing purposes. To do so rapidly, business teams want the ability to self-serve provisioning of their own accounts and resources.</p> 
<p>For organizations that have adopted ServiceNow, IT operations teams can offer business teams the ability to self-service provisioning of accounts through a familiar interface. This is accomplished by connecting <a href="https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html">AWS Control Tower</a> and <a href="https://docs.aws.amazon.com/servicecatalog/latest/adminguide/introduction.html">AWS Service Catalog</a> into ServiceNow using the <a href="https://docs.aws.amazon.com/smc/latest/ag/overview.html">AWS Service Management Connector</a> which enables ServiceNow end users to provision, manage, and operate AWS resources natively through ServiceNow. AWS Control Tower enables customers to set up and govern a multi-account environment at scale. For guidance building a well-structured foundation and isolating AWS services by function and team, customers can reference the <a href="https://docs.aws.amazon.com/pdfs/prescriptive-guidance/latest/security-reference-architecture/security-reference-architecture.pdf">Security Reference Architecture</a>.</p> 
<p>In this blog post, we’ll demonstrate how to set up an environment that allows end users the ability to self-serve provisioning of accounts through ServiceNow. By the end of this hands-on session, you should be able to:</p> 
<ul> 
 <li>Integrate AWS Service Catalog (AWS Control Tower Account Factory Portfolio) with ServiceNow ITSM Portal using AWS Service Management Connector to enhance Cloud Management.</li> 
 <li>Order and provision an AWS Account as an end user from ServiceNow Portal in a standardized, secure, and governed fashion.</li> 
</ul> 
<h2><strong>Prerequisites</strong></h2> 
<p><strong>AWS</strong></p> 
<ul> 
 <li>Obtain/use a clean AWS account with admin credentials. Although we’re using admin privileges for the purpose of this blog, it is security best practice to apply <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#grant-least-privilege">least-privilege</a> permissions and grant only the permissions required to perform a task.</li> 
 <li>Create Landing Zone using <a href="https://docs.aws.amazon.com/controltower/latest/userguide/quick-start.html">AWS Control Tower</a>.</li> 
</ul> 
<p><strong>ServiceNow</strong></p> 
<ul> 
 <li>Obtain ServiceNow <a href="https://developer.servicenow.com/dev.do#!/learn/learning-plans/utah/new_to_servicenow/app_store_learnv2_buildmyfirstapp_utah_personal_developer_instances">Personal Developer Instance</a> or use clean ServiceNow Developer Environment. You will need an Account that has admin privileges to perform the configuration steps in ServiceNow.</li> 
</ul> 
<h2><strong>Part 1: AWS Configuration</strong></h2> 
<p>This section describes the configuration required on AWS. At a high-level, the following steps will be performed:</p> 
<ol> 
 <li>Verify AWS Control Tower Landing Zone is set up.</li> 
 <li>Verify AWS Service Catalog Portfolio Product is active.</li> 
 <li>Create two IAM Users for AWS Service Management Connector:<br /> <em>SCSyncUser</em><br /> <em>SCEndUser</em></li> 
 <li>Create two IAM Policies:<br /> <em>Demo-ControlTower-FullAccess</em><br /> <em>Demo-SSO-FullAccess</em></li> 
 <li>Create an IAM Role <em>Demo-Role-For-LaunchConstraint</em> and attach required policies.</li> 
 <li>Create AWS Service Catalog Launch Constraint and assign the role <em>Demo-Role-For-LaunchConstraint.</em></li> 
 <li>Grant access to the Service Catalog portfolio to users: <em>SCSyncUser</em> and <em>SCEndUser</em>.</li> 
</ol> 
<p><strong>Step </strong>1: In your AWS Console, verify <a href="https://console.aws.amazon.com/controltower">AWS Control Tower</a> is setup in the <em>US East (N. Virginia)</em> region.</p> 
<p><img alt="Fig 1 shows Control Tower is setup and ready to use" class="alignnone size-full wp-image-45724" height="342" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/1.png" width="1324" /></p> 
<p><strong>Step 2</strong>: Verify that the status of <strong>AWS Control Tower Account Factory</strong> Portfolio version is Active from the <strong>AWS Service Catalog</strong> service in your AWS Console.</p> 
<p><img alt="" class="alignnone size-full wp-image-45736" height="913" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/2-1.png" width="1347" /></p> 
<p><strong>Step 3: Create IAM Users</strong></p> 
<p>For each AWS account, the Connector for ServiceNow requires two IAM users:</p> 
<ul> 
 <li><strong>AWS Sync User</strong>: An IAM user to sync AWS resources (such as potfolios and products) to ServiceNow.</li> 
 <li><strong>AWS End User</strong>: An IAM user who can provision products as an end user, execute requests, and view resources that ServiceNow exposes. This role includes any required roles to provision and execute.</li> 
</ul> 
<p>After creating each user, make a note of the access keys details as you would need them when configuring the new AWS Account in ServiceNow.</p> 
<p>To add the users, Navigate to the <a href="https://us-east-1.console.aws.amazon.com/iamv2/home#/home">Identity and Access Management</a> (IAM) service.</p> 
<p><strong>Step 3.1</strong>: Create User 1</p> 
<ol> 
 <li>Create the first user <em>SCSyncUser</em>, assign Permissions and create Access key (Programmatic Access) as shown in the screenshots.</li> 
 <li>Select “<em>Users</em>” from the left-hand menu and then choose “<em>Add users</em>” button.</li> 
 <li>Enter “<em>SCSyncUser</em>” for the user name. Since this user will only require programmatic access, leave the box unchecked for access to the AWS Management Console.<img alt="Fig 3 shows creating a user called SCSyncUser" class="alignnone size-full wp-image-45737" height="456" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/3-1.png" width="1321" /></li> 
 <li>Choose ‘<em>Next</em>’.</li> 
 <li>Select the permissions option ‘<em>Attach policies directly</em>’ and search for the policy “<em>AWSServiceCatalogAdminReadOnlyAccess</em>”. Select the check-box against the policy name.<img alt="Fig 4 shows how to attach a policy to SCSyncUser" class="alignnone size-full wp-image-45738" height="740" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/4-2.png" width="2457" /></li> 
 <li>Choose ‘<em>Next’</em>, <em>Review</em> the details and choose ‘<em>Create User</em>’ to complete the User Creation.</li> 
 <li>To create an access key for this user, select “<em>Users</em>” from the left-hand menu and then choose <em>SCSyncUser</em>.</li> 
 <li>Select the ‘<em>Security credentials</em>’ tab.&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<img alt="Fig 5 shows steps to adding access key to SCSyncUser" class="alignnone size-full wp-image-45739" height="379" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/5-2.png" width="745" /></li> 
 <li>Scroll down to the section for ‘<em>Access keys</em>’ and select ‘<em>Create access key</em>’.<img alt="Fig 6 shows steps to adding access key to SCSyncUser" class="alignnone size-full wp-image-45740" height="204" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/6.png" width="1212" /></li> 
 <li>Select the radio button option for ‘<em>Third-party service</em>’, check the box at the bottom and choose ‘<em>Next</em>’.<img alt="Fig 7 shows steps to adding access key to SCSyncUser" class="alignnone size-full wp-image-45741" height="702" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/7-2.png" width="1514" /></li> 
 <li>On the next page, choose ‘<em>Create access key</em>’.</li> 
 <li>To Retrieve the access keys, choose ‘<em>Download .csv file</em>’ to save the access key and then choose ‘<em>Done</em>’.<img alt="Fig 8 shows how to view and download secret access key" class="size-full wp-image-45742 alignnone" height="586" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/8-1.png" width="2062" /></li> 
</ol> 
<p><strong>Step 3.2</strong>: Create User 2</p> 
<ol> 
 <li>Create the second user <em>SCEndUser</em>, assign Permissions and create Access key (Programmatic Access) as shown in the screenshots.</li> 
 <li>Select “<em>Users</em>” from the left-hand menu and then choose&nbsp; “<em>Add users</em>” button.</li> 
 <li>Enter “<em>SCEndUser</em>” for the user name. Since this user will only require programmatic access, leave the box unchecked for access to the AWS Management Console.<img alt="Fig 9 shows the creation of user called SCEndUser" class="alignnone size-full wp-image-45743" height="382" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/9-1.png" width="927" /></li> 
 <li>Choose ‘<em>Next</em>’.</li> 
 <li>Select the permissions option ‘<em>Attach policies directly</em>’ and search for the policy <em>AWSServiceCatalogEndUserFullAccess</em>”. Select the check-box against the policy name.<img alt="Fig 10 shows how to attach a policy to SCEndUser" class="alignnone size-full wp-image-45744" height="718" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/10-1.png" width="2439" /></li> 
 <li>Choose ‘<em>Next</em>’, Review the details and select ‘<em>Create User</em>’ to complete the User Creation.</li> 
 <li>To create access key for this user, select “<em>Users</em>” from the left-hand menu and then choose <em>SCEndUser</em>.</li> 
 <li>Select the ‘<em>Security credentials</em>’ tab.&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<img alt="Fig 11 shows steps to adding access key to SCEndUser" class="alignnone size-full wp-image-45745" height="394" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/11-1.png" width="983" /></li> 
 <li>Scroll down to the section for ‘<em>Access keys</em>’ and choose ‘<em>Create access key</em>’.<img alt="Fig 12 shows steps to adding access key to SCEndUser" class="alignnone size-full wp-image-45746" height="206" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/12.png" width="1208" /></li> 
 <li>Select the radio button option for ‘<em>Third-party service</em>’, check the box at the bottom and choose ‘<em>Next</em>’.<img alt="Fig 13 shows steps to adding access key to SCEndUser" class="alignnone size-full wp-image-45772" height="578" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/13-1.png" width="1137" /></li> 
 <li>On the next page, choose ‘<em>Create access key</em>’.</li> 
 <li>To Retrieve the access keys, select ‘<em>Download .csv file</em>’ to save the access key and then choose ‘<em>Done</em>’.<img alt="Fig 14 shows how to view and download the access key to SCEndUser" class="alignnone size-full wp-image-45748" height="651" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/14.png" width="1983" />It is best practice to rotate AWS Access keys periodically and <a href="https://docs.aws.amazon.com/smc/latest/ag/sn-sync-keys.html">sync updated keys programmatically in ServiceNow</a>.</li> 
</ol> 
<p><strong>Step 4: Create IAM Policies</strong></p> 
<p>Before creating a launch constraint to which we attach an IAM role that Service Catalog assumes when an end user launches a product, the following policies with necessary permissions need to be created and subsequently attached to the IAM Role.</p> 
<p>Navigate to the Identity and Access Management (IAM) service.</p> 
<p><strong>Step 4.1</strong>: Create a policy called <em>Demo-ControlTower-FullAccess</em>. This policy allows full access to the AWS Control Tower Service to provision new accounts.</p> 
<ol> 
 <li>In the left navigation pane, choose “<em>Policies</em>” and then select the “<em>Create policy</em>” button.</li> 
 <li>On the “<em>Create policy</em>” page, select the “JSON” and replace the default policy document with the following: <pre><code class="lang-json">{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "controltower:*",
            "Resource": "*"
        }
    ]
}</code></pre> </li> 
 <li>When you are finished, choose <em>Next: Tags</em> and then choose <em>Next: Review</em> on the following page.</li> 
 <li>On the <em>Review policy</em> page, type the Name “<em>Demo-ControlTower-FullAccess</em>”.</li> 
 <li>Review your policy and choose “<em>Create policy</em>” to save it.</li> 
</ol> 
<p><strong>Step 4.2</strong>: Create another policy called <em>Demo-SSO-FullAccess</em>. This policy allows access to AWS IAM Identify Center (successor to AWS Single Sing-on) service to create the IAM Identity Center Users (for the accounts provisioned using AWS Service Catalog – AWS Control Tower Account Factory Portfolio).</p> 
<ol> 
 <li>In the left navigation pane, choose “<em>Policies</em>” and then choose the “<em>Create policy</em>” button.</li> 
 <li>On the “<em>Create policy</em>” page, select the “JSON” and replace the default policy document with the following: <pre><code class="lang-json">{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "sso-directory:*",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "sso:*",
            "Resource": "*"
        }
    ]
}</code></pre> </li> 
 <li>When you are finished, choose <em>Next: Tags</em> and then choose <em>Next: Review</em> on the following page.</li> 
 <li>On the Review policy page, type the Name “<em>Demo-SSO-FullAccess</em>”.</li> 
 <li>Review your policy and choose “<em>Create policy</em>” to save it.</li> 
</ol> 
<p><strong>Step 5: Create IAM Role</strong></p> 
<ol> 
 <li>Navigate to the Identity and Access Management (IAM) service.</li> 
 <li>In the left navigation menu, select “<em>Roles</em>“.</li> 
 <li>Choose the “<em>Create role</em>” button.</li> 
 <li>Select “<em>AWS service</em>” for the Trusted entity type and “<em>Service Catalog</em>” for the service.&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;<img alt="Fig 15 shows how to create a role for Service Catalog service" class="alignnone size-full wp-image-45749" height="661" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/15.png" width="826" /></li> 
 <li>When you are finished, choose <em>Next</em>.</li> 
 <li>Select the following permission policies to attach to the role: <pre><em><code class="lang-text">AWSCloudFormationFullAccess AmazonS3FullAccess Demo-SSO-FullAccess Demo-ControlTower-FullAccess </code></em></pre> </li> 
 <li>When you are finished, choose <em>Next</em>.</li> 
 <li>Enter the name “<em>Demo-Role-For-LaunchConstraint</em>” for the role and choose the “<em>Create role</em>” button.</li> 
</ol> 
<p><strong>Step 6: Create Service Catalog Launch Constraint</strong></p> 
<ol> 
 <li>Navigate to the <a href="https://console.aws.amazon.com/servicecatalog/home"><em>Service Catalog</em></a> service.</li> 
 <li>In the left navigation menu, select “<em>Portfolios</em>” and select the portfolio named ‘<em>AWS Control Tower Account Factory Portfolio</em>’.<img alt="Fig 16 shows steps to creating Service Catalog Launch Constraint" class="alignnone size-full wp-image-45750" height="444" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/16.png" width="824" /></li> 
 <li>On the next page, select the tab “<em>Constraints</em>” and choose the “<em>Create constraint</em>” button.<img alt="Fig 17 shows steps to creating Service Catalog Launch Constraint" class="alignnone size-full wp-image-45751" height="473" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/17.png" width="957" /></li> 
 <li>Select ‘<em>AWS Control Tower Account Factory Portfolio</em>’ for Product and ‘<em>Launch</em>’ for the Constraint type.<img alt="Fig 18 shows steps to creating Service Catalog Launch Constraint" class="alignnone size-full wp-image-45752" height="412" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/18.png" width="787" /></li> 
 <li>For the Launch Constraint Method, choose “<em>Select IAM role</em>” and select the IAM role <em>Demo-Role-For-LaunchConstraint</em> from the drop-down.<img alt="Fig 19 shows how to attach an IAM role to Service Catalog Launch Constraint" class="alignnone size-full wp-image-45753" height="559" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/19.png" width="840" /></li> 
 <li>Choose “<em>Create</em>”.</li> 
</ol> 
<p><strong>Step 7: Grant AWS Service Management Connector (SMC) Users access to portfolio</strong></p> 
<p>To grant Service Management Connector users <em>SCSyncUser </em>and <em>SCEndUser</em>:</p> 
<ol> 
 <li>Navigate to the <em>Service Catalog</em> service.</li> 
 <li>In the left navigation menu, select “<em>Portfolios</em>” and select the portfolio named ‘<em>AWS Control Tower Account Factory Portfolio</em>’.</li> 
 <li>On the next page, select the tab “<em>Access</em>” and choose “<em>Grant access</em>” button.</li> 
 <li>Select “<em>IAM Principal</em>” for the Access type and select the “<em>Users</em>” tab.</li> 
 <li>Select the users <em>SCEndUser</em> and <em>SCSyncUser</em> by checking the boxes.</li> 
 <li>Choose the “<em>Grant access</em>” button to grant access to the users.<img alt="Fig 20 shows how to Grant AWS Service Management Connector (SMC) Users access to the portfolio" class="alignnone size-full wp-image-45754" height="796" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/20.png" width="1130" /></li> 
</ol> 
<p>This concludes the configuration required on AWS.</p> 
<h2><strong>Part 2: ServiceNow Configuration</strong></h2> 
<p>This section describes how to configure core components in ServiceNow.</p> 
<p><strong>Step 1: Configuring AWS Service Management Connector</strong><br /> AWS Service Management Connector requires a ServiceNow plugin, called <em>User Criteria Scoped API (for AWS Service Catalog integration)</em>, that provides useful components to the integration features.</p> 
<p>The following configuration steps, as detailed in the <a href="https://docs.aws.amazon.com/smc/latest/ag/sn-config-core-components.html">AWS Service Management Connector</a> Administrator Guide, need to be performed:</p> 
<ol> 
 <li><a href="https://docs.aws.amazon.com/smc/latest/ag/sn-config-core-components.html#sn-activate-plugins">Activating ServiceNow plugins</a></li> 
 <li><a href="https://docs.aws.amazon.com/smc/latest/ag/sn-config-core-components.html#sn-install-connector">Installing ServiceNow Connector scoped application</a></li> 
 <li><a href="https://docs.aws.amazon.com/smc/latest/ag/sn-config-core-components.html#sn-guided-setup">Configuring Connector using Guided Setup</a></li> 
 <li><a href="https://docs.aws.amazon.com/smc/latest/ag/sn-config-core-components.html#sn-configure-connector">Platform system administrator components</a></li> 
 <li><a href="https://docs.aws.amazon.com/smc/latest/ag/sn-config-core-components.html#sn-permissions-admin">ServiceNow permissions for administrators of the Connector scoped app</a></li> 
 <li><a href="https://docs.aws.amazon.com/smc/latest/ag/sn-config-core-components.html#sn-configure-sc-connector-scoped-app">Configuring AWS Service Management Connector scoped application</a></li> 
 <li><a href="https://docs.aws.amazon.com/smc/latest/ag/sn-config-core-components.html#sn-configure-accounts">Configuring AWS accounts to synchronize in the Connector</a> 
  <ul> 
   <li>When choosing the visible AWS service integrations for this AWS account, select 
    <ul> 
     <li>‘Integrate with AWS Service Catalog (including AppRegistry)‘ check-box option from the visible AWS service integrations for this AWS account.</li> 
    </ul> </li> 
  </ul> </li> 
 <li><a href="https://docs.aws.amazon.com/smc/latest/ag/sn-config-core-components.html#validate-regions">Validating connectivity to AWS Regions</a></li> 
 <li><a href="https://docs.aws.amazon.com/smc/latest/ag/sn-config-core-components.html#manual-sync-scheduled-jobs">Manually syncing scheduled jobs</a> 
  <ul> 
   <li>Choose the following job names to execute: 
    <ul> 
     <li>Synchronize AWS Service Catalog</li> 
     <li>Synchronize changes to all AWS accounts</li> 
    </ul> </li> 
  </ul> </li> 
</ol> 
<p><strong>Step 2: Adding the My AWS Products widget to the Service Portal view</strong><br /> The widget enables users to view their AWS product requests, view outputs, and perform post-operational actions such as update and terminate.</p> 
<p>To include the <em>My AWS Products</em> widget on the Service Portal view, log in as system administrator in the ServiceNow standard user interface (Fulfiller view).</p> 
<ol> 
 <li>In the navigator panel, find <em>Service Portal.</em></li> 
 <li>Choose Service <em>Portal Configuration</em>.</li> 
 <li>Choose <em>Designer</em>.</li> 
 <li>Search for <em>Index</em> in the filter. Choose the <em>Home Page</em> box with a house image and the word Index in the lower left corner.</li> 
 <li>In the left panel in Widgets, enter <em>My AWS Products</em> in the Filter Widget.</li> 
 <li>Drag the widget to the Service Portal edit view to your desired location.</li> 
 <li>Preview your changes.</li> 
</ol> 
<h2><strong>Part 3: Provisioning an AWS Account from ServiceNow</strong></h2> 
<p>This section describes an example of how to order and provision an AWS Account from ServiceNow.</p> 
<ol> 
 <li>In the navigator panel, find Service Portal.</li> 
 <li>Choose Service Portal Home.</li> 
 <li>Choose Catalog Menu on the top</li> 
 <li>Select the Browse by Categories link</li> 
 <li>Under the Categories section on the left, expand the AWS Service Catalog to view the available AWS products.</li> 
 <li>Chooseo AWS Control Tower Account Factory Portfolio [Service Catalog] and then select AWS Control Tower Factory product to provision.<img alt="Fig 21 shows how to access AWS Control Tower Account Factory Portfolio from ServiceNow Portal" class="alignnone size-full wp-image-45755" height="510" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/21.png" width="1463" /></li> 
 <li>Enter product request details including Product Name, Parameters and Tags (if any).<img alt="Fig 22 shows steps to ordering an AWS Account from ServiceNow Portal" class="alignnone size-full wp-image-45756" height="636" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/22.png" width="2096" /></li> 
 <li>Choose <em>Order Now</em> to submit the ServiceNow request and provision the AWS Service Catalog product.<img alt="Fig 23 shows steps to ordering an AWS Account from ServiceNow Portal" class="alignnone size-full wp-image-45757" height="891" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/23.png" width="2097" /></li> 
 <li>Choose <em>Check Out</em> to submit&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; <img alt="Fig 24 shows order status acknowledging the submission." class="alignnone size-full wp-image-45758" height="326" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/24.png" width="655" /></li> 
</ol> 
<p>Once done, you will receive an order status acknowledging the submission. The request shows that the submissions and execution have been successful and outputs the information submitted to AWS.</p> 
<p><img alt="Fig 25 shows the ServiceNow Request details" class="alignnone size-full wp-image-45759" height="978" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/25.png" width="1298" /></p> 
<h2><strong>Part 4: AWS validation of self-serve account provisioned from ServiceNow</strong></h2> 
<p>In your AWS Console, navigate to AWS Control Tower</p> 
<ol> 
 <li>Choose <em>Organization</em> on the left panel.</li> 
 <li>Expand the <em>Sandbox</em> Organizational Unit (OU) on the right to verify that the account provisioned from ServiceNow is Enrolled.</li> 
</ol> 
<p><img alt="Fig 26 shows the Account provisioned in AWS Control Tower" class="alignnone size-full wp-image-45760" height="292" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/24/26.png" width="1041" /></p> 
<p>This completes the final task in this hands-on session.</p> 
<h2><strong>Part 5: Clean Up</strong></h2> 
<p>To clean up AWS Configurations:</p> 
<ol> 
 <li>Remove access to AWS Control Tower Account Factory Portfolio for users: <em>SCSyncUser</em> and <em>SCEndUser</em>.</li> 
 <li>Delete the AWS Control Tower Account Factory Portfolio Launch Constraint.</li> 
 <li>Delete IAM Role <em>Demo-Role-For-LaunchConstraint.</em></li> 
 <li>Delete the two IAM Policies: 
  <ul> 
   <li><em>Demo-ControlTower-FullAccess</em></li> 
   <li><em>Demo-SSO-FullAccess</em></li> 
  </ul> </li> 
 <li>Delete the two IAM Users: 
  <ul> 
   <li><em>SCSyncUser</em></li> 
   <li><em>SCEndUser</em></li> 
  </ul> </li> 
</ol> 
<p>To clean up in ServiceNow, <a href="https://developer.servicenow.com/dev.do#!/guides/utah/developer-program/pdi-guide/managing-your-pdi#releasing-your-pdi">release the Personal Developer Instance</a> provisioned for this Lab by following instructions in ServiceNow Documentation.</p> 
<h2><strong> Conclusion</strong></h2> 
<p>Customers using ServiceNow as their ITSM solution can leverage the AWS Service Management Connector to self-serve the vending of AWS Accounts natively from ServiceNow. In this blog post, we’ve shown how to implement steps to integrate AWS Service Catalog (AWS Control Tower Account Factory Portfolio) with ServiceNow ITSM using AWS Service Management Connector and raise a change request to provision an AWS Account through the ServiceNow Portal in a secure and standardized fashion. In doing so, customers can accelerate migration and AWS adoption at scale through oversight and governance in their declared operational tooling and system of record.</p> 
<h2><strong>About the Author</strong></h2> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="aligncenter size-full wp-image-11636" height="160" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/10/25/ProfilePic.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Siddhartha Angara</h3> 
  <p>Sid is a Senior Solutions Architect at Amazon Web Services. He helps enterprise customers design and build well-architected solutions in the cloud and accelerate cloud adoption. He enjoys playing the guitar, reading and family time!</p> 
 </div> 
</footer>
