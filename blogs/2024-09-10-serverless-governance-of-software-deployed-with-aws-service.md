---
title: "Serverless Governance of Software Deployed with AWS Service Catalog"
url: "https://aws.amazon.com/blogs/mt/serverless-governance-of-software-deployed-with-aws-service-catalog/"
date: "Tue, 10 Sep 2024 16:46:58 +0000"
author: "Ran Isenberg (AWS Serverless Hero)"
feed_url: "https://aws.amazon.com/blogs/mt/tag/aws-service-catalog/feed/"
---
<p><a href="https://aws.amazon.com/servicecatalog/">AWS Service Catalog</a> (Service Catalog) is a powerful tool that empowers organizations to manage and govern approved services and resources. It significantly benefits platform engineering by standardizing environments, accelerating service delivery, and enhancing security. With its automated provisioning and resource management, Service Catalog supports infrastructure as code, enabling scalable, reliable deployments.</p> 
<p>Platform engineering teams are responsible for maintaining the security and governance of software deployed within their organizations. These platform teams are challenged with meeting diverse customer requirements for deployment options and software updates. At the same time, they are responsible for security and governance of the software under their management.</p> 
<p>Tools like Service Catalog can help facilitate software deployment, but platform teams still need tools that provide visibility into the versions and locations of their organization’s software. This is crucial, as the software may include applications created by the organization, as well as third-party solutions. Ensuring the secure deployment of software with properly scoped permissions using <a href="https://aws.amazon.com/iam/">AWS Identity and Access Management</a> (IAM) to <a href="https://docs.aws.amazon.com/wellarchitected/latest/framework/sec_permissions_least_privileges.html">grant least privilege</a> is also critical.</p> 
<p>In this blog, we’ll review a serverless solution that addresses the key requirements of security, visibility, and governance that platform teams need to safely manage software.</p> 
<h2>Overview of solution</h2> 
<p>The solution contains two components, let’s examine those, then break down the architecture in Figure 1 below.</p> 
<p><img alt="" class="aligncenter size-full wp-image-55899" height="754" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/09/05/Serverless-Governance-components-1.png" width="1341" /></p> 
<p style="text-align: center;"><strong><em>Figure 1: The components distributed across two AWS Accounts</em></strong></p> 
<p>On the left side is the platform engineering account. It contains a Service Catalog portfolio and its extension, the <a href="https://aws.amazon.com/dynamodb/">Amazon DynamoDB</a> visibility table. Products can be provisioned once the portfolio is shared with an organization account and <a href="https://aws.amazon.com/cloudformation/">AWS CloudFormation</a> stacks are created.</p> 
<h3>Component One: Platform engineering products portfolio</h3> 
<p>The platform engineering products <a href="https://docs.aws.amazon.com/servicecatalog/latest/adminguide/catalogs_portfolios.html">portfolio</a> is published in an AWS account called “platform engineering” and contains products, either internal or 3rd party, that the organization deploys to its internal teams’ accounts. A product, in its essence, is a simple Infrastructure as Code (IaC) construct, like a CloudFormation template in this example, or an artifact of <a href="https://www.terraform.io/">Terraform</a>, <a href="https://www.ansible.com/">Ansible</a>, or any other IaC provider. Each product extends the organization’s governance or security practices. For example, one product can deploy a CloudFormation stack that creates a role for a CI/CD pipeline with just the bare minimum permissions required by that pipeline process, implementing the least privileged access practice. Another product could create an organization-approved <a href="https://aws.amazon.com/waf/">AWS Web Application Firewall</a> and <a href="https://docs.aws.amazon.com/waf/latest/developerguide/waf-rules.html">rules</a> that can be attached to any <a href="https://aws.amazon.com/api-gateway/">Amazon API Gateway</a> or<a href="https://aws.amazon.com/cloudfront/"> Amazon CloudFront</a>, thus enabling the same security and governance mechanisms across the organization’s APIs.</p> 
<p>The customers of these products, the organizations internal customers, developers and DevOps engineers, can deploy to AWS accounts across their organization with confidence, knowing that they are using a robust and reliable tool.</p> 
<p>As we’ll demonstrate in our example below, Service Catalog can help facilitate software deployment, but platform teams still need tools that provide visibility into the versions and locations of their organization’s software. This is crucial because the software may include applications created by the organization, as well as third-party solutions.</p> 
<h3>Component Two: Portfolio’s extended visibility component</h3> 
<p>The second component, the visibility mechanism, extends the portfolio. Service Catalog portfolios provide a managed way to publish and share portfolios and products across the organization while supporting versions. However, once the internal customers deploy the products to their accounts, i.e., provision the products, Service Catalog does not offer native visibility as to which product was deployed to what account, who deployed it, and what version was deployed. The visibility mechanism extends Service Catalog and adds a new layer of visibility and information. For every provisioned product, we can store valuable metadata such as the internal team’s name that deployed it, the portfolio it belongs to (its id), the deployed account and region, and product version. This mechanism is based on serverless technology and, in its center, a DynamoDB table. We will discuss this component’s inner workings in the next section.</p> 
<p>For platform teams, maintaining visibility into deployed products is important on multiple fronts. Gaining visibility means knowing what version of each product is deployed, and where. Older versions of products could have negative implications for security and compliance. Platform teams may also be tasked with staying within the licensing terms of third-party products as well, which again means understanding where, and what versions of our software, are deployed.</p> 
<p>Increased visibility drives operational efficiency as well acting as a tool to help keep internal teams using the same version of software wherever possible, a best practice that allows us to operate our internal products like SaaS solutions. Most importantly, the platform team, the security team, and the compliance team, all will find their experiences improved by having better governance over the applications they manage.</p> 
<h2>Solution High Level Design</h2> 
<p>The solution is divided into two domains represented by two CDK constructs:</p> 
<ol> 
 <li><a href="https://github.com/ran-isenberg/governance-sample-aws-service-catalog/blob/main/cdk/catalog/portfolio_construct.py">Platform engineering products portfolio</a></li> 
 <li><a href="https://github.com/ran-isenberg/governance-sample-aws-service-catalog/blob/main/cdk/catalog/visibility_construct.py">Portfolio’s extended visibility</a></li> 
</ol> 
<p>The first construct defines the Service Catalog portfolio and its products.</p> 
<p>Let’s focus on the second construct, which extends the capabilities of the first.</p> 
<h3>Visibility Mechanism Low Level Design</h3> 
<p>The visibility construct defines an <a href="https://aws.amazon.com/sns/">Amazon SNS</a> topic, an <a href="https://aws.amazon.com/sqs/">Amazon SQS</a> queue that subscribes to the topic, a DynamoDB table that will store product metadata and usage, and a <a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources-sns.html">CloudFormation custom resource</a>. Custom resources will be created for each of the portfolio’s products. It is the connecting resource between the portfolio’s products and the visibility mechanism. During creation, the custom resource will send the product’s metadata to the SNS topic, thus providing visibility and information to the platform engineering team. The custom resource supports product update and delete actions, giving complete insights into the products’ lifecycle of the organization account.</p> 
<h3>Provisioning a Product Flow</h3> 
<p>Figure 2 below describes the process of an admin user provisioning products from the platform engineering portfolio in an organization account. It demonstrates both parts but focuses on the visibility mechanism.</p> 
<p>Please note that before a user can provision a product, the following steps are mandatory:</p> 
<ol> 
 <li>Share portfolio with specific account or organization.</li> 
 <li>Import portfolio</li> 
 <li>Set portfolio access to specific users.</li> 
</ol> 
<p><img alt="" class="aligncenter size-full wp-image-56174" height="1888" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/09/08/Serverless-Governance-admin-flow-2.png" width="3212" /></p> 
<p style="text-align: center;"><strong><em>Figure 2: The admin user provisioning flow</em></strong></p> 
<p>Let’s review the diagram steps:</p> 
<p style="padding-left: 2em;"><strong>Step 1:</strong> The “Admin” initiates the provisioning of a product through Service Catalog and the platform engineering portfolio.</p> 
<p style="padding-left: 2em;"><strong>Step 2:</strong> Service Catalog triggers the creation of a CloudFormation stack for the provisioned product. For this example, the admin chose the CI/CD product.</p> 
<p style="padding-left: 2em;"><strong>Step 3:</strong> The CI/CD pipeline role is created.</p> 
<p style="padding-left: 2em;"><strong>Step 4: </strong>The stack includes a custom resource that is created during the stack creation process.</p> 
<p style="padding-left: 2em;"><strong>Step 5:</strong> The custom resource publishes a message containing the provisioned product metadata and parameters (consumer name) to the platform engineering’s Amazon SNS topic and enters a wait state.</p> 
<p style="padding-left: 2em;"><strong>Step 6:</strong> The SNS topic forwards the message to an SQS queue in the Platform Engineering Service Catalog Account.</p> 
<p style="padding-left: 2em;"><strong>Step 7:</strong> An <a href="https://aws.amazon.com/lambda/">AWS Lambda</a> function is triggered to handle the message from the SQS queue.</p> 
<p style="padding-left: 2em;"><strong>Step 8:</strong> The Lambda function adds information about the provisioned product, including its version, region, and user details, to the visibility DynamoDB table for enhanced portfolio visibility.</p> 
<p style="padding-left: 2em;"><strong>Step 9:</strong> The stack is released from the wait state, and the deployment process is completed.</p> 
<p>We have automated the visibility process by utilizing the CloudFormation ability to send SNS messages with custom parameters during the lifecycle of the provisioned products. During product update or deletion, a similar process triggers which results in the visibility table updating the product entry or deleting it.</p> 
<h4>More detailed information</h4> 
<p>Custom resource and consumer name parameter construct: <a href="https://github.com/ran-isenberg/governance-sample-aws-service-catalog/blob/main/cdk/catalog/products/governance_construct.py">https://github.com/ran-isenberg/governance-sample-aws-service-catalog/blob/main/cdk/catalog/products/governance_construct.py</a></p> 
<p>Usage. i.e. CI/CD product creation: <a href="https://github.com/ran-isenberg/governance-sample-aws-service-catalog/blob/main/cdk/catalog/products/cicd_product.py">https://github.com/ran-isenberg/governance-sample-aws-service-catalog/blob/main/cdk/catalog/products/cicd_product.py</a></p> 
<h2>Walkthrough</h2> 
<h3>Prerequisites</h3> 
<p>To deploy the solution, you will need the following prerequisites:</p> 
<ol> 
 <li>AWS account with necessary permissions.</li> 
 <li>AWS CLI installed and configured.</li> 
 <li>Node.js,npm and npx installed.</li> 
 <li>AWS CDK 2.141.0 and higher (Cloud Development Kit) installed.</li> 
 <li>AWS account after running the CDK bootstrap command in the designated region</li> 
 <li>Python 3.12 installed</li> 
 <li>Poetry installed</li> 
 <li>Docker installed</li> 
</ol> 
<h3>Initial setup</h3> 
<p>Clone the code from GitHub onto a local machine:</p> 
<pre><code class="language-bash">Git clone git@github.com:ran-isenberg/governance-sample-aws-service-catalog.git</code></pre> 
<p>To install the packages, utilize a virtual environment:</p> 
<pre><code class="lang-bash">poetry config --local virtualenvs.in-project true
poetry shell
poetry install
</code></pre> 
<p>To prepare the services for deployment, execute the following commands to create the Lambda function layer requirements.txt file and export the function code to the .build folder:</p> 
<pre><code class="language-bash">mkdir -p .build/lambdas ; cp -r catalog_backend .build/lambdas
mkdir -p .build/common_layer ; poetry export --without=dev --format=requirements.txt &gt; .build/common_layer/requirements.txt</code></pre> 
<p>If you are running linux/mac you can run ‘make build’ instead</p> 
<p>To deploy the services with AWS CDK, run the following command:</p> 
<pre><code class="language-bash">npx cdk deploy --app="${PYTHON} ${PWD}/app.py" --require-approval=never</code></pre> 
<p>After deployment, you should see a similar entry to Figure 3 below in the Service Catalog ‘local portfolios’ section under the Administration category.</p> 
<p><img alt="" class="aligncenter size-full wp-image-55903" height="395" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/09/05/Serverless-Governance-portfolios-3.png" width="1999" /></p> 
<p style="text-align: center;"><strong><em>Figure 3: Service Catalog Portfolios</em></strong></p> 
<h3>Account structure</h3> 
<p>In production portfolio scenarios, products will generally be distributed across several accounts. For demonstration, we can use a single account to deploy a portfolio and some products.</p> 
<p>Share portfolio at the organization level or specific accounts.</p> 
<p>Then you set access permissions by clicking on ‘Grant Access’.</p> 
<p><img alt="" class="aligncenter size-full wp-image-55902" height="954" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/09/05/Serverless-Governance-permissions-4.png" width="1999" /></p> 
<p style="text-align: center;"><strong><em>Figure 4: Granting permission to the portfolio</em></strong></p> 
<p>Set permissions to allow your admin to provision it.</p> 
<p>The video <a href="https://www.youtube.com/watch?v=BVSohYOppjk">Share Portfolios Across Accounts in AWS Service Catalog</a> demonstrates these actions:</p> 
<p>You define which users and roles will be able to provision</p> 
<p><img alt="" class="aligncenter size-full wp-image-55904" height="328" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/09/05/Serverless-Governance-users-5.png" width="1999" /></p> 
<p style="text-align: center;"><strong><em>Figure 5: Adding users and roles</em></strong></p> 
<h2>Provisioning a Product</h2> 
<p>Let’s assume that team A’s DevOps member wants to deploy CI/CD pipeline role product.</p> 
<p>We’ll enter the product name ‘Team_A_CICD’ and under ‘consumerName’ parameter, ‘Team A’ and click on ‘Launch Product’.</p> 
<p><img alt="" class="aligncenter size-full wp-image-55901" height="1999" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/09/05/Serverless-Governance-lauching-6.png" width="1571" /></p> 
<p style="text-align: center;"><strong><em>Figure 6: Launching a Service Catalog product</em></strong></p> 
<p>Deployment of a new CloudFormation stack will begin.</p> 
<p>All stack deployments start with a prefix ‘SC’. Here’s the deployed stack below.</p> 
<p><img alt="" class="aligncenter size-full wp-image-55898" height="623" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/09/05/Serverless-Governance-cloudformation-7.png" width="1999" /></p> 
<p style="text-align: center;"><strong><em>Figure 7: CloudFormation stack resulting from product launch</em></strong></p> 
<p>The stack contains the CI/CD role that we wanted to deploy and in addition, the platform engineering custom role that extends the catalog’s visibility.</p> 
<p>Let’s view the visibility table back at the platform’s engineering account.</p> 
<p>We’ll open DynamoDB in the console and view the visibility table.</p> 
<p><img alt="" class="aligncenter size-full wp-image-55900" height="240" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2024/09/05/Serverless-Governance-dynamodb-8.png" width="1740" /></p> 
<p style="text-align: center;"><strong><em>Figure 8: Entry in DynamoDB displaying product details</em></strong></p> 
<p>As expected, a new entry was added upon deployment completion. The entry holds the following information:</p> 
<ul> 
 <li>portfolio id</li> 
 <li>provisioned product stack ARN</li> 
 <li>account id that deployed sqs</li> 
 <li>product (Team’s A account number)</li> 
 <li>product name</li> 
 <li>time of deployment</li> 
 <li>region</li> 
 <li>version</li> 
</ul> 
<h3>Cleaning up</h3> 
<p>To delete the service stack, run the following commands:</p> 
<pre><code class="language-bash">Poetry shell
npx cdk destroy --app="${PYTHON} ${PWD}/app.py" --force</code></pre> 
<p>For those who use mac or linux, you can use ‘make destroy’</p> 
<h2>Conclusion</h2> 
<p>In this blog, we’ve explored a serverless governance solution for platform engineering teams that enhances Service Catalog and its product provisioning capabilities. The solution addresses the platform engineering team’s critical security, visibility, and governance needs.</p> 
<p>We’ve explored how a Service Catalog product portfolio and an extended visibility mechanism improve governance of applications and can help drive best practices for consumers and producers of the software they manage. By leveraging serverless services, the solution offers automated provisioning, consistent deployment practices, and enhanced monitoring across the organization. Make sure to check out the <a href="https://github.com/ran-isenberg/governance-sample-aws-service-catalog">Governance Sample AWS Service Catalog repo</a> for more details.</p>
