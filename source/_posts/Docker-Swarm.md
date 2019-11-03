---
title: Docker swarm traefik, Let's Encrypt en Portainer
date: 2019-11-03 12:00:00
tags:
- Docker swarm
- Let's Encrypt
- Traefik
- Portainer
---

Case omschrijving 
---
Niet lang geleden heb ik besloten om me eens goed te gaan verdiepen in Docker. Je hoort het overal en het wordt steeds populairder en ik wilde eens gaan kijken wat hier nu zo vet aan is. Ik ben begonnen om me te certifieren, ik heb een cursus gevolgd en mijn examen met success behaald. Ik vind het altijd belangerijk om te weten hoe iets in elkaar zit en waarom het zo in elkaar zit. Mijn idee is om een of meerdere productie applicaties op een Docker Swarm cluster te gaan draaien. In deze blog ga ik laten zien hoe ik een goede infrastructuur bouw om de applicaties te hosten. 
Ik had me de volgende voorwaarden gegegeven:
- Er moeten meerdere web applicaties op kunnen draaien welke op poort 80 en 443 benaderd kunnen worden.
- Alle webapplicaties moeten draaien op ssl, dit moet me zo min mogelijk werk kosten.
- De oplossing moet schaalbaar zijn en bestand zijn tegen het uitvallen of offline gaan van servers.
- Ik wil de oplossing kunnen hosten bij een Cloud provider, Hosting provider of lokaal om te kunnen testen.
- Het beheren van de containers moet gemakkelijk en overzichtelijk zijn. 

De oplossing welke ik gemaakt heb is een combinatie van Docker Swarm, Traefik, Let's Encrypt en Portainer.

Wat is Docker Swarm
---
Docker Swarm is een tool waarmee je Docker-containers kunt beheren en schalen. Als je Docker installeerd dan krjg je daar meteen swarm bij. Docker Swarm is een standaard product van Docker wat bij elke installatie wordt meegeleverd. Met Docker Swarm kun je een cluster bouwen van verschillende virtuele machines. Hierop kunnen de Docker-containers worden gedeployed. ALs je een cluster wilt hebben wat een hoge beschikbaarheid heeft wordt aangeraden om minstens 3 manager nodes te hebben. Ik ga hier niet uitleggen hoe een Docker Swarm cluster op te bouwen hier zijn hele goede handleidingen voor bijv. in de docker documentatie https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm/ 

<img src="/images/Docker-Swarm-overview.png" />
Een overzicht hoe het cluster eruit ziet met 3 managers en 2 workers.

Omdat ik Docker Swarm gebruik kan ik de volgende punten afvinken van mijn lijstje
- De oplossing moet schaalbaar zijn en bestand zijn tegen het uitvallen of offline gaan van servers. (In het cluster zitten 3 managers dus er kan er 1 offline gaan. Ook hebben we 2 workers welke de containers kunnen hosten).
- Ik wil de oplossing kunnen hosten bij een Cloud provider, Hosting provider of lokaal om te kunnen testen.

Wat is Let's Encrypt
---

Wat is Traefik
---
Traefik is een opensource router welke speciaal is ontworpen voor container oplossingen. Traefik wordt als global service op elke manager gedeployed op het cluster. Dit wil zeggen elke manager instantie krijg een Traefik container. De reden dat Traefik op de manager nodes gedeployed dient te worden is dat de Docker api wordt uitgelezen. Zodra er een container bij komt en deze is geconfigureerd met de Traefik labels kan Traefik de labels van de container uitlezen en een virtuele host aanmaken voor de container en een SSL certificaat aanvragen bij Let's Encrypt. Zodoende is de container beschikbaar voor de buitenwereld met een SSL certificaat.








De combinatie van Docker swarm traefik, Let's Encrypt en Portainer
---


Conclusie
---
