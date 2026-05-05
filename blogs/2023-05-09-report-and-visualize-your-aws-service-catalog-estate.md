---
title: "Report and visualize your AWS Service Catalog estate"
url: "https://aws.amazon.com/blogs/mt/report-and-visualize-your-aws-service-catalog-estate/"
date: "Tue, 09 May 2023 00:33:14 +0000"
author: "Raphael Sack"
feed_url: "https://aws.amazon.com/blogs/mt/tag/aws-service-catalog/feed/"
---
<p>AWS Service Catalog allows organizations to create and manage catalog of IT services that are approved for use on AWS. These IT services can include everything from virtual machine images, servers, software, and databases to complete multi-tier application architectures. In addition, organizations can centrally manage deployed IT services, applications, resources, and metadata. This helps you achieve consistent governance to meet your compliance requirements while enabling users to quickly deploy approved IT services.</p> 
<p>To maximize value to their end-users, customers are interested in understanding how the catalog of IT Services (delivered via AWS Service Catalog) are consumed across their AWS accounts. Tracking who is consuming them and how efficiently these are being adopted will help provide insight into business and investment decisions. Hence gaining insights and reporting capabilities will allow customers to determine the impact when driving adoption of delivering services through Service Catalog.</p> 
<p>For example, a customer can use AWS Service Catalog as a vending machine for an application. Consumers within their organisation deploy multiple instances of these application through a self-serve mechanism. In this case, it is important for the customer to understand who is using the application, if an outdated/out of support version of the application is being used, and the adoption rate of the application.</p> 
<p>In this article we will show you how to leverage AWS services to extract, visualize, and report on Service Catalog usage across your AWS accounts.</p> 
<h2>Solution Overview</h2> 
<p><img alt="Solution architecture showing config, data extraction and the Quicksight dashboard in the aggregator/management account, and Config deployed in member/tenant accounts." class="alignnone size-full wp-image-37328" height="675" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/03/06/SCReporting-Fig1.png" width="879" /></p> 
<p style="text-align: center;">Figure 1: Proposed Architecture</p> 
<h2>Prerequisites</h2> 
<p>Before deploying the solution explained in this post, kindly follow the guide below:</p> 
<ul> 
 <li>Prior to starting, we recommend reviewing the up-to-date pricing for Amazon S3, AWS Config, AWS Lambda and Amazon QuickSight.</li> 
 <li>Setup AWS Config in one or more accounts. For setting up AWS Config you can refer to <a href="https://docs.aws.amazon.com/config/latest/developerguide/getting-started.html">Getting Started with AWS Config</a>. If you use <a href="https://aws.amazon.com/organizations/">AWS Organizations</a>, you can enable AWS Config for all accounts using <a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-getting-started-create.html">StackSets</a> with <a href="https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_org_support-all-features.html">all features enabled</a>.</li> 
 <li>Setup Aggregator in the Management Account using <a href="https://docs.aws.amazon.com/config/latest/developerguide/setup-aggregator-console.html">Setting Up Aggregator</a> 
  <ul> 
   <li>AWS recommends delegating administration of AWS Config to a delegated administrator account rather than using the management account, for more information, review the AWS Config documentation</li> 
  </ul> </li> 
 <li><a href="https://docs.aws.amazon.com/quicksight/latest/user/signing-up.html">Sign up for an Amazon QuickSight subscription</a> in the same AWS account where AWS Config Aggregator is enabled</li> 
</ul> 
<h2>Walkthrough</h2> 
<p>As depicted in <em>Figure 1</em>, we are leveraging AWS Config, AWS Lambda, Amazon S3, Amazon QuickSight to deliver a reporting workflow, where the results of each component are used to deliver the overall reporting of Service Catalog usage.</p> 
<ul> 
 <li>Aggregator from AWS Config consolidates the configuration state of AWS resources across the accounts</li> 
 <li>A Lambda function executes SQL queries on AWS Config to extract the Service Catalog configuration state</li> 
 <li>Lambda function stores the usage artifacts to S3 Bucket</li> 
 <li>A QuickSight dataset is set up using a S3 Bucket where the artifacts of the Service Catalog usage are stored</li> 
 <li>A QuickSight analyses is created based on the dataset</li> 
</ul> 
<p>Now let’s go into a bit more detail on each component from the workflow and discuss how the components are tied together.</p> 
<p>AWS Config provides an overview of the configuration of resources in your AWS account (see <a href="https://docs.aws.amazon.com/config/latest/developerguide/resource-config-reference.html">supported resource types in AWS Config</a>). AWS Config has a feature called <a href="https://docs.aws.amazon.com/config/latest/developerguide/querying-AWS-resources.html">advanced queries</a> which enables users to query the current state of resource configuration. In this article we are using queries to extract the state of a Service Catalog portfolio, products, and provisioned products.</p> 
<p>E.g. the following query extracts the list of portfolios:</p> 
<pre><code class="lang-sql">SELECT accountId, configuration.providerName, resourceName, resourceId, supplementaryConfiguration.portfolioAccess.accountIds WHERE resourceType = 'AWS::ServiceCatalog::Portfolio'</code></pre> 
<p>For complete list of properties that can be queried from AWS Config for Service Catalog, refer to:</p> 
<ul> 
 <li><a href="https://github.com/awslabs/aws-config-resource-schema/blob/master/config/properties/resource-types/AWS::ServiceCatalog::Portfolio.properties.json">Portfolios</a></li> 
 <li><a href="https://github.com/awslabs/aws-config-resource-schema/blob/master/config/properties/resource-types/AWS::ServiceCatalog::CloudFormationProduct.properties.json">Products</a></li> 
 <li><a href="https://github.com/awslabs/aws-config-resource-schema/blob/master/config/properties/resource-types/AWS::ServiceCatalog::CloudFormationProvisionedProduct.properties.json">Provisioned Products</a></li> 
</ul> 
<p>You may have a multi-account setup and the adoption of Service Catalog could be across your AWS estate, within an organization or across multiple standalone AWS accounts.</p> 
<p>AWS Config aggregator can collect configuration and compliance data from multiple AWS accounts and Regions into a single account and Region to get a centralized view of the resources. Above mentioned advanced queries can be executed on the aggregator.</p> 
<p><img alt="Config Aggregator overview" class="alignnone size-full wp-image-37327" height="398" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/03/06/SCReporting-Fig2.png" width="998" /></p> 
<p style="text-align: center;">Figure 2: AWS Config Aggregator</p> 
<p>In the proposed solution (<em>Figure 1</em>), we are using a Lambda function which executes a set of SQL queries on an aggregator and extracts the information on Service Catalog Usage across the accounts. The extracted information is stored as an artifact into an S3 bucket in JSON format.</p> 
<p>To visualize and analyze the extracted information, we are using Amazon QuickSight. The stored artifacts in S3 Bucket will be used to create a dataset on QuickSight and the analyses will be performed on the created datasets. The solution described in this article deploys the resources which are shown in the <em>Figure 1</em>, an analyses and a dashboard in QuickSight.</p> 
<h3>Step 1: Pre-deployment Steps</h3> 
<ul> 
 <li>Download the AWS CloudFormation template from <a href="https://d2908q01vomqb2.cloudfront.net/artifacts/MTBlog/cloudops-801/qs-analyses-template-Release.yml">here</a></li> 
 <li>Create a New S3 bucket in the same region where the solution will be deployed (to store usage reports/artifacts) and enable permissions for QuickSight to access this S3 bucket as described <a href="https://docs.aws.amazon.com/quicksight/latest/user/accessing-data-sources.html">here</a></li> 
 <li>Create a User Group on QuickSight (this group can be used to manage permissions to the QuickSight analyses) as described <a href="https://docs.aws.amazon.com/quicksight/latest/user/creating-quicksight-groups.html">here</a></li> 
</ul> 
<h3>Step 2: Deploy the CloudFormation template</h3> 
<p>Creating the Stack using the CloudFormation template requires the following inputs:</p> 
<ol> 
 <li><strong>AWSConfigAggregator</strong> – Name of the Config aggregator to be used as the source for information</li> 
 <li><strong>ReportingBucketName</strong> – S3 Bucket name where reports will be stored; QuickSight must have permissions to this bucket</li> 
 <li><strong>QuickSightIdentityRegion</strong> – Region where the QuickSight is setup</li> 
 <li><strong>QuickSightGroup</strong> – Group name of QuickSight author/admin from default namespace to which the dashboards/analyses will be shared</li> 
 <li><strong>Suffix</strong> (optional) – Use a numeric suffix if you need to create multiple instances of this sample on same AWS account</li> 
</ol> 
<p>Open CloudFormation from AWS Command Console and select “Create stack” → “with new resources”. Upload the “qs-analyses-template-Release.yaml” template from your local machine and provide the inputs to the stack (all inputs are mandatory):</p> 
<p><img alt="Create stack from the provided CloudFormation template" class="alignnone size-full wp-image-37326" height="622" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/03/06/SCReporting-Fig3.png" width="879" /></p> 
<p style="text-align: center;">Figure 3: Create Stack in CloudFormation</p> 
<p><img alt="Specify stack parameters" class="alignnone size-full wp-image-37337" height="700" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/03/06/SCReporting-Fig4-1.png" width="977" /></p> 
<p style="text-align: center;">Figure 4: Create Stack – Parameters</p> 
<p>Deploy the stack after providing the above inputs and once the deployment is deployed successfully, open QuickSight and search for “QSProductAnalysis” in Analysis section. Open analyses to visualize the results.</p> 
<p>Similarly, open Dashboards section in QuickSight and search for “ProductUsage-Dashboard” which allows you to print or download the usage reports.</p> 
<p>The results from the deployment from a test account are as below:</p> 
<p><img alt="QuickSight Dashboard part 1" class="alignnone size-full wp-image-37338" height="270" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/03/06/SCReporting-Fig5-1.png" width="977" /></p> 
<p style="text-align: center;">Figure 5: QuickSight Dashboard Part 1</p> 
<p><img alt="QuickSight Dashboard part 2" class="alignnone size-full wp-image-37339" height="287" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/03/06/SCReporting-Fig6-1.png" width="977" /></p> 
<p style="text-align: center;">Figure 6: QuickSight Dashboard Part 2</p> 
<h3>Step 3: Additional steps</h3> 
<ul> 
 <li>If you are interested to build additional analysis or dashboards depending on your use case, you can leverage the datasets created by the above stack to build your own analysis. You can refer to the AWS documentation on how the develop the <a href="https://docs.aws.amazon.com/quicksight/latest/user/creating-an-analysis.html">analysis</a>.</li> 
 <li>For any additional properties or queries which needs to be included in your dataset, refer to the properties document in the intro of this blog and modify/add the queries to the Lambda function</li> 
 <li>If you do not have a multi-account setup and are interested to capture the Service Catalog usage from a single account, you could leverage the similar approach but query from AWS Config instead of the AWS Config Aggregator</li> 
 <li>To setup an ongoing data refresh, the Lambda function is enabled with a CloudWatch event trigger which updates the usage artifacts every 30 minutes. However, the dataset update has to be enabled manually. To enable data refresh on QuickSight datasets see <a href="https://docs.aws.amazon.com/quicksight/latest/user/refreshing-imported-data.html">https://docs.aws.amazon.com/quicksight/latest/user/refreshing-imported-data.html</a></li> 
</ul> 
<h2>Cleanup</h2> 
<ul> 
 <li>To delete the resources created, you can delete the CloudFormation stack using either the AWS Console or AWS CLI</li> 
 <li>Ensure to empty the <strong>ReportingBucket</strong> S3 bucket before deleting it</li> 
</ul> 
<h2>Summary</h2> 
<p>In this article we discussed how we can use AWS Config aggregators and advanced SQL queries to consolidate the state of Service Catalog resources in a multi-account setup. We further discussed how QuickSight can be used to analyze and visualize the usage results. We deployed a Lambda function which executes the queries and stores the information on S3. The queried information is integrated into QuickSight through Datasets.</p> 
<p>The above approach enables you to keep track of the number of products (catalog of IT services) being used with in your AWS environment. Additionally, it enables you to gain a deeper understanding of the versions of these and consumption of them. All this will assist you in providing the <strong>right</strong> solutions to your end-users, ensure latest versions with features and compliance are being used and more. We look forward to seeing the additional data and reports you generate using the described approach!</p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Raphael" class="wp-image-11636 alignleft" height="160" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/05/16/raphsack.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Raphael Sack</h3> 
  <p style="text-align: left;">Raphael is a technical product manager in the Migration Services organization. He enjoys tinkering with automation and code and active member of the Cloud Operations community.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Sandeep" class="wp-image-11636 alignleft" height="160" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2023/03/15/sndvenk-high-res-current-photo.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Sandeep Kappala Venkat</h3> 
  <p style="text-align: left;">Sandeep is a Cloud Architect at Amazon Web Services, focusing on Application Migration and Modernisation. He enjoys helping customers with their cloud journey and building innovative solutions using AWS to drive their business excellence.</p> 
 </div> 
</footer>
