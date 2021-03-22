---
title: Azure Purview
date: 2021-03-22 19:38:02
tags:
---

If you don't know what kind of data you have and where it resides, it's difficult to use it. Azure Purview can help with that! 

Azure Purview is a unified data governance service that helps you scan and map all your data no matter where the data is stored! tThis data could be on-premise, multi-cloud, or SaaS data.

Azure Purview is an update to the existing Azure Data Catalog Gen 2 service. Azure Data catalog Gen 2 focused more on managing metadata instead of real data governance and was capable of mapping, searching for, and tagging data within data sources but wasn't able to classify its data.This made it hard to be compliant to different laws pertaining data protection, like GDPR. 

Azure Purview allows you to answer the following questions:

- What kind of data do I have?
- What are all the different tables and fields in the database? 
- Where is any sensitive data? 

# How does Azure Purview work?

It all starts with the data. Organisations can have many different data assets like files, tables,reports and databases. These data sources can exist across different environments, like in the public cloud, on-premise, and SaaS environments. Azure Purview even supports multi-cloud. Because of this support, you do not have to move data from on-premise or other cloud providers to Azure: everything you want to scan can stay where it is. 

<img src="/images/azure-purview/cloud-model.png" />

You can then connect these different environments with Azure Purview via the different connectors that are available. 

Purview can scan these data sources to extract different things like metadata, lineage and data classification. The scans operate serverless, so you only pay for what you use. 

When the scanning is done, all the data that is found is published to the Azure Purview datamap. The data map is like a graph describing all the data and relationships between data assets across different environments.The more data you scan, the more powerful and useful this graph becomes!

## Business Context, Classification & Lineage

Once the scan is complete, the data users in your company can start using the data that was just scanned. One thing they can do is add business context to the different fields

<img src="/images/azure-purview/business-context.png" />

In the example you can see that you can expand on the data within your organistation by adding descriptions and context to them. This allows an organisation to define any domain knowledge or business context to their data users in a clear way. 

Secondly, Classification is another important part of Azure Purview. It's important for an organisation to be compliant with rules and regulations like GDPR. Azure Purview helps with this by classifying data fields. It uses AI to see what a field contains, and classifies it accordingly. With this knowledge, you know where within your company your sensitive data resides. 

Last but not least, Azure Purview offers Data Lineage. Data Lineage lets you see where your data comes from and where you're data is going to. The image below shows the flow of data from the salesOrderHeader table to different areas within the organisation. Some operations are applied to the data, which is then transformed and shown in PowerBI. 

<img src="/images/azure-purview/lineage.png" />

Data Lineage in combined with Classification can be really helpful to see where sensitive data is used and whether it should be available in those specific places. 

# **How to get started with Azure Purview**

To get started, you first need to make sure you are the administrator for your subscription. It is also important to note that you need to enable specific resources for the subscription to be able to use Azure Purview:

- Microsoft.Purview
- Microsoft.EventHub
- Microsoft.Storage

*Not having these enabled could cause some problems down the line.*

Once these are enabled, Click on "Create a resource", search for Azure Purview to get started!

<img src="/images/azure-purview/create-resource.png" />

The Basic tab is pretty straightforward. Provide a valid resource group and name for the Purview account and select the appropriate location suitable for your usage.

<img src="/images/azure-purview/configuration.png" />

In the configuration tab, there are a few new options. As Azure Purview is still in preview, not much is known about what the platform sizing, catalog options and/or data insights settings mean (the learn more button only points to the pricing). 

But, knowing that Azure Purview focuses on doing scans, platform size might have to do with scanning frequency and the number of scans you can perform. 

Under the **catalog** category, there are two options (which are greyed out due to it being in preview). Here you can select which one is relevant for your implementation. The two options as of writing this blog are:

- Sources registration, automated scans and data discovery
- Business glossary and lineage visualization

The last category, **Data insights**, is about creating statistics and analytics for all the data that you scanned. 

After selecting the correct settings, you can create the resource. 

*Note: It can take quite a while to create this service.*

## Azure Purview Studio

The Azure Purview Studio is the overview on which you can start your scans, look at lineage add business glossary and much more.

Azure Purview Studio contains a search field in which you can look through all the Assets that have been scanned. To get there, we first have to register some data sources and perform a scan. 

<img src="/images/azure-purview/purview-studio.png" />

### Register datasources

To get started with registering datasources, either click on the "Register sources" pane or on the database icon on the left side.

The first time you arrive on the sources page, there wont be any collections. Azure Purview uses collections to group different datasources. This gives you the opportunity to, for example, create a specific collection for each of the environments.

Start out by creating a collection by clicking on "**+ new collection**". The collection requires you to fill in a name. In this example a collection called "westeuropedatasources" was created in which one could store all data sources available in the West Europe region. 

<img src="/images/azure-purview/collection.png" />

After a collection is created, you can start registering data sources. Currently, because of preview the number of supported data sources is limited. However, this list is sure to grow quickly and will also include multi –cloud data sources (i.e. giving you the ability to scan data in AWS) 

<img src="/images/azure-purview/datasources.png" />

For now, we will use the Azure SQL Database to scan. When selecting the Azure SQL Database to scan, it will give the the option to select a database, which will then be registered under sources. 

<img src="/images/azure-purview/collection-region.png" />

### Scanning datasource

To start scanning, press the scan button on the registered datasource.

<img src="/images/azure-purview/start-scan.png" />

The data that has to be filled in is pretty straightforward. Under the credentials, we can select how we want to connect to it. When setting it up for the Purview MSI, it requires extensive permissions to a point where you have to give it database owner permission on the database. This gives it full access to the database.

If this is something that you dont want, you can create a new managed identity.

To create a new managed identity, it requires a name, what kind of authentication method it should use (we selected SQL authentication). No matter what method you use, it will require you to use Azure Keyvault credentials which make sure you don’t hardcode the password anywhere. This is a very nice security feature they included. 

<img src="/images/azure-purview/create-credential.png" />

After having set up the managed identity and given the managed identity access to the keyvault, it is time to start the scan. Select which tables you want to scan and in which interval you want the scan to be executed. You can also create your own custom rulesets, that is outside the scope of this article. 

<img src="/images/azure-purview/sqldatabase.png" />

After the scan is complete, the details on the database pane will show how many scans have run and how many assets it has scanned. 

## Data catalog

After the data source registrations and scans, we can use the results of the scanned data to know more about the data in our organisation. 

We can start on the homepage, which is accessible to the organisation (access can be given to accounts, based on role). When we start searching you can see that we start to see data about the database we just scanned. When typing in customer, it shows the result of the asset it found, which in this case is a table about customer addresses. 

<img src="/images/azure-purview/search.png" />

Once you have found something you want to know more of, you can simply click the result to show more details about the asset

<img src="/images/azure-purview/metadata.png" />

On the overview page, you get to see a lot of useful metadata as well as the hierarchy on the right. The hierarchy shows where the table came from. In our case, it came from a schema called SalesLT which is located in the database "cr-purview" (which we just scanned) on the server "crpurview.database.windows.net". 

Something that already existed before Azure Purview was introduced as Data catalog. As described earlier, this gave a way to do scans on existing data sources, like a SQL database. It would then show what tables this database contained and what those tables contained. 

Azure Purview still uses Azure Data Catalog. Data Catalog enables you to register sources, automated scanning and classification of data as well as data discovery.

And thats a little taste of what Azure Purview can offer you and how to set it up!

# Conclusion

Azure Purview is a great tool for organisations that struggle to get a hold of their data. The ability to scan any resources containing data, classifying this data and having one place where your data users can start discovering data is a great addition to the Azure Data Landscape.

