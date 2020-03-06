---
title: Kubernetes, Nginx ingress controller en Let's Encrypt op AKS
date: 2020-03-04 12:00:00
tags:
- Kubernetes
- Let's Encrypt
- Nginx ingress controller
- AKS
---

Case omschrijving
---

Hier komt de omschrijving.

Waarom heb ik Kubernetes nodig
---

Ik hoor vaak het argument waarom zou ik kubernetes gebruiken wij hebben alles via [Paas](https://en.wikipedia.org/wiki/Platform_as_a_service) en [Faas](https://en.wikipedia.org/wiki/Function_as_a_service). Nu is Kubernetes ook geen vervanging voor Paas of Faas maar het is een toevoeging aan je toolbox. Er is een quote "Als je alleen een hamer hebt, neig je ernaar elk probleem te zien als een spijker." Je kan een hele hoop oplossingen kwijt in Paas en Faas maar niet voor alles is Paas of Faas de juiste oplossing bijv.

- Als je complexe architecturen hebt kan dit een uitdaging zijn.
- Als je volledige controle wilt hebben over je infrastructuur.
- Als je legacy applicaties wilt hosten.
- Als je geen vendor lock wilt hebben.
- Kubernetes is ook mogelijk in je eigen datacentrum.
- De applicaties zijn schaalbaar tot ... instanties.
- Je hebt een standaard deploy methode voor elke applicatie.

Wat is Kubernetes
---

Kubernetes is heel plat gezegd een beheer tool voor je containers. Kubernetes kan je onder andere helpen met de volgende zaken:

## Wat kan kubernetes doen voor je

### Service discovery en loadbalancing

Kubernetes kan de load van applicaties verdelen over de verschillende instanties van de applicatie zodat de load verdeelt wordt over de verschillende instanties. Als je applicatie gaat schalen zullen de nieuwe instanties worden toegevoegd aan de interne loadbalanacer.

### Storage orchestration

In Kubernetes heb je de mogelijkheid om een gedeelde storage in te stellen op je cluster. Stel je voor dat je een MySql server draait dan wil je niet dat als je MySql container stuk gaat dat je je database weg is. Deze kan je dan op een storage zetten. Hierdoor gaat je data nooit verloren als je container verwijder wordt.
Er zijn veel providers welke ondersteuning bieden aan Kubernetes:
- awsElasticBlockStore
- azureDisk
- azureFile
- gcePersistentDisk
- iscsi
- local
- nfs

Voor een complete lijst kijk op [Types of Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes)

### Geautomatiseerd applicaties updaten en terug rollen

In kubernetes kun je geautomatiseerd je applicaties updaten zonder downtime. Je kan opgeven hoeveel pods er offline mogen zijn tijdens een update van de applicatie. Je kunt ook opgeven dat er extra pods moeten worden aangemaakt tijdens de update zodoende blijft je applicatie op de gewenste hoeveelheid instanties. 
Mocht de nieuwe versie toch niet goed zijn is deze gemakkelijk terug te rollen. Kubernetes houd een history bij van de gedeployde versies.

### Automatische verdeling van resources

Kubernetes verdeeldt automatisch de load over de nodes gebaseerd op de recourse requirements en de beschikbaarheid op de nodes. Nodes kunnen worden voorzien van labels zodat alleen bepaalde workloads daar mogen draaien.

### Automatisch herstellend

Mocht er een workload niet meer goed functioneren dan kan Kubernetes zelf een nieuwe versie van de pod opstarten. Of de pod nog goed functioneerd kan op meerdere manieren gecontroleerd worden.

- In een Dockerfile is een ENTRYPOINT gedefineerd zodra dit process niet meer beschikbaar is zal de pod gestopt worden.
- In een pod definitie kan je een livenessProbe instellen met bijv. een url welke gecontroleerd word als de resonse van de URL anders is dan een status code 200 zal de pod als unhealthy worden gezien.

### Secret en configuration beheer

In kubernetes kun je secrets en configuraties aanmaken welke dan gebruikt kunnen worden in de applicaties. Deze objecten zijn op alle nodes beschikbaar en worden beheerd door kubernetes. Secrets en configuraties kunnen worden uitgelezen als environment variabelen of als een volume worden gemount in de pod.


Hoe begin je met Kubernetes
---

Nou het eenvoudigste is eigenlijk neem een managed instantie bij een cloud provider. Bij een managed instantie hoef je je niet meer druk te maken over de installatie en configuratie van het cluster. Het kost namelijk een hele hoop tijd om een goed werkend en een veilig cluster te bouwen. Je moet het cluster blijven monitoren of het nog goed werkt en zelf alerts instellen om op de hoogte gehouden te worden als het cluster niet goed functioneert. Omdat kubernetes zo uitgebreidt is kan er ook ontzettend veel misgaan en dan moet je het zelf troubleshooten en oplossen.

Er zijn heel veel smaakjes van kubernetes te krijgen bijv.

- Azure Kubernetes Service (AKS)
- Amazon Elastic Kubernetes Service (EKS)
- Google Kubernetes Engine (GKE)

Ik ga het hier verder over AKS hebben dit is de managed Kubernetes oplossing van Azure.


Wat is AKS
---

Wat is een Nginx ingress controller
---

Wat is Let's Encrypt
---

Let's Encrypt is een certificaatautoriteit opgericht op 16 april 2016. Het geeft X.509 certificaten uit voor het Transport Layer Security (TLS) encryptie-protocol, zonder dat dit kosten met zich meebrengt. De certificaten worden uitgegeven via een geautomatiseerd proces dat is ontworpen om het tot nu toe complexe proces van handmatige validatie, ondertekening, installatie en hernieuwing van certificaten voor beveiligde websites te elimineren. ([Wikipedia](https://nl.wikipedia.org/wiki/Let%27s_Encrypt))

Voor meer informatie over Let's Encrypt zie https://letsencrypt.org/

Conclusie
---
