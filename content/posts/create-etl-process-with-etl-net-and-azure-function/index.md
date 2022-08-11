---
title: 'Create ETL Process With Azure Function and ETL.NET'
date: 2022-07-18T09:30:33+02:00
draft: false
cover:
  image: 'images/_cover.png'
  alt: 'Create ETL Process With Azure Function and ETL.NET'
images:
  - 'images/_cover.png'
tags: ['Azure', 'Azure Function', '.NET', 'C#']
---

> ETL - Extract,Transform, and Load. One of the most important processes in Bussiness Intelligence allowing integrate data from different sources into central repository.

In Microsoft world when you think of ETLs or moving data around, the first thing that comes to your mind is SSIS (SQL Server Integration Services) or Azure Data Factory, it depends if you are cloud first or not :)

Those are robust, battle tested, and not cheap tools to gather and prepare data for analyis or integrate with your system.

Let's imagine you have a case where you have got a file, csv or Excel, with transactions details. Your job is to extract customer details and transaction history and save it in corresponding database tables.

Spinning up Azure Data Factory for this case seems a little overlkill. Azure Logic Apps? Much better choice, in this case it it should be perfectly fine. However, we are developers so we allways tend to write some code. A few lines of code should do the job.

Then you start to dig deeper and think of things like parsing the file, taking care of number and date formatting, removing duplicates, logging, SQL that will edit or insert customer data. You keep adding Nuget packages, cloning culture info, and checking MERGE statement docs (I always do that, no matter how many times I used it). But there is a smart library that does it for you (and many more).

## ETL.NET

It is called [ETL.NET](https://paillave.github.io/Etl.Net/) written by Stéphane Royer [@paillave](https://github.com/paillave). As you can read on their website it is a data processing framework for .NET supporting many popular data sources, and helps you with common data processing tasks like normalization, join, and upsert. Additionaly, it comes with tracing and error tracking bundled. Basically, everything that you need to move data around places.

## Serverless ETL

I am in love with Azure Functions. This simple, event-based programming model usually ticks all the boxes when developing modern apps. You can create a function triggered by a file uploaded to an Azure Blob Storage container, scheduled timer, or on demand with HTTP request. Especially, the first one looks interesting in terms of a data integration.

The one think that you need to take into consideration is processing time. When you will need to process a big number of rows you might be caught by a timetout. So that you should carefully choose a [hosting option](https://docs.microsoft.com/en-us/azure/azure-functions/functions-scale#timeout) and set the ```functionTimeout``` setting in ```host.json``` file.

## Creating ETL Function