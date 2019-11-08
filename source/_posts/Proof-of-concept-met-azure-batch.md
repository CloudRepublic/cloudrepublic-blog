---
title: Proof of concept met Azure Batch
date: 2019-06-27 19:54:37
tags:
---

Case omschrijving 
---

Net als in mijn vorige [blog post](https://cloudrepublic.github.io/2019/03/28/Proof-of-concept-met-durable-functions/) heb ik een Proof of Concept gemaakt om grote hoeveelheden XML bestanden te transformeren, het gaat dan 3000 bestanden per keer met een totale grote van 18 gigabyte. Voor dit artikel kan ik wegens privacy redenen niet de echte data gebruiken en heb ik een dataset gebruikt van https://www.kaggle.com/datasets. Het gaat hier om een dataset van landen en de wijnen.

Het doel is om de wijnen uit de XML dump te halen en deze om te zetten naar een JSON formaat en deze bestanden te uploaden in een blob container. We krijgen dus per wijn een JSON bestand in een blob container. Dit alles moet gebeuren op basis Azure Batch met een Azure Function als orchestrator.

Wat is de opzet van de POC
---
We gaan de XML bestanden transformeren doormiddel van een Azure Batch component en we starten en beheren de Azure Batch doormiddel van een Azure Function.
De Azure Function zal de XML bestanden toevoegen aan een Job in Azure Batch doormiddel van een [event grid trigger](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-event-grid) en via een httptrigger is de voortgang te zien van het batch proces. 

<img src="/images/azure-batch-overview.png" />

Wat is Azure Batch
---
Azure Batch is plat gezegd eigenlijk een beheer tool voor virtuele machines. Elk van deze machines kan een taak oppakken en dit als input gebruiken voor een commandline applicatie en het resultaat uploaden in bijvoorbeeld een blob container. 

Voor de complete omschrijving wat je met Azure Batch kunt doen verwijs ik je graag door naar de Microsoft site https://azure.microsoft.com/nl-nl/services/batch/

Hoe werkt het Proof of Concept
---

Azure Batch bestaat uit een aantal componenten.

* Een Task is een opdracht welke uitgevoerd dient te worden op een node. 
* Een Job is een verzameling van tasks. Aan een Job hangt ook een Pool.
* Een Pool is een verzameling van nodes.
* Een Node is een virtuele machine welke een van de tasks gaat uitvoeren. 

Ik heb 40 bestanden met landen en wijnen. 1 bestand is ongeveer 75 mb groot. Voor testdoeleinden zijn dit dezelfde bestanden met een andere naam. Dit is meer om een gelijkwaardige load op de functie te krijgen als bij de echte POC.

Ik heb een Azure Function welke een event grid trigger heeft welke afgaat op het moment dat er een bestand wordt geupload in de blobcontainer.
```csharp
        [FunctionName("BatchOrchestrator")]
        public static async Task Run(
           [EventGridTrigger] EventGridEvent eventGridEvent,
           ExecutionContext context,
           ILogger log)
        {
           var data = JsonConvert.DeserializeObject<StorageBlobCreatedEventData>(eventGridEvent.Data.ToString());
           string name = data.Url.Split('/').Last();

           log.LogInformation($"C# Blob trigger function Processed blob\n Name:{name}");
           await RunBatch(log, context, name);
        }
```
De bestandsnaam wordt uit de URL gehaald en doorgestuurd naar de methode RunBatch. 
De methode RunBatch initialiseert een BatchClient met de credentials welke opgegeven zij in de config.
```csharp
        private static async Task RunBatch(ILogger log, ExecutionContext context, string name)
        {
            var config = ReadSettings(context);

            BatchSharedKeyCredentials credentials = new BatchSharedKeyCredentials(config["BatchUrl"], config["BatchAccount"], config["BatchKey"]);
            using (BatchClient batchClient = BatchClient.Open(credentials))
            {
            }
        }
```

Hierna gaan we kijken of er al een job bestaat in het Azure Batch Account. Als er al een job bestaat en hij is nog actief of wordt aangemaakt voegen we hier een task aan toe. Als er geen job actief is of wordt opgestart dan maken we een Job aan. Dit gebeurt in de *CreateJob* methode. 
```csharp
        private static async Task RunBatch(ILogger log, ExecutionContext context, string name)
        {

            var config = ReadSettings(context);

            BatchSharedKeyCredentials credentials = new BatchSharedKeyCredentials(config["BatchUrl"], config["BatchAccount"], config["BatchKey"]);
            using (BatchClient batchClient = BatchClient.Open(credentials))
            {
                string jobId = string.Empty;

                try
                {
                    batchClient.CustomBehaviors.Add(RetryPolicyProvider.ExponentialRetryProvider(TimeSpan.FromSeconds(5), 3));

                    if (batchClient.JobOperations.ListJobs().Any())
                    {
                        var jobs = batchClient.JobOperations.ListJobs();
                        CloudJob activeJob = jobs.FirstOrDefault(job => job.State == JobState.Active || job.State == JobState.Enabling);

                        if (activeJob != null)
                        {
                            log.LogDebug("Job still active");
                            CreateTaskIfNotExists(batchClient, activeJob.Id, name);
                        }
                        else
                        {
                            jobId = await CreateJob(name, batchClient);
                        }
                    }
                    else
                    {
                        jobId = await CreateJob(name, batchClient);
                    }
                }
                catch (Exception e)
                {
                    log.LogError(e, e.Message);
                    if (!string.IsNullOrEmpty(jobId))
                    {
                        log.LogDebug($"Deleting job: {jobId}");
                        await batchClient.JobOperations.DeleteJobAsync(jobId);
                    }
                }
            }
        }
```

Een Job heeft een pool met nodes nodig welke het werk uitvoeren. We gaan deze dus eerst aanmaken.

* We specificeren het besturingssysteem in dit geval staat de waarde *5* voor *Windows Server 2016* voor de overige waardes check de Azure Guest OS Releases https://docs.microsoft.com/en-us/azure/cloud-services/cloud-services-guestos-update-matrix#releases
* We specificeren een virtuele machine size in dit geval een *standard_d1_v2* voor de overige ondersteunde machines check https://docs.microsoft.com/en-us/azure/batch/batch-pool-vm-sizes#supported-vm-families-and-sizes
* We specificeren hoeveel tasks er per node gedraaid mogen worden in dit geval 4. Er zullen dus 4 tasks per keer op de node worden gestart.
* We specificeren welke applicatie er op de node gedraaid dient te worden. Dit moet een applicatie of script zijn welke via de commandline te draaien is. Deze applicatie kan als een zip bestand worden gupload in de portal. 
* We zetten de lifetime op *PoolLifetimeOption.Job* dit wil zeggen zodra alle tasks in de Job klaar zijn zal de pool verwijdert worden en zul je dus ook niet meer betalen voor de virtuele machines. 

```csharp
        private static PoolInformation CreatePool()
        {
            return new PoolInformation()
            {
                AutoPoolSpecification = new AutoPoolSpecification()
                {
                    AutoPoolIdPrefix = "Wine",
                    PoolSpecification = new PoolSpecification()
                    {
                        CloudServiceConfiguration = new CloudServiceConfiguration("5"),
                        VirtualMachineSize = "standard_d1_v2",
                        MaxTasksPerComputeNode = 4,
                        ApplicationPackageReferences = new List<ApplicationPackageReference>()
                        {
                            new ApplicationPackageReference()
                            {
                                ApplicationId = "converter"
                            }
                        }
                    },
                    KeepAlive = false,
                    PoolLifetimeOption = PoolLifetimeOption.Job
                }
            };
        }
```

Nu we een Pool hebben kunnen we deze koppelen aan de Job. De *CreateJob* methode maakt een unieke naam aan voor de job doormiddel van een timestamp te prefixen met "WineConverter".

```csharp
        private static async Task<string> CreateJob(string name, BatchClient batchClient)
        {
            string jobId;

            CloudJob activeJob;
            jobId = CreateJobId("WineConverter");

            //create pool
            PoolInformation pool = CreatePool();

            // create a job
            activeJob = await CreateJobAsync(batchClient, jobId, pool);

            //create tasks from the blobs
            CreateTaskIfNotExists(batchClient, jobId, name);

            await activeJob.RefreshAsync();
            activeJob.OnAllTasksComplete = OnAllTasksComplete.TerminateJob;
            await activeJob.CommitChangesAsync();

            return jobId;
        }
```

We controlleren eerst of de task al is aangemaakt en is toegevoegd aan de job, zoniet dan voegen we hem toe. De naam van de task mag alleen letters en cijfers bevatten en een koppelteken en underscore.

Ook stellen we in welke package de task moet starten met welke argumenten. Hier starten we converter.exe met als argument een naam van de blob (in dit geval een XML bestand met wijn data).

De converter.exe bevat alle logica om de xml te verwerken en het resultaat te uploaden in een Azure blob container.
In de Azure portal kan je een package uploaden welke op de nodes geinstalleerd moeten worden. Meer informatie over hoe packages werken met Azure batch is te vinden op: https://docs.microsoft.com/en-us/azure/batch/batch-application-packages

```csharp
        private static void CreateTaskIfNotExists(BatchClient batchClient, string jobId, string blobName)
        {
            IPagedEnumerable<CloudTask> tasks = batchClient.JobOperations.ListTasks(jobId);
            string taskId = $"task_{SanitizeString(blobName)}";

            if (tasks.Any(x => x.Id == taskId))
            {
                Console.WriteLine($"Task with id: {taskId} all ready exists");
                return;
            }

            batchClient.JobOperations.AddTask(jobId, new CloudTask(taskId, $"cmd /c %AZ_BATCH_APP_PACKAGE_CONVERTER%\\converter.exe -name {blobName}"));
        }
```

De volledige code
---

Zie hieronder de volledige code. Er is een extra function aan toegevoegd met een httptrigger zodra je deze aanroept worden alle jobs met de bijhorende tasks weergegeven en de status van de tasks. Het endpoint is nu niet beveiligd maar dit is omdat het een demo is.

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Net.Http;
using System.Threading.Tasks;
using Microsoft.Azure.Batch;
using Microsoft.Azure.Batch.Auth;
using Microsoft.Azure.Batch.Common;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using JobState = Microsoft.Azure.Batch.Common.JobState;

namespace WineConverter
{
    public static class BatchOrchestrator
    {
        [FunctionName("BatchOrchestrator")]
        public static async Task Run(
           [EventGridTrigger] EventGridEvent eventGridEvent,
           ExecutionContext context,
           ILogger log)
        {
           var data = JsonConvert.DeserializeObject<StorageBlobCreatedEventData>(eventGridEvent.Data.ToString());
           string name = data.Url.Split('/').Last();

           log.LogInformation($"C# Blob trigger function Processed blob\n Name:{name}");
           await RunBatch(log, context, name);
        }

        private static async Task RunBatch(ILogger log, ExecutionContext context, string name)
        {

            var config = ReadSettings(context);

            BatchSharedKeyCredentials credentials = new BatchSharedKeyCredentials(config["BatchUrl"], config["BatchAccount"], config["BatchKey"]);
            using (BatchClient batchClient = BatchClient.Open(credentials))
            {
                string jobId = string.Empty;

                try
                {
                    batchClient.CustomBehaviors.Add(RetryPolicyProvider.ExponentialRetryProvider(TimeSpan.FromSeconds(5), 3));

                    if (batchClient.JobOperations.ListJobs().Any())
                    {
                        var jobs = batchClient.JobOperations.ListJobs();
                        CloudJob activeJob = jobs.FirstOrDefault(job => job.State == JobState.Active || job.State == JobState.Enabling);

                        if (activeJob != null)
                        {
                            log.LogDebug("Job still active");
                            CreateTaskIfNotExists(batchClient, activeJob.Id, name);
                        }
                        else
                        {
                            jobId = await CreateJob(name, batchClient);
                        }
                    }
                    else
                    {
                        jobId = await CreateJob(name, batchClient);
                    }
                }
                catch (Exception e)
                {
                    log.LogError(e, e.Message);
                    if (!string.IsNullOrEmpty(jobId))
                    {
                        log.LogDebug($"Deleting job: {jobId}");
                        await batchClient.JobOperations.DeleteJobAsync(jobId);
                    }
                }
            }
        }

        [FunctionName("BatchStatus")]
        public static string BatchStatus([HttpTrigger(AuthorizationLevel.Anonymous, "get")]HttpRequestMessage req, ExecutionContext context, ILogger log)
        {
            List<Job> jobList = new List<Job>();

            var config = ReadSettings(context);
            BatchSharedKeyCredentials credentials = new BatchSharedKeyCredentials(config["BatchUrl"], config["BatchAccount"], config["BatchKey"]);
            using (BatchClient batchClient = BatchClient.Open(credentials))
            {
                IPagedEnumerable<CloudJob> jobs = batchClient.JobOperations.ListJobs();
                if (jobs.Any())
                {

                    foreach (CloudJob cloudJob in jobs.ToList())
                    {
                        jobList.Add(new Job()
                        {
                            Id = cloudJob.Id,
                            Name = cloudJob.DisplayName,
                            Status = cloudJob.State.ToString(),
                            Tasks = cloudJob.ListTasks().Select(task => new JobTask()
                            {
                               Id = task.Id,
                               Name = task.DisplayName,
                               Status = task.State.ToString()
                            }).ToList()
                        });
                    }
                }
            }

            return JsonConvert.SerializeObject(jobList, Formatting.Indented);
        }

        private static async Task<string> CreateJob(string name, BatchClient batchClient)
        {
            string jobId;

            CloudJob activeJob;
            jobId = CreateJobId("WineConverter");

            //create pool
            PoolInformation pool = CreatePool();

            // create a job
            activeJob = await CreateJobAsync(batchClient, jobId, pool);

            //create tasks from the blobs
            CreateTaskIfNotExists(batchClient, jobId, name);

            await activeJob.RefreshAsync();
            activeJob.OnAllTasksComplete = OnAllTasksComplete.TerminateJob;
            await activeJob.CommitChangesAsync();

            return jobId;
        }

        private static PoolInformation CreatePool()
        {
            return new PoolInformation()
            {
                AutoPoolSpecification = new AutoPoolSpecification()
                {
                    AutoPoolIdPrefix = "Wine",
                    PoolSpecification = new PoolSpecification()
                    {
                        CloudServiceConfiguration = new CloudServiceConfiguration("5"),
                        VirtualMachineSize = "standard_d1_v2",
                        MaxTasksPerComputeNode = 4,
                        ApplicationPackageReferences = new List<ApplicationPackageReference>()
                        {
                            new ApplicationPackageReference()
                            {
                                ApplicationId = "converter"
                            }
                        }
                    },
                    KeepAlive = false,
                    PoolLifetimeOption = PoolLifetimeOption.Job
                }
            };
        }

        private static async Task<CloudJob> CreateJobAsync(BatchClient batchClient, string jobId, PoolInformation pool)
        {
            CloudJob unboundJob = batchClient.JobOperations.CreateJob();
            unboundJob.Id = jobId;

            unboundJob.PoolInformation = pool;
            await unboundJob.CommitAsync();

            return unboundJob;
        }

         private static void CreateTaskIfNotExists(BatchClient batchClient, string jobId, string blobName)
        {
            IPagedEnumerable<CloudTask> tasks = batchClient.JobOperations.ListTasks(jobId);
            string taskId = $"task_{SanitizeString(blobName)}";

            if (tasks.Any(x => x.Id == taskId))
            {
                Console.WriteLine($"Task with id: {taskId} all ready exists");
                return;
            }

            batchClient.JobOperations.AddTask(jobId, new CloudTask(taskId, $"cmd /c %AZ_BATCH_APP_PACKAGE_CONVERTER%\\converter.exe -name {blobName}"));
        }

        private static string CreateJobId(string prefix)
        {
            return $"{prefix}-{DateTime.Now:yyyyMMdd-HHmmss}";
        }

        private static string SanitizeString(string text)
        {
            string pattern = @"[^A-Za-z0-9-_]";
            return System.Text.RegularExpressions.Regex.Replace(text, pattern, string.Empty);
        }

        private static IConfigurationRoot ReadSettings(ExecutionContext context)
        {
            return new ConfigurationBuilder()
                .SetBasePath(context.FunctionAppDirectory)
                .AddJsonFile("local.settings.json", optional: true, reloadOnChange: true)
                .AddEnvironmentVariables()
                .Build();


        }
    }

    public class JobTask
    {
        public string Id { get; set; }
        public string Name { get; set; }
        public string Status { get; set; }
    }

    public class Job
    {
        public string Id { get; set; }
        public string Name { get; set; }
        public string Status { get; set; }
        public List<JobTask> Tasks { get; set; }
    }
}

```

Conclusie
---
Azure batch is echt een serieuze keuze als je grote hoeveelheden data moet verwerken en je wilt in controle zijn wat er allemaal gebeurt.
Je kan zowel horizontaal als verticaal schalen en het aantal nodes wat het werk kan doen is standaard 20 maar je kunt een request doen voor meer nodes. Voor de recource limieten check de documentatie https://docs.microsoft.com/en-us/azure/batch/batch-quota-limit.

Ik ken zelf weinig projecten waar ze Azure batch gebruiken maar ik ben echt onder de indruk hoe simpel en krachtig Azure batch is plus je betaald alleen voor de tijd dat de nodes ook echt iets doen dus geen vaste maandelijkse kosten.

Deze POC heeft het qua performance zijn doel wel behaald wat we voor ogen hadden alleen het bevat toch net te veel stappen om het volledig via CI/CD gemakkelijk te deployen. 

In een volgende blog zal ik de uiteindelijke oplossing uitwerken met verschillende functie apps op een consumption plan welke aan alle eisen voldoet.