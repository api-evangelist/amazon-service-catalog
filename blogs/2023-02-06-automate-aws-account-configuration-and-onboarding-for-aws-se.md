---
title: "Automate AWS Account configuration and onboarding for AWS Service Management Connector for ServiceNow"
url: "https://aws.amazon.com/blogs/mt/automate-aws-account-configuration-and-onboarding-for-aws-service-management-connector-for-servicenow/"
date: "Mon, 06 Feb 2023 14:11:03 +0000"
author: "Sivasankar Achin Jayapal"
feed_url: "https://aws.amazon.com/blogs/mt/tag/aws-service-catalog/feed/"
---
<p>Many enterprises use <a href="https://www.servicenow.com/">ServiceNow</a> to support their IT Service Management (ITSM) processes.&nbsp; These enterprises are looking for ways to manage and integrate their AWS cloud operations with their existing ServiceNow deployments.&nbsp; AWS provides the <a href="https://docs.aws.amazon.com/smc/latest/ag/sn-what-is.html">AWS Service Management Connector (SMC) for ServiceNow</a> to enable users to provision, manage, and operate AWS resources natively through ServiceNow.</p> 
<p>To setup the SMC, customers are required to configure <a href="https://aws.amazon.com/iam/?nc2=type_a">AWS Identity and Access Management (IAM)</a> user credentials for all required AWS accounts within their <a href="https://aws.amazon.com/organizations/">AWS Organizations</a> deployment. This requires customers to share credentials with their cloud operations team and their ServiceNow team for manual entry into ServiceNow.&nbsp; The SMC provides integrations with multiple AWS services and these must also be configured per <a href="https://docs.aws.amazon.com/smc/latest/ag/sn-config-core-components.html#sn-configure-accounts">account within ServiceNow</a> and the AWS account.&nbsp; The IAM user access keys configured for every account also needed to be <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html#Using_RotateAccessKey">rotated</a> on a defined schedule and manually updated in SMC to comply with security best practices.</p> 
<p>Customers may also utilize multiple ServiceNow instances for development, test, and production that need to be integrated and configured with SMC. This blog provides a solution to automate the AWS account onboarding and configuration process for SMC.</p> 
<p>The source code for the solution is available <a href="https://github.com/aws-samples/aws-service-management-connector">here</a>.</p> 
<h2>Solution overview</h2> 
<p>This solution enables you to automatically configure new and existing AWS accounts with ServiceNow instances. The solution automates the IAM credential configuration and management required by the SMC across multiple AWS accounts and regions. &nbsp;IAM user access key rotation and update within SMC can also be automated on a schedule of your choice.</p> 
<p>Using this solution, SMC integrations can be selectively enabled / disabled consistently across multiple accounts.</p> 
<h2>Services used</h2> 
<p>The following services are used within this post:</p> 
<ol> 
 <li><a href="https://aws.amazon.com/organizations/">AWS Organizations</a> – The solution reads AWS account names from Organizations to name each corresponding configuration within the SMC in ServiceNow.</li> 
 <li><a href="https://aws.amazon.com/cloudformation/">AWS CloudFormation</a> – <a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-concepts.html#stacksets-concepts-stackset-permission-models">Service-managed StackSets</a> are used to automatically deploy and update the solution across AWS accounts within the Organizational Units (OUs) that you specify. A CloudFormation custom resource is used to manage the solution lifecycle for each account.</li> 
 <li><a href="https://aws.amazon.com/lambda/">AWS Lambda</a> – Lambda implements the automation logic for the solution as a CloudFormation custom resource. The code is written in Python.</li> 
 <li><a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html">AWS Systems Manager Parameter Store</a> – ServiceNow credentials are stored within AWS Systems Manager Parameter Store for each ServiceNow instance. These credentials are read by the Lambda function to make authenticated calls with ServiceNow to automatically configure the SMC.</li> 
</ol> 
<h3>Architecture diagram</h3> 
<div class="wp-caption aligncenter" id="attachment_35796" style="width: 710px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/12/28/cloudops_942_1.png"><img alt="Figure 1. Architecture diagram describing how IAM credentials are created in individual workload accounts and the Lambda invocation to create the AWS account entry in ServiceNow." class="wp-image-35796" height="495" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/02/03/solution3.png" width="700" /></a>
 <p class="wp-caption-text" id="caption-attachment-35796">Figure 1. Architecture diagram describing how IAM credentials are created in individual workload accounts and the Lambda invocation to create the AWS account entry in ServiceNow</p>
</div> 
<h2>CloudFormation service-managed StackSets</h2> 
<p>CloudFormation service-managed StackSets are used by the solution to deploy the IAM Users, access keys, and additional resources required by the SMC for the target OUs that you want to enable. Automatic deployments are used by the solution to automate the process for new accounts. Service-managed StackSets, along with automatic deployments, automate the solution account management when accounts are added to a target Organization or OU, removed from a target Organization or OU, or moved between target OUs.</p> 
<p>Before you create a StackSet with service-managed permissions, you must first complete the following tasks:</p> 
<p><a href="https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_org_support-all-features.html">Enable all features</a> in Organizations. With only consolidated billing features enabled, you can’t create a StackSet with service-managed permissions.<br /> Enable trusted access with Organizations. After trusted access is enabled, StackSets creates the necessary IAM roles in the Organization’s management account and target (member) accounts when you create StackSets with service-managed permissions.</p> 
<p>Service-managed StackSets must be deployed in either the Organizations Management account or a <a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-orgs-delegated-admin.html">Delegated Administrator account for CloudFormation StackSets</a>.</p> 
<h2>CloudFormation custom resource backed by Lambda</h2> 
<p>A Lambda-backed CloudFormation custom resource is used to manage the credentials in ServiceNow for the account. The custom resource receives the IAM User access key and secret access key under resource properties. Deploying the solution as a CloudFormation custom resource enables you to manage the solution lifecycle for each AWS account and automate the solution deployment and update.</p> 
<p>Based on invocation type, a new account entry will be created in ServiceNow (CloudFormation create) or the existing account will be updated (CloudFormation Update). If you’re deploying the Custom Resource Lambda within a VPC, then make sure that any firewalls in use are updated to allow connectivity from Lambda to the ServiceNow endpoints.</p> 
<p>The Lambda function should be deployed in an AWS account, such as a shared services account, where connectivity to the configured ServiceNow endpoints is available. As shown in the previous figure, CloudFormation stack instances for each AWS account invoke the Lambda function to configure the SMC in ServiceNow. ServiceNow API credentials are stored in AWS Systems Manager Parameter Store and the Lambda function stores the generated IAM credentials in <a href="https://aws.amazon.com/secrets-manager/">AWS Secrets Manager</a> as a backup. When IAM credentials are changed on a schedule, the AWS Secrets Manager keys are updated as well.</p> 
<h2>Solution deployment</h2> 
<p>Follow these steps to deploy the solution. The source code is available <a href="https://github.com/aws-samples/aws-service-management-connector">here</a>.</p> 
<h2>Prerequisites</h2> 
<p>The following prerequisites are required to follow along with this post:</p> 
<ol> 
 <li>An <a href="https://aws.amazon.com/s3/">Amazon Simple Storage Service (Amazon S3)</a> bucket where you can copy files for Lambda function deployment.</li> 
 <li>A ServiceNow account and password for each target ServiceNow instance that ’you’ll be integrating with the <a href="https://docs.aws.amazon.com/servicecatalog/latest/smcguide/overview.html">AWS Service Management Connector</a>.</li> 
 <li>Permissions to deploy an IAM role in your Organizations Management/Root account or Delegated Admin account.</li> 
 <li>An AWS account where the Lambda function will be deployed and where your ServiceNow account credentials will be securely stored in AWS Systems Manager Parameter Store (referred to as <strong>Solution Deployment account</strong> in this README).</li> 
</ol> 
<h2>1. Storing the ServiceNow API credentials</h2> 
<p>The ServiceNow API endpoint credentials are stored in the AWS Systems Manager parameter store as a secret string. The path should be defined as shown in the following.</p> 
<ul> 
 <li>Log in to the AWS Shared Service Account</li> 
 <li>Navigate to AWS Systems Manager -&gt; Parameter Store</li> 
</ul> 
<p>Create the Username Parameter:</p> 
<ul> 
 <li>Select the <strong>Create parameter</strong> button.</li> 
 <li>Complete the fields on the form 
  <ul> 
   <li>Name: <strong>/ServiceNow/&lt;ServiceNow Instance Name&gt;.servicenow.com/username</strong></li> 
   <li>Description: Valid description for parameter. For example: “ServiceNow API username”</li> 
   <li>Tier: <strong>Standard</strong></li> 
   <li>Type: <strong>String</strong></li> 
   <li>Data Type: <strong>Text</strong></li> 
   <li>Value: Provide the username that will be used to authenticate with the ServiceNow AWS Accounts API.</li> 
  </ul> </li> 
 <li>Select the Create Parameter at the bottom to save.</li> 
</ul> 
<p>Create the password parameter:</p> 
<ul> 
 <li>Select the <strong>Create parameter</strong> button.</li> 
 <li>Complete the fields on the form 
  <ul> 
   <li>Name: <strong>/ServiceNow/&lt;ServiceNow Instance Name&gt;.servicenow.com/password</strong></li> 
   <li>Description: Valid description for parameter. For example: “ServiceNow API password”</li> 
   <li>Tier: <strong>Standard</strong></li> 
   <li>Type: <strong>SecureString</strong></li> 
   <li>KMS Key Source: <strong>My Current Account</strong></li> 
   <li>KMS Key ID: Choose the Key that will be used to encrypt the secret. Note that the Lambda function should have access to this key to decrypt the password parameter.</li> 
   <li>Data Type: <strong>Text</strong></li> 
   <li>Value: Provide the password that will be used to authenticate with the ServiceNow api.</li> 
  </ul> </li> 
 <li>Select <strong>Create parameter</strong> at the bottom to save.</li> 
</ul> 
<h2>2. Creating the custom resource Lambda</h2> 
<p>To deploy the Lambda function, the code must be packaged as a zip and uploaded to an S3 bucket in the shared services Account.</p> 
<p><strong>Package the Lambda function code following these instructions:</strong></p> 
<p>Make a build directory to store the files that must be packaged:</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-bash">mkdir -p src/build/</code></pre> 
</div> 
<p>Copy the python files to the build directory:</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-bash">cd src/functions
cp aws_client.py ../build
cp snow_client.py ../build
cp lambda_function.py ../build</code></pre> 
</div> 
<p>Install the Python dependencies targeting the build directory:</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-bash">pip install -r requirements.txt -t ../build</code></pre> 
</div> 
<p><strong>Linux:</strong></p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-bash">cd ../build
zip -r snowregister-v1.zip .</code></pre> 
</div> 
<p><strong>Windows:</strong></p> 
<ul> 
 <li>Navigate to the build folder.</li> 
 <li>Select all -&gt; Send to compressed file.</li> 
 <li>Name the zip file as snowregister-v1.zip.</li> 
</ul> 
<p>Upload the zip file to the S3 bucket through <a href="https://aws.amazon.com/cli/">AWS Command Line Interface (AWS CLI)</a>:</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-bash">aws s3 cp snowregister-v1.zip s3://&lt;bucket-name&gt;/ --region us-east-1
cd ../..
rm -rf src/build</code></pre> 
</div> 
<p><strong>Via Console:</strong></p> 
<ul> 
 <li>Log in to the <a href="https://aws.amazon.com/console/">AWS Console</a>.</li> 
 <li>Navigate to the Amazon S3 Service console.</li> 
 <li>Select the bucket to upload the zip file.</li> 
 <li>Choose <img alt="" class="alignnone wp-image-36449 size-full" height="34" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/01/25/cloudops_942_2_upload-1.png" width="115" /></li> 
 <li>Select the <strong>snowregister-v1.zip</strong> and upload.</li> 
</ul> 
<h2>3. Creating the required stacks and StackSets</h2> 
<h3>3.1 Create the Lambda function stack</h3> 
<p>Create a CloudFormation stack named servicenow-automation in the shared services account via AWS CLI. Update the following command by replacing the parameters with the correct values for S3CodeBucket, VpcId, SubnetIdList, ManagementAccountId, OrganizationID, AssumrRoleName, and AssumeRoleAccountID.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-bash">aws cloudformation deploy --stack-name servicenow-automation \
--template-file src/templates/servicenow_connector_auto_register.yaml \
--parameter-overrides AssumeRoleArn=None \
S3CodeBucket='&lt;S3 Bucket Name&gt;' \
VpcId='vpc-id-1' \
SubnetIdList='subnet-1,subnet-2' \
ManagementAccountId='111111111111' \
OrganizationID='o-xxxxxx' \
--region us-east-1 --capabilities CAPABILITY_NAMED_IAM</code></pre> 
</div> 
<h3>3.2 Setup the assume role</h3> 
<p>The Lambda function needs access to the Management or Delegated admin account to query the Organizations service to get the account name using the account ID. To achieve this, a role must be created in the Management or Delegated admin account. Run the following command after updating the values for SourceRoleArn and ManagementAccountId.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-bash">aws cloudformation deploy --stack-name servicenow-lambda-assume-role-stack \
--template-file src/templates/servicenow_assume_role.yaml –parameter-
overrides SourceRoleArn=&lt;Lambda Role ARN&gt; ManagementAccountId =’11111111111’ --region us-east-1</code></pre> 
</div> 
<h3>3.3 Deploy the IAM User Stackset (Delegated Admin or Management Account)</h3> 
<p>The IAM stack invokes the Lambda as a Custom Resource to insert/update the IAM User access tokens into ServiceNow. The custom resource properties manages the service integrations configuration of the connector.</p> 
<p>The highlighted properties in the custom resource properties in the following CloudFormation snippet enable or disable the connector integration with the AWS Services. Update the template src/templates/servicenow_connector_global_config.yaml with the desired settings for the service integrations before creating the stackset.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-bash">RegisterSerNow:
    Type: Custom::ServiceNowReg
    Properties:
      ServiceToken: !Ref LambdaArn
      Region: !Ref "AWS::Region"
      AccountId: !Ref AWS::AccountId
      KeyVersion: !Ref KeyVersion
      ServiceNowUrl: !Ref ServiceNowUrl
      SCEndUser:
        UserName: !Ref SCEndUserName
        AccessKey: !Ref 'SCEndUserAccessKeys'
        SecretAccessKey: !GetAtt 'SCEndUserAccessKeys.SecretAccessKey'
      SCSyncUser:
        UserName: !Ref SCSyncUserName
        AccessKey: !Ref 'SCSyncUserAccessKeys'
        SecretAccessKey: !GetAtt 'SCSyncUserAccessKeys.SecretAccessKey'
      
      EnableSystemsManager: False    
      EnableChangeManager: False    
      EnableOpsCenter: False        
      EnableServiceCatalog: False    
      EnableConfig: False            
      EnableAwsSupport: False        
      EnableSecurityHub: False       
      EnableHealthDashboard: False  
      EnableIncidentManager: False  
      EnableRegions:                
        - us-east-1
      Tags:
        - Key: App_Name
          Value: Service_Now_connector</code></pre> 
</div> 
<p>Run the following command after updating the values for</p> 
<p><strong>LambdaArn</strong> – This is the ARN of the Lambda function deployed in the Shared Service Account.</p> 
<p><strong>KeyVersion</strong> – Starts from 1.</p> 
<p><strong>ServiceNowUrl</strong> – The ServiceNow instance URL (&lt;ServiceNow instance name&gt;.servicenow.com).</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-bash">aws cloudformation create-stack-set --stack-set-name service-snow-config --description "Deploys IAM User in the target OU and calls the custom resource to store the keys in service now" --template-body file://src/templates/servicenow_connector_global_config.yaml \
--parameters ParameterKey=LambdaArn,ParameterValue='&lt;Lambda Function ARN&gt;' \
ParameterKey=KeyVersion,ParameterValue=1 \
ParameterKey=ServiceNowUrl,ParameterValue='XXXXX.servicenow.com' \
--capabilities CAPABILITY_NAMED_IAM \
--permission-model SERVICE_MANAGED \
--auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false \
--call-as DELEGATED_ADMIN \
--managed-execution Active=false \
--region us-east-1</code></pre> 
</div> 
<p>Create stack instances in the target Organization or OU.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-bash">aws cloudformation create-stack-instances --stack-set-name service-snow-config \
--deployment-targets OrganizationalUnitIds='ou-XXXXXXX' \
--regions us-east-1 \
--call-as DELEGATED_ADMIN \
--region us-east-1</code></pre> 
</div> 
<h3>3.4 Deploy Stackset to create an SQS queue for Security hub and Health Dashboard</h3> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-bash">aws cloudformation create-stack-set --stack-set-name servicenow-config-regional \
--description "Deploys IAM User in the target OU and calls the custom resource to store the keys in ServiceNow" \
--template-body file://src/templates/servicenow_connector_regional_config.yaml \
--capabilities CAPABILITY_NAMED_IAM \
--permission-model SERVICE_MANAGED \
--auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false \
--call-as DELEGATED_ADMIN \
--managed-execution Active=false \
--region us-east-1</code></pre> 
</div> 
<p>Add Target to the above Stackset. The Stackset name should match the above.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-bash">aws cloudformation create-stack-instances --stack-set-name service-snow-config-regional \
--deployment-targets OrganizationalUnitIds='ou-XXXXXXX' \
--regions us-east-1 \
--call-as DELEGATED_ADMIN \
--region us-east-1</code></pre> 
</div> 
<h3>3.5 To integrate the AWS Support, deploy another Stackset for the Support Queue as required by the connector (4.5.0)</h3> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-bash">aws cloudformation create-stack-set --stack-set-name service-snow-config-support \
--description "Deploys SQS queue for AWS Support integration" \
--template-body file://src/templates/servicenow_connector_support_queue.yaml \
--capabilities CAPABILITY_NAMED_IAM \
--permission-model SERVICE_MANAGED \
--auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false \
--call-as DELEGATED_ADMIN \
--managed-execution Active=false \
--region us-east-1</code></pre> 
</div> 
<p>Add Target to the regional Stackset. The Stackset name should match the above.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-bash">aws cloudformation create-stack-instances --stack-set-name servicenow-aws-support \
--deployment-targets OrganizationalUnitIds='ou-XXXXXXX' \
--regions us-east-1 \
--call-as DELEGATED_ADMIN \
--region us-east-1</code></pre> 
</div> 
<h2>4. Key rotation</h2> 
<p>As a security best practice, the IAM User Access and Secret access tokens should be changed on a set schedule to minimize the impact if the keys get compromised.</p> 
<p>To rotate the keys, increment the KeyVersion parameter and update the StackSet. The update action forces the CloudFormation AWS::IAM::AccessKey to recreate the keys. The custom resource gets called with the new keys and the Lambda function updates the existing account in the ServiceNow with a new set of keys.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-bash">aws cloudformation update-stack-set --stack-set-name servicenow-iam-users --description "Deploys IAM User in the target OU and calls the custom resource to store the keys in ServiceNow" --template-body file://src/templates/servicenow_connector_global_config.yaml \
--parameters ParameterKey=LambdaArn,ParameterValue='&lt;Lambda Function ARN&gt;' \
ParameterKey=KeyVersion,ParameterValue=1 \
ParameterKey=ServiceNowUrl,ParameterValue='&lt;ServiceNow instance name&gt;.servicenow.com' \
--capabilities CAPABILITY_NAMED_IAM \
--permission-model SERVICE_MANAGED \
--auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false \
--call-as DELEGATED_ADMIN \
--managed-execution Active=false \
--region us-east-1</code></pre> 
</div> 
<h2>Use cases</h2> 
<p>For existing AWS accounts within the Organization</p> 
<p>The above solution will deploy and create the stack instances and StackSets to all of the accounts within the target OUs.</p> 
<h3>New AWS account creation</h3> 
<p>The stackset is set to deploy instances targeting an OU. The deployment permission model is SERVICE_MANAGED. This makes sure that anytime a new account gets added to the target OUs, the stackset will automatically create a new stack-instance in the new account.</p> 
<h3>Deleting an AWS account</h3> 
<p>Removing an account from an OU automatically triggers the StackSet to delete the stack instance from the account. The account entry from ServiceNow is <strong>NOT</strong> removed automatically to preserve existing synched resources in ServiceNow.</p> 
<h2>Cleanup</h2> 
<p>To clean up the deployed solution, remove the stack instances from the CloudFormation StackSets. You can use the CloudFormation console or the AWS CLI to remove the instances from the StackSet. Wait for all of the instances to be removed before attempting to delete the StackSet. Delete the StackSets after all of the instances are removed. The solution doesn’t handle the deletion of the account from the Service Management Connector on the ServiceNow platform. Log in to the ServiceNow portal to manually remove the AWS account from the connector.</p> 
<h2>Summary</h2> 
<p>This post demonstrates how a customer can implement an automated solution for SMC configuration with multiple AWS accounts and ServiceNow instances. Using this approach, customers can eliminate the manual effort of onboarding AWS Account integration with ServiceNow using the SMC.&nbsp; You can fork the GitHub solution repository and use it as-is or customize it further to your liking.</p> 
<p>The <a href="https://docs.aws.amazon.com/smc/latest/ag/sn-what-is.html">AWS Service Management Connector</a> for ServiceNow application is available at no charge to all AWS customers. Please try the above solution with the connector and If you have feedback about this post, submit comments in the&nbsp;Comments&nbsp;section below.</p> 
<p><strong>About the authors:</strong></p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="aligncenter size-full wp-image-11636" height="160" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/12/28/Sivasankar-Achin-Jayapal.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Sivasankar Achin Jayapal</h3> 
  <p>Sivasankar Achin Jayapal is a Cloud Infrastructure Architect with AWS. He specializes in developing solutions to automate Infrastructure deployments and writing Infrastructure as Code. In his free time, he likes to spend time with family and friends, bingee-watch TV shows, take road trips, and explore new things.</p> 
 </div> 
</footer> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="aligncenter wp-image-11636" height="160" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/12/28/mjoethom.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Joe Thomas</h3> 
  <p>Joe Thomas is a Systems Development Engineer with the AWS Service Management Connectors team. He is passionate about driving operational excellence, troubleshooting customer issues, and enjoys building product features and enhancements that drive customer adoption and satisfaction. Outside of work, Joe enjoys spending time with his family on short summer hikes or traveling across the country exploring various food cultures.</p> 
 </div> 
</footer> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="aligncenter wp-image-11636" height="160" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/12/28/Khurram-Nizami.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Khurram Nizami</h3> 
  <p>Khurram Nizami is a Senior Consultant with AWS. Khurram is passionate about helping people build innovative solutions using technology. In his free time, Khurram enjoys hiking, nature, DIY projects, and travel.</p> 
 </div> 
</footer>
