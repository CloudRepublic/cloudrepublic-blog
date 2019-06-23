---
title: Getting started with SharePoint Framework
date: 2019-06-14 16:15:35
tags: 
- SharePoint Framework
- SPFx
- SharePoint Online
- Office 365
- React
- Gulp
- Workbench
- Frontend
- Webpart
---

Voor een klant zijn we begonnen met het bouwen van hun eerste SharePoint Online webpart. Binnen een Citrix omgeving waar we een Developer VM ter beschikking hebben, moeten we eerst een ontwikkelomgeving opzetten. Voor SharePoint on-premises developers is het een enorme omslag hoe dingen nu ontwikkeld worden met Javascript. Je krijgt te maken met NodeJS, NPM, gulp, Yeoman, etc. om maar wat termen te noemen. Als je ervaring hebt met frontend-ontwikkelen, dan ligt de moeilijkheid in het begrijpen van de (misschien onlogische) concepten van SharePoint Online.

Door simpelweg de instructies te volgen die te vinden is op https://docs.microsoft.com kom je er niet helemaal. Wat ik recent bent tegengekomen is dat er fouten staan omdat de gebruikte tooling online vernieuwd zijn, maar nog niet verwerkt zijn in het SharePoint Framework. Dit geeft als resultaat dat de uitgevoerde commando’s succesvol gelukt zijn met fouten….

<img src="/images/getting-started-with-spfx/why-successfull-with-errors.png" />


De oplossing: Wees niet eigenwijs door altijd de laatste versie te downloaden omdat het beter is (I did this...), maar download specifieke oude versies want onderhoud van gerelateerde componenten is complex. Je doel is om een webpart te maken wat uiteindelijk javascript is. Ik vertrouw erop dat Microsoft dit op zijn tijd zal vernieuwen.

Om heel wat frustraties (die ik had) je te besparen, heb ik de volgende stappen opgesteld zodat je als startende SPFx-developer snel van start kan gaan met het SharePoint Framework:

* Zet eerst een gratis Developer Tenant op volgens deze instructies:
https://docs.microsoft.com/en-us/sharepoint/dev/spfx/set-up-your-developer-tenant.
* Maak je development omgeving op volgens deze instructies, maar sla **Trusting the self-signed developer certificate** over:
https://docs.microsoft.com/en-us/sharepoint/dev/spfx/set-up-your-development-environment.
* Noteer de benodigde urls die je vaak nodig zult hebben
**Developer-account**: *[my-name]@[my-tenant].onmicrosoft.com*
**SharePoint Admin-Tenant**: *https://[my-tenant]-admin.sharepoint.com*
**App Catalog**: *https://[my-tenant].sharepoint.com/sites/appcatalog/SitePages/Introductiepagina.aspx*
**(Modern) Developer Site**: *https://[my-tenant].sharepoint.com/sites/DeveloperModern*
* Installeer NodeJS 10.15.3 (geen nieuwere versie):
https://nodejs.org/dist/v10.15.3/node-v10.15.3-x6msi
* Installeer Python 2.7 (geen nieuwere versie):
https://www.python.org/download/releases/2.7/
* In de command-prompt, maak nieuwe folder aan waar je de sources wilt hebben.
Voor de niet-DOS generatie developers hier de commando’s om op je C-schijf in de root een map aan te maken:
**C:**
**md myfirstwebpart**
**cd myfirstwebpart**
* SharePoint project aanmaken met volgende commando:
**yo @microsoft/sharepoint**
Beantwoord vervolgens de vragen die gesteld worden.
* Voor de zekerheid alle packages installeren van dit project met volgende commando:
**Npm install**
* Development certificaat vertrouwen met volgende commando:
**gulp trust-dev-cert**
<img src="/images/getting-started-with-spfx/dev-spfx-cert.png" />
Opmerking: Op de site van docs.microsoft.com stond deze stap boven het maken van een nieuw SPFx-project met yo@microsoft/sharepoint. Niet handig als je nog niet weet dat je een nieuw project gaat maken en zelf mag kiezen.
<img src="/images/getting-started-with-spfx/trust-self-signed-dev-cert.png" />
* Bundel alle assets voor development doeleinde (gehost op https://localhost:xxxx) met volgende commando:
**gulp bundle**
* Maak de SharePoint-package voor development doeleinde (gehost op https://localhost:xxxx) met volgende commando:
**gulp package-solution**
De uitvoer van dit commando komt terecht in de map ./sharepoint/solution
De SharePoint-package is het bestand eindigend met **.sppkg**.
* Start je lokale webserver en de Workbench met het volgende commando:
**Gulp serve**

Nu kun je aan de slag met Workbench welke automatisch gestart wordt. Je kunt het webpart toevoegen op de pagina en beginnen met ontwikkelen in Visual Studio Code. Let op: Je hebt nu geen beschikking tot data of SharePoint API's. Het is de bedoeling dat je eerst de UI ontwikkelt en fictieve data maakt en gebruikt. Zorg dat **gulp serve** in de command-prompt blijft draaien. Dit is je lokale webserver van je webpart! Elke keer als je iets aanpast, compileert **gulp serve** je code en ververst automatisch in je browser de Workbench.

**Klaar met mockdata en wil je ontwikkelen in SharePoint Online zodat je over de echte data beschikt van Graph en SharePoint?**
* Voer de volgende commando's uit:
**gulp bundle**
**gulp package-solution**
* Upload en ‘implementeer’ (installeer) de SharePoint-package (.sppkg) naar je eerder ingerichte App-Catalog site:
bijv.: https://[my-tenant].sharepoint.com/sites/appcatalog/AppCatalog/Forms/AllItems.aspx

Nu kun je de app installeren op elke SharePoint site als site beheerder en vervolgens de webpart op een willekeurige plek in je site plaatsen. Je hebt dan ook volledig beschikking tot de API’s van Graph en SharePoint.
*Let op dat je webpart 'werkt' zolang **gulp serve** op de achtergrond draait en als iemand op een ander apparaat dezelfde pagina benaderdt, deze een technische foutmelding krijgt*

**Klaar met ontwikkelen en wil je je ontwikkelde webpart deployen naar Test, Acceptatie en Productie?**
* Voer de volgende commando's uit:
**gulp bundle --ship**
**gulp package-solution --ship**
* Upload en ‘implementeer’ (installeer) de SharePoint-package (.sppkg) naar de App-Catalog site van gewenste tenant:
bijv.: https://[test/acc/prod-tenant].sharepoint.com/sites/appcatalog/AppCatalog/Forms/AllItems.aspx

Nu kun je de app installeren op elke SharePoint site als site beheerder en vervolgens de webpart op een willekeurige plek in je site plaatsen. Iedereen kan de webpart gebruiken en ook op telefoon en tablet via de SharePoint App.


