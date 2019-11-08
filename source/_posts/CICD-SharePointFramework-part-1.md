---
title: Continuous Integration & Deployment with SharePoint Framework Solutions - Part 1 of 2
date: 2019-10-17 12:15:31
tags: 
- SharePoint Framework
- SPFx
- SharePoint Online
- Office 365
- Frontend
- Webpart
- Continuous Integration
- Continuous Deployment
---
Microsoft heeft goed werk verricht voor ons als SPFx developers om met *gulp serve* snel te kunnen ontwikkelen binnen SharePoint Online. Je kunt lokaal je webpart testen en zelfs verbinden met live data binnen SharePoint Online zonder dat  het beschikbaar is voor anderen.

Als SPFx developer leveren we uiteindelijk ons SharePoint Online maatwerk aan onze klant. We zijn dan gewend de commando's als *gulp bundle -ship* en *gulp package-solution -ship* uit te voeren en te uploaden naar onze App Catalog en te "Implementeren". Maar hiervoor heb je rechten nodig en niet altijd ben je daarvoor bevoegd. Deze verantwoording is vaak belegd bij de applicatiebeheerder die verantwoordelijk is voor een stabiele applicatie.

Hoe fijn zou het zijn als je dit samen met de applicatiebeheerder kunt inregelen via Azure DevOps Continuous Integration en Deployment. Bij de Collaboration Summit in Wiesbaden in mei 2019 werd gedemonstreerd hoe dit ingeregeld kan worden. In deze blog ga ik dit met mijn Visual Studio Enterprise subscription in mijn Office 365 Developer tenant inrichten. Het doel van deel 1 van deze blog is wanneer dingen gecommit worden in de master-branch, dat die code gebuild wordt en de artifact 'myproject.sppkg' te downloaden is. In de volgende blog (Part 2) gaan we de installatie van deze package automatiseren met Continuous Deployment.

**Azure DevOps - Voorbereiding**
Voordat we de pipelines gaan inrichten, hebben we de volgende uitgangspunten:
- Broncode van een SPFx-webpart solution staat in een Git-repository in Azure DevOps.
- De SPFx-webpart solution is buildable zonder fouten.
- Je heb rechten binnen AzureDevOps om een Build-pipeline in te richten.

**Azure DevOps - Maken van de Build-pipeline voor een SharePoint Framework project**
Er zijn verschillende manieren om een build-pipeline op te zetten. Als je voor het eerst begint met build-pipelines, dan is de simpelste manier om de 'classic editor' te gebruiken en hiermee te experimenteren. De definitie en alle wijzigingen die je op de build pipeline maakt staan los van de source code.
De andere manier is d.m.v. een YAML-file die wel naast de source code staat en dus mee moet komen in je commit. In deze blog maken we een SharePoint Framework Build pipeline dmv een YAML-file:

- Ga naar je Azure DevOps project en ga naar 'Pipelines' -> 'Build'.
- Maak een nieuwe Build pipeline aan.
<img src="/images/cicd-sharepointframework/01-newbuildpipeline.png" />
- Kies vervolgens bij 'Where is your code?' voor 'Azure Repos Git' als je code standaard in Azure DevOps staat met Git. 
- Kies vervolgens je Repository.
- Kies vervolgens bij 'Configure your pipeline' voor de optie 'Starter pipeline'.
<img src="/images/cicd-sharepointframework/02-yaml-initialcode.png" />
*Figuur 1: Initiële yaml-code*

- Vervang de initiële code met het volgende:
```
# Deze build start wanneer er een wijziging op de 'master'-branch wordt gedaan.
trigger:
- master

# We gebruiken een hosted VM met Visual Studio 2017 op een Windows 2016 server van Azure
pool:
  vmImage: 'vs2017-win2016'
  
steps:
# Installeer Node 8.x
- task: UseNode@1
  displayName: 'Install Node 8.x'
  inputs:
    version: '8.x'
# Installeer alle packages van het project met Node
- task: Npm@1
  displayName: 'Run ''npm install'''
  inputs:
    command: 'install'
# Gulp-commando: Verzamelen van alle broncode van het SharePoint Framework-project
- task: Gulp@1
  displayName: 'Run ''gulp bundle --ship'''
  inputs:
    gulpFile: 'gulpfile.js'
    targets: 'bundle'
    arguments: '--ship'
    enableCodeCoverage: false
# Gulp-commando: Maak een solution package van het SharePoint Framework-project
- task: Gulp@1
  displayName: 'Run ''gulp package-solution --ship'''
  inputs:
    gulpFile: 'gulpfile.js'
    targets: 'package-solution'
    arguments: '--ship'
    enableCodeCoverage: false
# Publiceer de SharePoint Solution Package onder de artifact naam 'SPFx-myproject'
- task: PublishPipelineArtifact@1
  displayName: 'Publish ''MyProject'' to artifact ''SPFx-myproject'''
  inputs:
    targetPath: 'sharepoint/solution/myproject-webpart.sppkg'
    artifact: 'SPFx-myproject'
```
*Figuur 2: Standaard yaml build pipeline voor 'myproject'*

- Pas indien nodig alle 'myproject' teksten aan naar wens.
- Klik op 'Save & Run'
<img src="/images/cicd-sharepointframework/03-yaml-save-and-run.png" />
*Figuur 3: Save and run build pipeline*

Je ziet hier dat je je wijziging direct in de master-branch kan aanbrengen of apart in een Pull Request in een nieuwe branch. Voor nu kiezen we voor 'Commit directly to the master branch'.
- De build gaat van start:
<img src="/images/cicd-sharepointframework/04-build-running.png" />
*Figuur 4: Azure built nu MyProject-artifact*
- Na enkele minuten is de build klaar en verschijnt rechtsboven in het scherm de knop 'Artifacts'.
Onder deze knop kan de package gedownload worden.
<img src="/images/cicd-sharepointframework/05-build-finished.png" />
*Figuur 5: Artifact is gebuild en te downloaden*

In de volgende blog gaan we deze package installeren via Continuous Deployment.