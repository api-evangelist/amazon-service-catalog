---
title: "AWS Service Catalog Account Factory-Enhanced"
url: "https://aws.amazon.com/blogs/mt/aws-service-catalog-account-factory-enhanced/"
date: "Wed, 30 Nov 2022 01:14:08 +0000"
author: "Kenneth Walsh"
feed_url: "https://aws.amazon.com/blogs/mt/tag/aws-service-catalog/feed/"
---
<p>Many enterprise customers who use <a href="https://aws.amazon.com/controltower/">AWS Control Tower</a> to create accounts want an uncomplicated way to extend the next steps in the account creation process. These next steps cover common business use cases, including creating networks, security profiles, governance, and compliance. Executing these processes for every new account created manually is cumbersome and challenging to manage. Using third-party service providers to address the process can be expensive.</p> 
<p>There is the option to use Customizations for Control Tower to help alleviate some of these pain points. This solution lets you add customizations to AWS Control Tower and deploy your customizations to existing and new accounts. However, customers are looking for a more simplified way to create AWS accounts with enhancements unique to each account.</p> 
<p>This is where AWS Account Factory Enhancements come in. This solution leverages <a href="https://aws.amazon.com/servicecatalog/">AWS Service Catalog</a> to present an AWS Account Factory product to the End User to create an AWS account and, in the creation process, add enhancements that they would like. The enhancements are based on <a href="https://docs.aws.amazon.com/cloudformation">AWS CloudFormation</a> templates launched in the newly created account. The templates can perform fundamental tasks in the new accounts, like creating networks, security roles, storage profiles, configuring threat detection, and more.</p> 
<p>This particular post will show how you can add an <a href="https://aws.amazon.com/s3/">Amazon Simple Storage Service</a> (Amazon S3) for storage and/or <a href="https://aws.amazon.com/guardduty/">Amazon GuardDuty</a> for intelligent threat detection to the AWS account configuration process. Although we’re only showing a few options, this blog will also show you how to extend this capability by adding additional CloudFormation templates to address other business requirements.</p> 
<p>This post will show you how to use the Service Catalog Account Factory Enhance product to create accounts and perform several template deployments as additional steps.</p> 
<h2>Prerequisites­</h2> 
<ul> 
 <li>AWS Control Tower must be launched in your account</li> 
 <li>You must have access to create portfolios and products within Service Catalog</li> 
</ul> 
<p>To follow the steps in this post, you need an AWS account with permissions to create resources in these services:</p> 
<ul> 
 <li><a href="https://aws.amazon.com/s3/">Amazon Simple Storage Service (S3)</a></li> 
 <li><a href="https://aws.amazon.com/cloudformation/faqs/">AWS CloudFormation</a></li> 
 <li><a href="https://docs.aws.amazon.com/dynamodb/?icmpid=docs_homepage_databases">Amazon DynamoDB</a></li> 
 <li><a href="https://aws.amazon.com/servicecatalog/faqs/">AWS Service Catalog</a></li> 
 <li><a href="https://aws.amazon.com/lambda/">AWS Lambda</a></li> 
 <li><a href="https://docs.aws.amazon.com/step-functions/?icmpid=docs_homepage_serverless">AWS Step Functions</a></li> 
 <li><a href="https://docs.aws.amazon.com/apigateway/?icmpid=docs_homepage_serverless">Amazon API Gateway</a></li> 
</ul> 
<h2>Concepts and terminology</h2> 
<p>The following AWS Service Catalog concepts are used in this post</p> 
<ul> 
 <li>A <a href="http://docs.aws.amazon.com/servicecatalog/latest/adminguide/what-is_concepts.html#what-is_concepts-product">product</a> is a blueprint for building the AWS resources necessary to make it available for deployment on AWS, along with the configuration information. Create a product by importing an <a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-whatis-concepts.html">AWS CloudFormation template</a>, or, in case of <a href="https://aws.amazon.com/partners/aws-marketplace/">AWS Marketplace-based</a> products, by copying the product to the AWS Service Catalog. A product can belong to multiple portfolios.</li> 
 <li>A <a href="http://docs.aws.amazon.com/servicecatalog/latest/adminguide/what-is_concepts.html#what-is_concepts-portfolio">portfolio</a> is a collection of products, together with the configuration information. Use portfolios to manage user access to specific products. You can grant portfolio access for an <a href="https://aws.amazon.com/iam/">AWS Identity and Access Management (IAM)</a> user, IAM group, or IAM role level.</li> 
 <li>A <a href="http://docs.aws.amazon.com/servicecatalog/latest/adminguide/what-is_concepts.html#what-is_concepts-provprod">provisioned product</a> is an <a href="http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacks.html">AWS CloudFormation stack</a>. In other words, the AWS resources that are created. When an end-user launches a product, AWS Service Catalog provisions the product from an AWS CloudFormation stack.</li> 
 <li><a href="http://docs.aws.amazon.com/servicecatalog/latest/adminguide/what-is_concepts.html#what-is_concepts-constraints">Constraints</a> control the way that users can deploy a product. Launch constraints let you specify a role that the AWS Service Catalog can assume to launch a product.</li> 
</ul> 
<h2>Solution overview</h2> 
<p>The following diagram maps out the solution architecture.</p> 
<h3><img alt="The administrator launches the deployment process which creates the components needed for the solutions. The admin or end user then uses a service catalog product to deploy templates in a spoke account" class="aligncenter wp-image-34397" height="671" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/11/16/blog-dev902Arcv4a4.png" width="700" /></h3> 
<h3>Diagram</h3> 
<h4>A</h4> 
<p>A Service Catalog administrator will manage and launch the AWS CloudFormation template that includes the enhancements required by the business.</p> 
<h4>B</h4> 
<p>When an Account Factory Enhanced account product is launched, this kicks off an AWS Step Functions workflow that will add in the enhancements desired.</p> 
<h3>Gather the prerequisites</h3> 
<p><strong>You will also need these prerequisites:</strong></p> 
<ul> 
 <li>Portfolio ID containing the ‘AWS Control Tower Account Factory’ product</li> 
 <li>Service Catalog product name of ‘AWS Control Tower Account Factory’ product. It should be the same.</li> 
</ul> 
<ol> 
 <li>Log in to the <a href="https://aws.amazon.com/console/">AWS Console</a> using the management account in your AWS Control Tower environment.</li> 
 <li>Navigate to the Service Catalog landing page.</li> 
 <li>Select <strong>Portfolios</strong> under Administration on the left.</li> 
 <li>Search for <strong>AWS Control Tower Account Factory Portfolio</strong>.</li> 
 <li>Select the <strong>AWS Control Tower Account Factory Portfolio</strong> link.</li> 
 <li>Copy the <strong>Portfolio ID</strong>.</li> 
 <li>Copy the <strong>Product Name</strong>.</li> 
</ol> 
<p><img alt="this screen shows how the user can find and copy the portfolio ID" class="aligncenter wp-image-34385" height="495" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/11/16/blog902preqeq.png" width="700" /></p> 
<h2>Configure the environment</h2> 
<p>Download the CloudFormation template and upload this to an Amazon S3 bucket.</p> 
<ol> 
 <li>Download the content in this <a href="https://github.com/aws-samples/aws-service-catalog-reference-architectures/blob/master/blog_content/service_catalog_enhanced_acct_fact/scenhanceaccafact.zip?raw=true">zip file</a>.</li> 
 <li>Extract the zip file, and it will create a folder called <strong>content</strong>.</li> 
 <li>Log in to your AWS account as an administrator that can create AWS resources.</li> 
 <li>Create an Amazon S3 bucket and note this name.</li> 
 <li>Upload the content folder to your newly created S3 bucket.</li> 
 <li>Drill down into the <strong>content</strong>/<strong>scenhanceaf</strong> folder.</li> 
 <li>Choose the checkbox next to <strong>scenhanceaf_setup.json</strong>.</li> 
 <li>Right-click and copy the <strong>Object URL</strong>.</li> 
</ol> 
<h3>Deploy the CloudFormation template</h3> 
<ol> 
 <li>Navigate to <a href="https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks?filteringStatus=active&amp;filteringText=&amp;viewNested=true&amp;hideStacks=false">CloudFormation</a> console.</li> 
 <li>Select <strong>Stacks</strong> on the left navigation pane. Select <strong>Create stack</strong> and choose <strong>with new resources (standard)</strong>.</li> 
 <li>On the ‘Create stack’ page, under ‘specify template’ add in the Amazon S3 URL from step 8. Choose <strong>Next</strong>.</li> 
 <li>On the ‘Specify stack details’ page, enter a stack name.</li> 
 <li>Under Parameters: 
  <ol> 
   <li>Copy the <strong>portfolio ID</strong> from the prerequisites step</li> 
   <li><strong>AccountFactoryPortfolioId</strong>. Enter product id from prerequisites step</li> 
   <li><strong>Scproductname</strong> enter the Service Catalog product name, e.g., [AWS Control Tower Account Factory] from prerequisites step.</li> 
   <li>Under <strong>SourceBucket</strong>, enter in the S3 bucket name that you created.</li> 
  </ol> </li> 
 <li>On <strong>Configure stack</strong> options page choose <strong>Next</strong>.</li> 
 <li>On the <strong>Review</strong> page, select the check boxes that acknowledge <strong>that AWS CloudFormation might create IAM resources with custom names and that it might require the CAPABILITY_AUTO_EXPAND</strong>.</li> 
 <li>Choose <strong>Create stack</strong>.</li> 
 <li>Wait for <strong>Create Complete</strong>.</li> 
</ol> 
<h3>Launch the Account Factory Enhanced product(s): Account Factory and templates</h3> 
<p>This will create an account and deploy up to five stacks into the new account.</p> 
<p><strong>Prerequisites­</strong></p> 
<ul> 
 <li>A role/user with administrative permission to create an account, for example, the management account in the AWS Control Tower environment.</li> 
</ul> 
<h3>Provide access to the new Portfolio</h3> 
<ol> 
 <li> 
  <ol> 
   <li>Log in to the <a href="https://aws.amazon.com/console/">AWS Console</a> using the management account in your AWS Control Tower environment.</li> 
   <li>Navigate to <a href="https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks?filteringStatus=active&amp;filteringText=&amp;viewNested=true&amp;hideStacks=false">CloudFormation</a> console that you just created.</li> 
   <li>Choose the <strong>Resources</strong> tab.</li> 
   <li>Choose the <strong>EnhanceAFPortfolio</strong> URL.</li> 
   <li>Choose <strong>Group, roles, and users</strong> tab<img alt="the user selects grup and add group and user" class="wp-image-34387 aligncenter" height="170" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/11/16/blog902-porfolio-adduser001-1024x249.png" width="700" /></li> 
   <li>Choose <strong>Add groups, roles, users</strong> button</li> 
   <li>Select the <strong>Group, Role or Users</strong> depending on how you logged in, in step 1</li> 
   <li>Select the <strong>check box</strong> next to your Group, Role or user<img alt="the admin selects a user" class="aligncenter wp-image-34388" height="272" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/11/16/blog902-porfolio-adduser002-1024x398.png" width="700" /></li> 
   <li>Select the <strong>Add access</strong> button</li> 
  </ol> </li> 
</ol> 
<h3>Launch the Account Factory Enhanced product</h3> 
<ol> 
 <li>Choose <strong>Products</strong> from the top left</li> 
 <li>Choose the <strong>AccountFactoryEnhanced</strong> product.</li> 
 <li>&nbsp;Choose <strong>Launch Product</strong>.</li> 
 <li>Select the check box next to Generate Name<img alt="the admin selects the parameters" class="aligncenter wp-image-34390" height="978" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/11/16/blogdev902scprodpara-733x1024.png" width="700" /></li> 
 <li>For Parameters: 
  <ol> 
   <li>AccountEmail enter the account email</li> 
   <li>AccountName enter the account name</li> 
   <li>ManageOrganizationalUnit select one from the dropdown</li> 
   <li>SSOUserEmail enter the SSOUserEmail</li> 
   <li>SSOUserFirstName</li> 
   <li>SSOUserLastName</li> 
  </ol> </li> 
 <li>For Enhancement Steps parameters: 
  <ol> 
   <li>Select a <strong>stack</strong> from the dropdown. You can run up to five stacks in the new account. Select <strong>None</strong> to skip the stack run per step.</li> 
  </ol> </li> 
 <li>Choose <strong>Launch product</strong></li> 
</ol> 
<p>The Account factory Service Catalog product will launch, followed by a StepFunction which will launch each selected stack in sequence in the newly created account. Now you’ve created an AWS account customized with enhancements!</p> 
<h3>Verify Account creation and stack deployment</h3> 
<h4>Account creation</h4> 
<ol> 
 <li>Navigate to the <a href="https://us-east-1.console.aws.amazon.com/controltower/home/">AWS Control Tower Console</a></li> 
 <li>Choose <strong>Organization</strong></li> 
 <li>Choose the manage organization unit (OU) to which you added the account</li> 
</ol> 
<p>The account should be listed with an Enrolled state.</p> 
<h4>Stack Deployment</h4> 
<ol> 
 <li>Log in to the newly created <strong>manage account</strong></li> 
 <li>Navigate to <a href="https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks?filteringStatus=active&amp;filteringText=&amp;viewNested=true&amp;hideStacks=false">CloudFormation</a> console</li> 
</ol> 
<p>A stack for each selection that you choose should be listed.</p> 
<h3>Launch the Account Factory Enhanced product(s): <em>Templates only</em> deployed in a managed account.</h3> 
<p style="padding-left: 40px;">This will deploy up to five stacks into an existing manage account created by Account Factory. The stack will also create an IAM role with console access</p> 
<ol> 
 <li>Navigate to the <a href="https://us-east-1.console.aws.amazon.com/servicecatalog/home?region=us-east-1#portfolios?activeTab=localAdminPortfolios">Service Catalog Console</a></li> 
 <li>Choose <strong>Products</strong> from the top left</li> 
 <li>Choose the <strong>AccountFactoryEnhancedTemplate</strong> product</li> 
 <li>Choose <strong>Launch Product</strong></li> 
 <li>Select the check box next to <strong>Generate Name</strong></li> 
 <li>For <strong>TargetAccount</strong> select the account to deploy the stack into.</li> 
 <li>For Enhancement Steps parameters: 
  <ol> 
   <li>Select a <strong>template</strong> from the dropdown. You can run up to five stacks in the manage account. Select <strong>None</strong> to skip the stack run per step.</li> 
  </ol> </li> 
 <li>Choose <strong>Launch product</strong></li> 
</ol> 
<p>The StepFunction will launch each selected stack in sequence in the managed account.</p> 
<h3>Verify Account creation and stack deployment</h3> 
<h4>Stack Deployment</h4> 
<ol> 
 <li>Log in to the newly created <strong>manage account</strong></li> 
 <li>Navigate to <a href="https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks?filteringStatus=active&amp;filteringText=&amp;viewNested=true&amp;hideStacks=false">CloudFormation</a> console</li> 
</ol> 
<h3>Updating the Service Catalog Account Factory Enhanced product.</h3> 
<p>The Account Factory Enhanced product is automatically updated when new accounts are created by the AWS Control Tower Account factory process.</p> 
<p><strong>Adding new CloudFormation templates to be deployed after the account is created.</strong></p> 
<p><strong>Prerequisites­</strong></p> 
<ul> 
 <li>Create and test the new CloudFormation template. The template should be able to run without parameters in a new account.</li> 
</ul> 
<h4>Uploading the new template</h4> 
<ol> 
 <li>Log in to the AWS Console using the management account in your AWS Control Tower environment.</li> 
 <li>Navigate to the <a href="https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks?filteringStatus=active&amp;filteringText=&amp;viewNested=true&amp;hideStacks=false">CloudFormation</a> console and the stack used to deploy the solution.</li> 
 <li>Choose the <strong>Outputs</strong> tab.</li> 
 <li>Choose the <strong>TemplateLibraryFolder</strong> URL</li> 
 <li>Choose the <strong>Upload</strong> button</li> 
 <li>Choose the <strong>Add files</strong> button</li> 
 <li>Select the new template</li> 
 <li>Choose the <strong>Upload</strong> button</li> 
</ol> 
<p>The Service Catalog Account Factory Enhanced product will be updated, and the new CloudFormation template will be available to deploy during the account creation step.</p> 
<h2><strong>Cleanup</strong></h2> 
<p>You must close any AWS accounts created that you don’t plan to continue to use, as well as remove the Service Catalog product and portfolio created.</p> 
<h2>Conclusion</h2> 
<p>AWS Service Catalog enables organizations to create and manage catalogs of approved IT services for use on AWS. In this post, we showed how Service Catalog can extend the capabilities of the account creation process in AWS Control Tower, addressing standard business requirements when creating AWS accounts.</p> 
<h3>About the authors:</h3> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Kenneth Walsh" class="size-full wp-image-20009 alignleft" height="129" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2021/04/28/KennethWalsh.png" width="128" />
  </div> 
  <h3 class="lb-h4">Kenneth Walsh</h3> 
  <p>Kenneth Walsh is a New York-based Solutions Architect whose focus is AWS Marketplace. Kenneth is passionate about cloud computing and loves being a trusted advisor for his customers. When he’s not working with customers on their journey to the cloud, he enjoys cooking, audio books, movies, and spending time with his family and dog.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="size-thumbnail wp-image-22324 alignleft" height="150" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/01/06/Devipaulvannan.png" width="150" />
  </div> 
  <p><strong>Devi Paulvannan Chapman</strong></p> 
  <p>Devi Paulvannan Chapman is a Solutions Architect with AWS. She enjoys working with customers to provide architectural and technical guidance on their cloud journey. Outside of work, she loves spending time outdoors rock climbing, hiking, and traveling to new places.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="alignleft wp-image-34402" height="148" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/11/17/blodayoOmosebi.png" width="181" />
  </div> 
  <h3 class="lb-h4">Ayo Omosebi</h3> 
  <p>Ayo is a Sr. Business Services SDM at AWS. He is passionate about building and promoting integrations between AWS services and customers business platforms. Outside of work, he enjoys spending time with his family outdoors, running and mountain hiking.</p> 
 </div> 
</footer>
