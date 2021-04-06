---
title: Azure Purview
date: 2021-03-22 19:38:02
tags:
---

If you don't know what kind of data you have and where it resides, it is difficult to use your data effectively. Azure Purview is here to solve that problem! 

Azure Purview is a unified data governance service that helps you scan and map all your data no matter where the data is stored. Whether this data is on-premise, multi-cloud, or SaaS data.

Azure Purview is an update to the existing Azure Data Catalog Gen 2 service. Azure Data Catalog Gen 2 focused more on managing metadata instead of real data governance and was capable of mapping, searching for, and tagging data within data sources but wasn't able to classify its data. This made it difficult to comply to various data protection laws, such as GDPR. 

Azure Purview helps you enter the following questions:

- What kind of data do I have?
- What tables and fields exist in the database?
- Where is my sensitive data?

# How Does Azure Purview Work?

It all starts with the data. Organisations can have many different data assets such as files, tables, reports and databases. These data sources can exist across different environments, like in the public cloud, on-premise, and SaaS platforms. Azure Purview even supports multi-cloud. Because of this support, you do not have to move data from on-premise or other cloud providers to Azure: everything you want to scan can stay where it is. 

<img src="/images/azure-purview/cloud-model.png" />

You can then connect these different platforms with Azure Purview via the available connectors. 

Purview can scan these data sources to extract different things like metadata, lineage and data classification. The scans operate serverless, so you only pay for what you use. 

When the scanning is done, all the data that is found is published to the Azure Purview Data Map. The Data Map is like a graph describing all the data and relationships between data assets across different environments. The more data you scan, the more powerful and useful this graph becomes!

## Business Context, Classification & Lineage

Once the scan is complete, the data users in your company can start using the data that was just scanned. One thing they can do is add business context to the different fields.

<img src="/images/azure-purview/business-context.png" />

In this example, you will note that you can expand the data within your organization by adding descriptions and context to them. This allows an organisation to clearly define any domain knowledge or business context to their data users.

Secondly, classification is another important part of Azure Purview. It's important for an organisation to be compliant with rules and regulations like GDPR. Azure Purview helps with this by classifying data fields. It uses AI to see what a field contains, and classifies it accordingly. With this knowledge, you know where within your company your sensitive data resides. 

Last but not least, Azure Purview offers data lineage. data lineage lets you see where your data originates from and where your data is headed. The image below shows the flow of data from the ```salesOrderHeader``` table to different areas within the organisation. Some operations are applied to the data, which is then transformed and shown in PowerBI. 

<img src="/images/azure-purview/lineage.png" />

Data lineage in combination with Classification can be really helpful to see where sensitive data is used and whether it should be available in those specific places. 

# **Getting Started With Azure Purview**

To get started, you first need to make sure you are the administrator for your subscription. It is also important to note that you need to enable specific resources for the subscription to be able to use Azure Purview:

- Microsoft.Purview
- Microsoft.EventHub
- Microsoft.Storage

*Not having these enabled could cause some problems down the line.*

Once these are enabled, click on "Create a resource" and search for Azure Purview to get started!

<img src="/images/azure-purview/create-resource.png" />

The Basics tab is pretty straightforward. Provide a valid resource group and name for the Purview account and select the appropriate location for your usage.

<img src="/images/azure-purview/configuration.png" />

In the Configuration tab, there are a few new options. As Azure Purview is still in preview, not much is known about what the platform sizing, catalog options and data insights settings mean (the learn more button only points to the pricing). 

However, knowing that Azure Purview focuses on doing scans, we can conclude that platform size might have something to do with scanning frequency and the number of scans you can perform. 

Under the catalog category, there are two options (which are greyed out due to it being in preview). Here you can select which one is relevant for your implementation. The two options as of writing this blog are:

- Sources registration, automated scans and data discovery
- Business glossary and lineage visualization

The last category, **Data insights**, is about creating statistics and analytics for all the data that you scanned. 

After configuring the settings, you can create the resource. 

*Note: It can take quite a while to create this service.*

## Azure Purview Studio

Azure Purview Studio contains the overview where you can start your scans, look at lineage, add business glossary, and much more.

Azure Purview Studio also has a search field you can use to look through all the assets that have been scanned. To get there, we first have to register some data sources and perform a scan. 

<img src="/images/azure-purview/purview-studio.png" />

### Register Data Sources

To get started registering data sources, either click on the "Register sources" pane or on the database icon on the left side.

The first time you arrive on the sources page, there won't be any collections. Azure Purview uses collections to group different data sources. This gives you the opportunity to, for example, create a specific collection for each of the environments.

Create a collection by clicking on "**+ new collection**". The collection requires you to fill in a name. In this example a collection called "westeuropedatasources" was created in which one could store all data sources available in the West Europe region. 

<img src="/images/azure-purview/collection.png" />

After a collection is created, you can start registering data sources. Currently, because of preview, the number of supported data sources is limited. However, this list is sure to grow quickly and will also include multi–cloud data sources (i.e. giving you the ability to scan data in AWS). 

<img src="/images/azure-purview/datasources.png" />

For now, we will use an Azure SQL Database data source to scan. When selecting the Azure SQL Database to scan, it will give the the option to select a database, which will then be registered under sources. 

<img src="/images/azure-purview/collection-region.png" />

### Scanning a Data Source

To start scanning, press the scan button on the registered data source.

<img src="/images/azure-purview/start-scan.png" />

The configuration that has to entered is pretty straightforward. Under credentials we can select how we want to connect to it. Setting up the connection for Purview MSI will require extensive permissions to a point where you have to give it database owner permission on the database. This gives it full access to the database.

I personally would advice against giving full access to the database and instead create a new managed identity.

Creating a new managed identity will require a name and an authentication method (in the example, SQL authentication was used). No matter what method you choose, it will require you to use Azure Key Vault credentials which prevents you from hardcoding the password in the service itself. This is a convenient security feature they included. 

<img src="/images/azure-purview/create-credential.png" />

After having set up the managed identity and given the managed identity access to the keyvault, it is time to start the scan. Select which tables you want to scan and in which interval you want the scan to be executed. You can also create your own custom rulesets, that is outside the scope of this article. 

<img src="/images/azure-purview/sqldatabase.png" />

After the scan is complete, the details on the database pane will show how many scans have run and how many assets it has scanned. 

## Data catalog

After the data source registrations and scans, we can use the results of the scanned data to know more about the data in our organisation. 

Let's start at the homepage which is accessible to the organisation based on role and account access policies. As we start searching, the data catalog will return metadata about that database we just scanned.  When we enter ```customer```, it shows the result of the asset it found, which in this case is a table about customer addresses. 

<img src="/images/azure-purview/search.png" />

Once you have found something you want to learn more about, you can simply click the result to view the details of the asset.

<img src="/images/azure-purview/metadata.png" />

 The overview page provides a lot of useful metadata as well as a hierarchy on the right, which shows the source of the table. The hierarchy shows where the table came from. In our case, it came from a schema called SalesLT which is located in the database "cr-purview" (which we just scanned) on the server "crpurview.database.windows.net". 

Prior to Azure Purview, the service introduced as Data Catalog allowed users to scan existing data sources (such as a SQL database), which then provided insight into the database tables and the data that was in them.

Azure Purview still uses Azure Data Catalog. Data Catalog enables you to register sources and automate scanning, as well as discover and classify data.

And that's a little taste of what Azure Purview can offer you and how to set it up!

# Conclusion

Azure Purview is a great tool for organisations that struggle to get a grasp on their data. The ability to scan any resources containing data, classifying this data and having one place where your data users can start discovering data is a great addition to the Azure Data landscape.

