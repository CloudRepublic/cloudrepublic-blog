---
title: Proof of concept met Azure Batch
date: 2019-05-01 19:54:37
tags:
---

Case omschrijving 
---

Net als in mijn vorige blog post (https://cloudrepublic.github.io/2019/03/28/Proof-of-concept-met-durable-functions/) heb ik een Proof of Concept gemaakt om grote hoeveelheden XML bestanden te transformeren het gaat dan 3000 bestanden per keer met een totale grote van 18 gigabyte. Voor dit artikel kan ik wegens privacy niet de echte data gebruiken en heb ik een dataset gebruikt van https://www.kaggle.com/datasets het gaat hier om een dataset met landen en de wijnen welke uit het desbetreffende land komen. 

Het doel is om de wijnen uit de XML dump te halen en deze om te zetten naar een JSON formaat en deze bestanden te uploaden in een blob container. We krijgen dus per wijn een JSON bestand in een blob container. Dit alles moet gebeuren op basis Azure Batch met een Azure Function als orchestrator

Wat is de opzet van de POC
---
We gaan de xml bestanden transformeren doormiddel van een Azure Batch component en we starten en beheren de Azure Batch doormiddel van een Azure Function.
De Azure Function zal de xml bestanden toevoegen aan een Job in Azure Batch doormiddel van een event gridt rigger (https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-event-grid) en via een httptrigger is de voortgang te zien van het batch proces. 

Wat is Azure Batch
---
Azure Batch is plat gezegt eigenlijk een beheer tool om virtuele machines mee te beheren. Elk van deze machines kan een taak oppakken en dit als input gebruiken voor een commandline applicatie en het resultaat uploaden in een blob container. 

Voor de complete omschrijving wat je met Azure Batch kunt doen verwijs ik je graag door naar de Microsoft site https://azure.microsoft.com/nl-nl/services/batch/

Hoe werkt dit nu eigenlijk allemaal
---

Azure Batch bestaat uit een aantal componenten.

* We hebben een Task, dit is en opdracht welke uitgevoerd dient te worden op een node 
* We hebben een Job, dit is een verzameling van tasks. Aan een Job hangt ook een Pool.
* We hebben een Pool, dit is een verzameling van nodes
* We hebben een Node, dit is een virtuele machine welke de opdracht gaat uitvoeren. 


Ik heb 40 bestanden met landen en wijnen. 1 bestand is ongeveer 75 mb groot. Voor test doeleinde zijn dit dezelfde bestanden met een andere naam. Dit is meer om een gelijkwaardige load op de functie te krijgen als bij de echte POC.

Ik heb een Azure Function welke een event grid trigger heeft welke afgaat op het moment er een bestand wordt geupload in de blobcontainer.
```csharp
        [FunctionName("DuoBatchOrchestrator")]
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
De bestandsnaam wordt uit de url gehaald en door gestuurd naar de methode RunBatch. 


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

Hierna gaan we kijken of er al een job bestaat in het Azure Batch Account. Als er al een job bestaat en hij is nog actief of wordt aangemaakt voegen we hier een task aan toe. Als er geen job actief is of wordt opgestart dan maken we een Job aan. Dit gebeurdt in de *CreateJob* methode. 
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
* We specificeren hoeveel tasks er per node gedraaidt mogen worden in dit geval 4. Er zullen dus 4 tasks per keer op de node worden gestart.
* We specificeren welke applicatie er op de node gedraaidt dient te worden. Dit moet een applicatie of script zijn welke via de commandline te draaien is. Deze applicatie kan als een zip bestand worden gupload in de portal. 
* We zetten de lifetime op *PoolLifetimeOption.Job* dit wil zeggen zodra de alle tasks in de Job klaar zijn zal de pool verwijdert worden en zul je dus ook niet meer betalen voor de virtuele machines. 

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


<!-- Tot zover ziet het er goed en schaalbaar uit tot ik het publiceerde naar een Azure function app.
Ik kopieer met azcopy 40 bestanden van 75 mb in de blob container *wine* en start application insights op om te zien hoe de applicatie zich gedraagt.
```powershell
    AzCopy /Source:https://sourceaccount.blob.core.windows.net/source /Dest:https://destaccount.blob.core.windows.net/wine /SourceKey:key1 /DestKey:key2 /S
``` -->


Demoproject
---
Ik heb een demo project gemaakt welke een volledig werkende solution bevat. 
Zie hieronder de stappen om het durable functions project te starten:
* Clone de repository https://github.com/marcoippel/durablefunctionsdemo
* Maak een storage account aan in Azure
* Maak een functions app aan in Azure en configureer de connectionstring in de appsettings 
* Maak in de blob storage een 3 tal blob containers aan genaamd:
    * source
    * wines
    * wine
* Publiseer het project naar de functions app.
* Maak van het bestand DurableFunctionDemo/winedata.xml 40 kopieën en upload deze naar de wine folder in Azure storage

Start applications insight en kijk hoe de applicatie zich gedraagt.



Conclusie
---
Durable functions op een consumption plan is een hele mooie oplossing maar niet voor een applicatie welke intensief geheugen gebruikt en snel veel bestanden moet verwerken. Je blijft met het geheugen limiet van 1,5 gigabyte en je hebt geen invloed op welke activities er op een functions app worden gehost. 

<img src="/images/durable-functions-storage.png" />

Er zit best wel wat overhead in het proces hij gebruikt namelijk je Azure storage account als queue voor communicatie tussen de orchestrator en de activities. Als je bericht groter is dan in de queue pas plaatst hij het in een blob container. Bijvoorbeeld als de berichten welke naar de activity *A_UploadWine* gaan groter zijn als 64KB (https://docs.microsoft.com/nl-nl/azure/service-bus-messaging/service-bus-azure-and-service-bus-queues-compared-contrasted#capacity-and-quotas) dan zullen deze in een blob container *durablefunctionshub-largemessages* worden opgeslagen om vervolgens op gehaald te worden in de activity *A_UploadWine* en deze activity upload hem dan naar de uiteindelijke blob container. Hier zitten al 2 blob storage acties in welke ook tijd en resources kosten.



Deze POC heeft het doel niet behaald wat we voor ogen hadden en we zullen opzoek gaan naar een andere oplossing. Het idee is om de taken te verdelen over meerdere Azure function apps. Zodra die POC is uitgewerkt zal ik de bevindingen delen in een nieuwe blog.
