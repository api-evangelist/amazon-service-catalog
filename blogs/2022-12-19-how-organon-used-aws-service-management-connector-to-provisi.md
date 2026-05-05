---
title: "How Organon used AWS Service Management Connector to provision AWS resources from Service Now across multiple AWS accounts."
url: "https://aws.amazon.com/blogs/mt/how-organon-co-used-aws-service-management-connector-to-provision-aws-resources-from-service-now-across-multiple-aws-accounts/"
date: "Mon, 19 Dec 2022 17:44:46 +0000"
author: "Chandra Sekhar Chappa"
feed_url: "https://aws.amazon.com/blogs/mt/tag/aws-service-catalog/feed/"
---
<p><a href="https://www.organon.com/">Organon</a> has been exploring Amazon Web Services (AWS) to provide a simple, efficient way to their end users to easily provision cloud infrastructure across multiple accounts and regions. Additionally, they needed to ensure security, management, governance and compliance on the AWS services to follow <a href="https://aws.amazon.com/compliance/gxp-part-11-annex-11/">GxP regulations</a>.</p> 
<p>Organon uses <a href="https://www.servicenow.com/">ServiceNow</a> as the enterprise IT Service Management platform for end-user provisioning and they wanted to have capabilities for its users to provision AWS resources from <a href="https://www.servicenow.com/">ServiceNow</a>&nbsp; <a href="https://aws.amazon.com/servicecatalog/">Service Catalog</a>. They also want to extend the provisioning capabilities for other AWS services that are in scope.</p> 
<h2>How Organon made it work–Solution Overview</h2> 
<p>Organon engaged AWS Professional Services to build <a href="https://aws.amazon.com/servicecatalog/">AWS Service Catalog</a> products and portfolios that can create, organize, and govern a curated catalog of AWS services. With different permissions levels separated from requestors, we can share the product catalog with end users to quickly provision pre-approved resources without needing direct access to the underlying AWS services. <a href="https://aws.amazon.com/servicecatalog/">AWS Service Catalog</a> can further integrate with <a href="https://aws.amazon.com/service-management-connector/">AWS Service Management Connector</a> that enables end users to provision AWS resources from <a href="https://www.servicenow.com/">ServiceNow</a>. The plugin is available at no charge in the ServiceNow <a href="https://store.service-now.com/">store</a> and the integration is available in all AWS regions where <a href="https://aws.amazon.com/servicecatalog/">AWS Service Catalog</a> is available.</p> 
<p><img alt="Figure 1. Architecture Diagram for AWS Service Catalog implementation with AWS Service Management Connector" class="wp-image-35120 aligncenter" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/12/19/image-11.png" /></p> 
<p>Organon’s cloud management team identified the most frequently used AWS services by its end users for their application management. AWS Professional Services is tasked to build in Golden templates for the identified AWS services.</p> 
<p>We built golden templates using <a href="https://aws.amazon.com/cloudformation/">AWS CloudFormation</a> for services in scope and include specific security and regulatory compliance requirements. <a href="https://aws-service-catalog-factory.readthedocs.io/en/latest/">AWS Service Catalog Framework Factory</a> converts each golden template to products in AWS Service Catalog. <a href="https://aws-service-catalog-puppet.readthedocs.io/en/latest/">AWS Service Catalog Framework Puppet</a> then shares the products from the central account to multiple accounts/regions. We also built <a href="https://aws.amazon.com/config/">AWS Config</a> &amp; RAPIDQ custom solution for monitoring and alerting security findings. This implementation approach for AWS Service Catalog enables Organon to scale and centrally manage catalog of AWS services.</p> 
<p>As a result, the <a href="https://aws.amazon.com/service-management-connector/">AWS Service Management Connector</a> powered by <a href="https://aws.amazon.com/servicecatalog/">AWS Service Catalog</a> is now configured in Organon’s ServiceNow instance. It periodically synchronizes available <a href="https://aws.amazon.com/servicecatalog/">AWS Service Catalog</a> products in AWS accounts to <a href="https://www.servicenow.com/products/it-service-automation-applications/service-catalog.html">ServiceNow Service Catalog</a>.</p> 
<p>ServiceNow administrators then provide secured and governed <a href="https://aws.amazon.com/servicecatalog/">AWS Service Catalog</a> products and portfolios to applicable end users. <a href="https://aws.amazon.com/service-management-connector/">AWS Service Management Connector</a> provides Organon’s end users self-service mechanisms to browse and request vetted AWS services that can track the lifecycle of provisioned resources from their familiar ITSM platform <a href="https://www.servicenow.com/">ServiceNow</a>. It also enables them to take post provisioning actions that can update or terminate the provisioned products.</p> 
<h2>Summary</h2> 
<p>Organon now offers an automated self-service solution to end users to provision pre-approved products in authorized accounts from ServiceNow, while maintaining the products configuration templates in central account. It also enhances security and creates a repeatable foundation in the future using AWS Service Catalog and AWS Service Management Connector.</p> 
<p>Centralized cloud management teams can use this approach to curate the battle-tested, repeatable, predictable and best-practices based software-infrastructure blueprints, and offer those enterprise-wide for easy, self-service adoption as Service Catalog products.</p> 
<p>We have built this in collaboration with Organon team, as they design and build AWS solutions in their cloud adoption journey.</p> 
<p><strong>About the authors:</strong></p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="aligncenter size-full wp-image-11636" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/12/13/yay.png" />
  </div> 
  <h3 class="lb-h4">Shylu D</h3> 
  <p>Shylu is an AWS Professional Services DevOps Consultant from Connecticut. She enjoys to explore and handle challenging solutions. Continuous learner and believe in “Never stop learning, as life never stops teaching” – Buddha.<br /> Outside of office work, she likes to workout, dance, and enjoys driving with family.</p> 
 </div> 
</footer> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="aligncenter wp-image-11636" src="https://d2908q01vomqb2.cloudfront.net/972a67c48192728a34979d9a35164c1295401b71/2022/12/13/cchapra.png" />
  </div> 
  <h3 class="lb-h4">Chandra Chappa</h3> 
  <p>Chandra Chappa is a Denver based Sr. Service Management Specialist with AWS Service Management Connector. Chandra enjoys helping customers enable end-to-end IT lifecycle management to AWS Field, Customers, and Solutions Architect Partners. In his free time, he likes playing local club cricket and enjoys spending time with family and friends.</p> 
 </div> 
</footer>
