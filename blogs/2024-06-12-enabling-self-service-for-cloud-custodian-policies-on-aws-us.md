---
title: "Enabling Self Service for Cloud Custodian policies on AWS using AWS Service Catalog"
url: "https://aws.amazon.com/blogs/mt/enabling-self-service-for-cloud-custodian-policies-on-aws-using-aws-service-catalog/"
date: "Wed, 12 Jun 2024 14:08:41 +0000"
author: "Mokshith Kumar"
feed_url: "https://aws.amazon.com/blogs/mt/tag/aws-service-catalog/feed/"
---
<p>Customers are increasingly seeking tools and solutions that can help them achieve their desired outcomes more efficiently and effectively. In the context of cloud management, the need for self-service capabilities has become more pronounced as organizations strive to optimize their cloud resources, improve security, and enhance their overall cloud operations. <a href="https://aws.amazon.com/servicecatalog/">AWS Service Catalog</a> offers the self-service capability to centrally manage the cloud resources to achieve governance at scale.</p> 
<p><a href="https://cloudcustodian.io/">Cloud custodian</a> is a cloud agnostic open-source tool that is used by organizations across industry verticals to manage their cloud environments by helping ensure compliance to security policies, tag policies, garbage collection of unused resources, and cost management. Cloud custodian works by defining policies in a YAML file and running the defined policies against AWS account.</p> 
<p>Generally, Enterprise AWS customers manage these custodian policies centrally and will leverage CI/CD pipeline to <a href="https://aws.amazon.com/blogs/opensource/continuous-deployment-of-cloud-custodian-to-aws-control-tower/">roll out the custodian policies to AWS accounts</a>. This allows customer to ensure AWS accounts provisioned as part of a multi-account environment meet their organization’s compliance requirements. Sometimes customers are looking for ways to make the custodian policies as a self-service tool which will allow application team to choose the desired policy which will make all of their applications compliant and standard.</p> 
<p>In this post, we show you how to leverage AWS Service Catalog to manage and enable the self-service capability for the custodian policies in a multi-account environment. This will help you to start thinking on defining compliance requirements for your applications and also accelerate provisioning by leveraging the self-service tool.</p> 
<h2><strong>Overview of Solution</strong></h2> 
<p>In this solution an <a href="https://aws.amazon.com/codecommit/">AWS CodeCommit</a> repository is used to host the Cloud custodian policy file and the file containing IAM role permissions necessary for the custodian policy. From the below Figure 1, you can see that, once the above two files are added to the repository by the Portfolio administrators, (1) an <a href="https://aws.amazon.com/codepipeline/">AWS CodePipeline</a> gets invoked which will (2) validate the custodian policy, (3) convert the custodian policy to <a href="https://aws.amazon.com/serverless/sam/">SAM</a> template and (4) uploads the template file to S3 bucket. (5) Then an <a href="https://aws.amazon.com/eventbridge/">AWS EventBridge</a> rule triggers the lambda function which creates the AWS Service Catalog product for Cloud custodian policies. The Cloud custodian Service Catalog tool provisions the custodian policies and necessary IAM roles required for those policies.</p> 
<p style="text-align: center;"><img alt="Figure demonstrates architecture to provisioning Cloud custodian policies as Service Catalog product that is part of the Custodian Portfolio which shared with other account to enable Self Service capability" class="size-full wp-image-52614 aligncenter" height="940" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/05/30/Screenshot-2024-05-29-at-5.37.24 PM.png" width="2380" /></p> 
<p style="text-align: center;">Figure 1: The Solution architecture for enabling Self Service for custodian policies on AWS</p> 
<h2><strong>Walkthrough</strong></h2> 
<p>The infrastructure and automation code for this solution can be found in the following GitHub <a href="https://github.com/aws-samples/self-service-cloud-custodian-policies-using-aws-service-catalog/">repository custodian service catalog pipeline repository</a> and below are steps we will be implementing as part of this solution:</p> 
<ol> 
 <li>Deploy the Infrastructure for the Service Catalog automation in Service Catalog Hub account.</li> 
 <li>Adding the custodian policy to the AWS CodeCommit repository.</li> 
 <li>Validate the Service Catalog product creation.</li> 
 <li>Updating the custodian policy.</li> 
 <li>Validate the provisioning and testing of the custodian policy Service Catalog product in the developer account.</li> 
</ol> 
<h2><strong>Prerequisites</strong></h2> 
<p>For this walk-through, you should have the following prerequisites:</p> 
<ol> 
 <li>Two AWS accounts as part of <a href="https://aws.amazon.com/organizations/">AWS Organizations</a> with administrator privileges for the services used in this solution. 
  <ol> 
   <li>&nbsp;Service Catalog Hub Account – Used to provision the custodian policy as a Service Catalog product in a Portfolio.</li> 
   <li>Developer account – End-user to launch the Service Catalog product.</li> 
  </ol> </li> 
 <li>Ensure Trusted Access is Enabled for Service Catalog. Review the <a href="https://docs.aws.amazon.com/organizations/latest/userguide/orgs_integrate_services.html#orgs_how-to-enable-disable-trusted-access">steps</a> to enable trusted access for Service Catalog.</li> 
 <li>Ensure the Service Catalog Hub Account is enabled as Delegated administrator account for AWS Service Catalog in the Organizations Management Account. Run the below CLI command to register an account as delegated admin from the Organizations Management Account. 
  <div class="hide-language"> 
   <pre><code class="lang-bash">aws organizations register-delegated-administrator \--account-id service-catalog-account-id \--service-principal servicecatalog.amazonaws.com</code></pre> 
  </div> </li> 
 <li>Run the below CLI command to add a custom policy that allows delegated admin account to query accounts within the Organization (replace {service catalog account} with the 12-digit Account ID). The command should be run on the Organization’s Management Account. <p style="text-align: left;"><strong>This command requires CLI 2.10+</strong></p> 
  <div class="hide-language"> 
   <pre><code class="lang-bash">aws organizations put-resource-policy \ --content '{"Version": "2012-10-17","Statement": [{"Sid": "Statement","Effect": "Allow","Principal": {"AWS": "arn:aws:iam::'"${service_catalog_account_ID}"':root"},"Action": ["organizations:ListDelegatedAdministrators","organizations:ListParents","organizations:ListChildren","organizations:DescribeAccount"],"Resource": "*"}]}'
</code></pre> 
  </div> </li> 
 <li>The solution requires Service catalog portfolio provisioned in the Service Catalog delegated admin account with below features: 
  <ol> 
   <li>Portfolio sharing enabled for entire AWS Organization or Specific AWS Organization Unit to which the Service Catalog product to be made available.</li> 
   <li>Portfolio access to be provided to necessary IAM identity to launch the Service Catalog product.</li> 
  </ol> </li> 
 <li>An IAM Role with launch constraint for CloudFormation product permissions, and permissions to provision custodian resources should exist in each account to which the Service Catalog product will be made available. This role which will be associated to the Service Catalog product as a <a href="https://docs.aws.amazon.com/servicecatalog/latest/adminguide/constraints-launch.html">Launch Constraint</a> role.</li> 
</ol> 
<p>For this solution walkthrough, we are provisioning the <a href="https://github.com/aws-samples/self-service-cloud-custodian-policies-using-aws-service-catalog/blob/main/cloudformation_templates/sc_custodian_portfolio.yml">portfolio</a> and <a href="https://github.com/aws-samples/self-service-cloud-custodian-policies-using-aws-service-catalog/blob/main/cloudformation_templates/sc_launch_constraint_role.yml">launch constrain role</a> from the templates present in the repository. The Portfolio provisioned as <a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacks.html">CloudFormation stack</a> in the Service Catalog Hub Account is shared across the AWS Organization and is provided access for the <a href="https://aws.amazon.com/iam/identity-center/">AWS IAM Identity Center</a> Identity “AWSAdministratorAccess”. The Launch role is provisioned as a <a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-orgs-manage-auto-deployment.html">Service Managed CloudFormation Stack Set</a> to the Service Catalog Hub Account and Developer Account. This role has the permissions to provision Lambda and EventBridge rule – which are the resources that will be provisioned part of the example custodian policy in this blog.</p> 
<h2><strong>Deployment Steps</strong></h2> 
<h3><strong>Step 1: Deploy the Infrastructure for the Service Catalog Automation</strong></h3> 
<p>In this step, you will provision the infrastructure for the service catalog automation as a CloudFormation stack in the Service Catalog Hub Account which has the Service Catalog portfolio provisioned.</p> 
<ol> 
 <li>Log in to the Service Catalog Hub account and select the AWS Region in which custodian Hub Portfolio is provisioned.</li> 
 <li>Navigate to <a href="https://console.aws.amazon.com/cloudformation/home#/stacks/create">CloudFormation console</a> and create the stack using <a href="https://github.com/aws-samples/self-service-cloud-custodian-policies-using-aws-service-catalog/blob/main/cloudformation_templates/custodian_automation_pipeline.yml">custodian_automation_pipeline.yaml</a> template.</li> 
 <li>Enter the <strong>Stack name</strong>. In our example, we use custodian-Service-Catalog-Infra. Under the <strong>Parameters</strong> section, enter values for the following parameters: 
  <ol> 
   <li><strong>pPortfolioProviderName</strong> – The Team that is offering the custodian product to the end users in the Organization.</li> 
   <li><strong>pSCTemplateS3Bucket</strong> – The s3 Bucket that will hold the CloudFormation template for the custodian Service Catalog product.</li> 
   <li><strong>pSCTemplateS3LoggingBucket</strong> – The S3 Bucket that will be the logging bucket for the custodian policy bucket.</li> 
   <li><strong>pSCLaunchConstraintRoleName</strong> – The Launch Constraint role for the provisioning of Service Catalog products. <strong>Note:</strong> This role should be present in Service Catalog Hub Account and the Developer Account in which the product will be provisioned.</li> 
   <li><strong>pSCPortfolioName</strong> – The Portfolio that will hold the custodian Service Catalog products.</li> 
   <li><strong>pPipelineName</strong> – Name of the pipeline that manages the product deployment of custodian policies.</li> 
   <li><strong>pCodeCommitRepoName</strong> – Name of the AWS CodeCommit repository that will contain the custodian policy and permission files for the role.</li> 
   <li><strong>pCodepipelineArtifactBucket</strong> – The S3 bucket that will hold the CodePipeline artifacts.</li> 
   <li><strong>pOrganizationID</strong> – The Organization ID to allow GetObject on template S3 Bucket for custodian policy.<img alt="Figure show CloudFormation template parameters to be passed as part of the Infrastructure deployment" class="wp-image-52409 size-full aligncenter" height="858" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/05/21/Fig_2.png" width="1475" /> <p style="text-align: center;">Figure 2: The Pipeline Infrastructure CloudFormation Parameters</p> </li> 
  </ol> </li> 
 <li>On the <strong>Review</strong> page, validate the parameters and check the box <strong>I acknowledge that AWS CloudFormation might create IAM</strong>. Then select <strong>Submit</strong>.</li> 
 <li>When the CloudFormation stack status changes to CREATE_COMPLETE, navigate to the <strong>Resources</strong> tab and validate that all the resources have been created successfully.</li> 
</ol> 
<p><strong>NOTE:</strong> If you’re leveraging the existing portfolio, the Lambda Role “ExecutionRoleForCustodianProductCreation” should be provided access to the portfolio.</p> 
<h3>Step 2: Adding the Custodian policy to the AWS CodeCommit Repository</h3> 
<p>In this step, you will create a new Cloud custodian policy that will be available to end users via AWS Service Catalog. For this solution walkthrough, we will provision a custodian policy that terminates the EC2 instance that are provisioned with un-encrypted EBS volumes.</p> 
<p>NOTE: The solution doesn’t support <a href="https://cloudcustodian.io/docs/aws/aws-modes.html">custodian policy in pull mode</a>.</p> 
<ol> 
 <li> 
  <ol> 
   <li>Clone the GitHub repository by running the following <a href="https://aws.amazon.com/cli/">AWS Command Line Interface</a> (AWS CLI) commands from a terminal window. 
    <div class="hide-language"> 
     <pre><code class="lang-bash">git clone https://github.com/aws-samples/self-service-cloud-custodian-policies-using-aws-service-catalog.git custodian-repo
cd custodian-repo</code></pre> 
    </div> </li> 
   <li>Clone and copy all the <strong>codecommit_files</strong> from the <a href="https://github.com/aws-samples/self-service-cloud-custodian-policies-using-aws-service-catalog/tree/main/codecommit_files">custodian service catalog pipeline repository</a> to the custodian-repo created in the previous step.</li> 
  </ol> </li> 
</ol> 
<p><img alt="Figure show the file structure once CodeCommit files from the provided GitHub repository are added to the custodian CodeCommit repository locally" class="aligncenter wp-image-52196" height="312" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/05/14/Picture3-6.png" width="535" /></p> 
<p style="text-align: center;">Figure 3: The Folder Structure after copying the files.</p> 
<ol> 
 <li> 
  <ol start="3"> 
   <li>Create a sub folder in the <strong>cc_policies</strong> folder for each custodian policy you will be adding to the repository. Each folder is a separate Cloud custodian AWS Service Catalog product and folder name will be used for the Product name. The folder should contain custodian policy YAML file and permissions.yml that will contain permissions for the Lambda Role. For this walkthrough, we will create a custodian policy that will prevent the provisioning of EC2 instance with unencrypted EBS volumes. The files for provisioning the custodian policy “encrypted-volumes” is provided in the <a href="https://github.com/aws-samples/self-service-cloud-custodian-policies-using-aws-service-catalog/tree/main/example_custodian_policy">repository</a>.</li> 
  </ol> </li> 
</ol> 
<p><strong>Custodian policy:</strong></p> 
<div class="hide-language"> 
 <pre><code class="lang-yaml">policies:
- name: ec2-require-encrypted-volumes
  resource: aws.ec2
  description: Provision a lambda and cloud watch event target that looks at all new instances and terminates those with unencrypted volumes
  mode:
 &nbsp;&nbsp; type: cloudtrail
 &nbsp;&nbsp; role: custodian-lambda-role-encrypted-volumes
 &nbsp;&nbsp; events:
 &nbsp;&nbsp;&nbsp;&nbsp; - RunInstances
  filters:
 &nbsp;&nbsp; - type: ebs
 &nbsp;&nbsp;&nbsp;&nbsp; key: Encrypted
 &nbsp;&nbsp;&nbsp;&nbsp; value: false
  actions:
&nbsp;&nbsp;&nbsp; - terminate</code></pre> 
</div> 
<p><strong>Permissions:</strong></p> 
<div class="hide-language"> 
 <pre><code class="lang-yaml">PolicyName: "encrypted-volumes"
Permissions:
 Action:
 "ec2:Describe*"
- "ec2:CreateTags"
- "ec2:DeleteTags"
- "ec2:TerminateInstances"
Resource:
- "*"</code></pre> 
</div> 
<p><strong>NOTE:</strong> The Role used in the custodian policy should be in the format <code>custodian-lambda-role-&lt; PolicyName&gt;</code></p> 
<p style="text-align: center;"><img alt="Figure highlights the Role to be used as part of custodian policy" class="wp-image-52389 size-medium aligncenter" height="300" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/05/21/Picture4-6-253x300.png" width="253" /></p> 
<p style="text-align: center;">Figure 4: The Folder Structure after adding the custodian policy.</p> 
<ol> 
 <li> 
  <ol start="4"> 
   <li>Add the code to the repository by running the following commands.</li> 
  </ol> </li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-bash">git add .
git commit -m "Adding the encrypted volumes custodian policy with the initial buildspec files"
git push</code></pre> 
</div> 
<ol class="alignnone size-full wp-image-52594" start="5&lt;img src="> 
 <li>Once the changes are pushed, deployment takes about 3-5 minutes because of the CodePipelines and Lambda automation for Service Catalog product creation. Check the progress on the AWS CodePipeline Console on your Service Catalog Hub Account.</li> 
</ol> 
<h3><strong>Step 3: Validate the Service Catalog product Creation</strong></h3> 
<p>Once the product creation CodePipeline is completed successfully, follow the below steps to validate the Service Catalog product creation.</p> 
<ol> 
 <li> 
  <ol> 
   <li> 
    <ol> 
     <li>Navigate to Service Catalog service in the Service Catalog Hub Account.</li> 
     <li>Select the Portfolio which was passed as the parameter to the service catalog automation CloudFormation stack in step 1.</li> 
    </ol> </li> 
  </ol> <p style="text-align: center;"><img alt="Figure shows the Service Catalog Product creation of the custodian policy" class="size-full wp-image-52394 aligncenter" height="675" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/05/21/Picture5-6.png" width="1430" />Figure 5: The Service Catalog product created successfully</p> 
  <ol start="3"> 
   <li>Navigate to S3 and review the metadata of the policy file &lt;policy-name&gt;.yml – there are metadata for Version and policy name. In this walkthrough, the S3 object encrypted-volumes.yml will have Version with Value “1” and policy with value “encrypted-volumes”</li> 
  </ol> </li> 
</ol> 
<p style="text-align: center;"><img alt="Figure shows the S3 Object metadata version of the custodian policy that is provisioned as Service Catalog Product" class="size-full wp-image-52395 aligncenter" height="485" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/05/21/Picture6-4.png" width="1430" />Figure 6: The S3 Object metadata for the Service Catalog product Template File</p> 
<h3>Step 4: Updating the Custodian policy</h3> 
<p>In this step, we will update the existing custodian policy and validate the version update of the Service Catalog product.</p> 
<ol> 
 <li> 
  <ol> 
   <li> 
    <ol> 
     <li>We will add “auto-tag aws userName on resources” custodian policy to the existing policy Custodian policy.</li> 
    </ol> </li> 
  </ol> 
  <div class="hide-language"> 
   <pre><code class="lang-yaml">- name: ec2-auto-tag-user
  resource: ec2
  mode:
 &nbsp;&nbsp; type: cloudtrail
 &nbsp;&nbsp; role: custodian-lambda-role-encrypted-volumes
 &nbsp;&nbsp; events:
 &nbsp;&nbsp;&nbsp;&nbsp; - RunInstances
  filters:
 &nbsp;&nbsp; - tag:CreatorName: absent
  actions:
 &nbsp;&nbsp; - type: auto-tag-user
 &nbsp;&nbsp;&nbsp;&nbsp; tag: CreatorName
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; principal_id_tag: CreatorId</code></pre> 
  </div> 
  <ol> 
   <li> 
    <ol start="2"> 
     <li>Commit and push the changes to code commit repository by running the following commands.</li> 
    </ol> </li> 
  </ol> 
  <div class="hide-language"> 
   <pre><code class="lang-bash">git commit -a -m "Updated the encrypted volumes custodian policy with the auto tag custodian policy"
git push</code></pre> 
  </div> 
  <ol> 
   <li> 
    <ol start="3"> 
     <li>Once the Service Catalog product pipeline is completed successfully, navigate to Service Catalog console and you will notice the previous version is Deprecated and the new version is made available. Note: Users cannot launch new provisioned products using a deprecated product version. If a provisioned product launched previously uses a now deprecated version, users can only update that provisioned product using the existing version or a new version.</li> 
    </ol> </li> 
  </ol> <p><img alt="Figure shows the Service Catalog product version update of the custodian policy" class="wp-image-52417 size-full aligncenter" height="720" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/05/21/Fig_7.png" width="1589" /></p> <p style="text-align: center;">Figure 7: The Service Catalog product version update</p> 
  <ol> 
   <li> 
    <ol start="4"> 
     <li>Navigate to S3 and review the metadata of the S3 object encrypted-volumes.yml. The Version is updated to “2” as well.<img alt="Figure shows the updated S3 Object metadata version of the custodian policy that is provisioned as Service Catalog product" class="wp-image-52418 size-full aligncenter" height="493" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/05/21/Fig_8.png" width="1512" /></li> 
    </ol> </li> 
  </ol> <p style="text-align: center;">Figure 8: The S3 Object metadata version update for the Service Catalog product template file</p> </li> 
</ol> 
<h3>Step 5: Validate the provisioning and testing of the Custodian Policy Service Catalog product in the Developer Account.</h3> 
<p>In this step, will validate the custodian policy by provisioning the Service Catalog product from the developer account.</p> 
<ol> 
 <li> 
  <ol> 
   <li> 
    <ol> 
     <li>Login the Developer account which has the Service Catalog Portfolio shared with the IAM Identity that’s provided access.</li> 
     <li>Navigate to Service Catalog console, select the product “custodian-encrypted-volumes” and Launch Product.</li> 
    </ol> </li> 
  </ol> <p style="text-align: center;"><img alt="Figure demonstrates the step to provision the shared Service Catalog product from the developer account." class="wp-image-52419 size-full aligncenter" height="753" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/05/21/Fig_9.png" width="1025" />Figure 9: The Service Catalog product provisioning in the Developer Account</p> 
  <ol> 
   <li> 
    <ol start="3"> 
     <li>Once the Product provisioned successfully, to test the custodian product, we will launch an EC2 instance which is unencrypted EBS volume.</li> 
    </ol> </li> 
  </ol> <p style="text-align: center;"><img alt="Figure demonstrates the step to provision an EC2 instance with Unencrypted EBS Volume as a part of validation of the custodian policy which is provisioned Service Catalog product" class="wp-image-52399 size-large aligncenter" height="516" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/05/21/Picture10-3-1024x516.png" width="1024" />Figure 10: Provisioning an EC2 Instance with Unencrypted EBS Volume</p> 
  <ol start="4"> 
   <li>&nbsp;The instance will be terminated at the time of provisioning due the custodian policy Lambda. Note: You could also configure the Cloud custodian policy to alternatively send <a href="https://aws.amazon.com/sns/">SNS</a> notifications to your subscribed topic instead of taking a remediating action such as shutting down the instance.</li> 
  </ol> </li> 
</ol> 
<p style="text-align: center;"><img alt="Figure shows the termination of an EC2 instance which was provisioned with Unencrypted EBS Volume by the lambda function provisioned as part of the custodian policy " class="size-full wp-image-52400 aligncenter" height="378" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/05/21/Picture11-3.png" width="1430" />Figure 11: Custodian automation terminating the provisioned unencrypted instance</p> 
<h2>Cleaning up</h2> 
<p>If you are following along the blog and would want to terminate the resources to avoid incurring future charges, follow the below steps:</p> 
<ol> 
 <li> 
  <ol> 
   <li>Login to the Developer Account to <a href="https://docs.aws.amazon.com/servicecatalog/latest/userguide/enduser-delete.html">delete the provisioned product</a>.</li> 
   <li>Login to the Service Catalog Hub Account and terminate the product “custodian-encrypted-volumes” following the <a href="https://docs.aws.amazon.com/servicecatalog/latest/adminguide/productmgmt-delete.html">deleting products</a> steps.</li> 
   <li>Empty the object from the custodian S3 bucket, Logging bucket and CodePipeline bucket prior to <a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-delete-stack.html">deleting the stack</a> which provisioned the custodian automation pipeline.</li> 
   <li>If you have provisioned the Launch Constraint role as part of this blog, <a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stackinstances-delete.html">delete the stack instances</a>, then <a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-delete-stack.html">delete the StackSets</a> from the AWS CloudFormation console.</li> 
   <li>If you have provisioned the portfolio as part of the blog, first remove the portfolio share of the portfolio and then delete the CloudFormation stack.</li> 
  </ol> </li> 
</ol> 
<h2>Conclusion</h2> 
<p>Cloud custodian enables users to keep their AWS environment secure and well managed by defining policies. One of the key benefits of self-service for defining compliance requirements is the ability to empower users and teams to take ownership of their cloud resources. In this blog, we showed you how to enable Self-Service capability for custodian policy leveraging Service Catalog in the multi-account environment. We also walked through deploying a sample custodian policy as Service Catalog product in the Service Catalog Hub Account and provisioning the custodian policy as Service Catalog product from the developer account.</p> 
<p>In this post, we deployed couple of example <a href="https://cloudcustodian.io/docs/aws/gettingstarted.html">custodian policies</a> as single Service Catalog product but a Service Catalog Portfolio can hold more than Service Catalog product. Review the custodian policies and enable more than one policy as part of the portfolio as separate Service Catalog product based on your organization requirements. If you need help in the design and implementation of such solution, feel free to reach out to us or your local <a href="https://aws.amazon.com/professional-services/">AWS Professional Services team</a>. We hope that you’ve found this post informative and we look forward to hearing how you use this approach.</p>
