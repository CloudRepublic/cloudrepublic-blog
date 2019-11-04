---
title: Docker Swarm, Traefik, Let's Encrypt en Portainer
date: 2019-11-03 12:00:00
tags:
- Docker swarm
- Let's Encrypt
- Traefik
- Portainer
---

Case omschrijving 
---
Niet lang geleden heb ik besloten om mij eens goed te gaan verdiepen in Docker. Je hoort het overal en het wordt steeds populairder en en ik hoor steeds vaker dat het een goed alternatief is voor de huidige manier van werken namelijk Faas en Paas in een public cloud hierdoor ben ik nieuwsgierig geworden en wil ik eens kijken wat hier nu de voordelen van zijn. Ik ben begonnen om mij te certificeren, ik heb een cursus gevolgd en mijn examen met succes behaald. Ik vind het altijd belangrijk om te weten hoe iets in elkaar zit en waarom het zo in elkaar zit. Mijn idee is om een of meerdere productie applicaties op een Docker Swarm cluster te gaan draaien. In deze blog ga ik laten zien hoe ik een goede infrastructuur bouw om de applicaties te hosten.

De omgeving moest aan de volgende voorwaarden voldoen:

- Er moeten meerdere web applicaties op kunnen draaien welke op poort 80 en 443 benaderd kunnen worden.
- Alle web applicaties moeten draaien op SSL, dit moet mij zo min mogelijk werk kosten.
- De oplossing moet schaalbaar zijn en bestand zijn tegen het uitvallen en/of offline gaan van servers.
- Ik wil de oplossing kunnen hosten bij een Cloud provider, Hosting provider of lokaal om te kunnen testen.
- Het beheren van de containers moet gemakkelijk en overzichtelijk zijn.
- Aangezien ik een echte Hollander ben moeten de kosten ook een beetje beperkt blijven.

De oplossing welke ik gemaakt heb is een combinatie van Docker Swarm, Traefik, Let's Encrypt en Portainer.

Wat is Docker Swarm
---
Docker Swarm is een tool waarmee je Docker-containers kunt beheren en schalen. Als je Docker installeert dan krijg je daar meteen Swarm bij. Docker Swarm is een standaard product van Docker wat bij elke installatie van Docker wordt meegeleverd. Met Docker Swarm kun je een cluster bouwen van verschillende virtuele machines deze worden hierna nodes genoemd. Hierop kunnen de Docker-containers worden gedeployed als een stack\*\* of een service\*. Als je een cluster wilt hebben wat een hoge beschikbaarheid heeft wordt aangeraden om minstens 3 manager nodes te hebben. Docker Swarm maakt namelijk gebruik van het Raft Consensus Algoritme, 1 manager is de leider van de swarm en de status van de manager wordt gesyncroniseerd over de overige managers. Mocht de leider niet meer beschikbaar zijn om wat voor reden dan ook kan een andere manager zijn taken over nemen.

Om te berekenen hoeveel managers er mogen uitvallen voordat het cluster niet meer kan functioneren wordt de volgende berekening gebruikt: (N-1/2).
Bij dus een cluster van 3 manager mag er 1 manager uitvallen en zal het cluster nog steeds functioneren. Docker adviseert om niet meer dan 7 managers te gebruiken om performance issues met het syncrosniseren te voorkomen. Voor meer informatie zie https://docs.docker.com/engine/swarm/raft/

#TODO omschrijven hoe je een cluster opbouwd en er nodes aan toevoegd

Ik ga hier niet uitleggen hoe je een Docker Swarm cluster moet bouwen, hier zijn hele goede handleidingen voor bijv. in de Docker documentatie https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm/ 

<img src="/images/Docker-Swarm-overview.png" />
Een overzicht hoe het cluster eruit ziet met 3 managers en 2 workers.


Omdat ik Docker Swarm gebruik kan ik de volgende punten afvinken van mijn lijstje:
- De oplossing moet schaalbaar zijn en bestand zijn tegen het uitvallen of offline gaan van servers. 
    -- In het cluster zitten 3 managers dus er kan er 1 offline gaan. Ook hebben we 2 workers welke de containers kunnen hosten
- Ik wil de oplossing kunnen hosten bij een Cloud provider, Hosting provider of lokaal om te kunnen testen.
    -- Omdat Docker Swarm standaard in elke Docker installatie zit kan het zonder verdere installatie van tools gebruikt worden overal waar je Docker hebt geïnstalleerd.  

Wat is Let's Encrypt
---
Let's Encrypt is een certificaatautoriteit opgericht op 16 april 2016. Het geeft X.509 certificaten uit voor het Transport Layer Security (TLS) encryptie-protocol, zonder dat dit kosten met zich meebrengt. De certificaten worden uitgegeven via een geautomatiseerd proces dat is ontworpen om het tot nu toe complexe proces van handmatige validatie, ondertekening, installatie en hernieuwing van certificaten voor beveiligde websites te elimineren. (Wikipedia)

Voor meer informatie over Let's Encrypt zie https://letsencrypt.org/

Wat is Traefik
---
Traefik is een opensource router welke speciaal is ontworpen voor container oplossingen. Traefik wordt als global service op elke manager gedeployed op het cluster. Dit wil zeggen elke node met als rol manager krijgt een Traefik container. De reden dat Traefik op de manager nodes gedeployed dient te worden is dat de Docker api wordt uitgelezen. Zodra er een container bij komt en deze is geconfigureerd met de Traefik labels kan Traefik de labels van de container uitlezen en een virtuele host aanmaken voor de container en een SSL-certificaat aanvragen bij Let's Encrypt. Zodoende is de container beschikbaar voor de buitenwereld met een SSL-certificaat.

Voor meer informatie over Traefik zie https://traefik.io/

Omdat ik Traefik gebruik als reverse proxy kan ik de volgende punten afvinken van mijn lijstje:
- Er moeten meerdere web applicaties op kunnen draaien welke op poort 80 en 443 benaderd kunnen worden.
    -- Doordat Traefik de labels van containers kan uitlezen maakt het automatisch virtuele hosts aan.
- Alle webapplicaties moeten draaien op ssl, dit moet mij zo min mogelijk werk kosten.
    -- Traefik ondersteund out of the box Let's Encrypt SSL-certificaten.

Wat is Portainer
---
Portainer is eeen opensource web interface om je Docker te beheren en om nieuwe containers uit te rollen en te updaten en bevat nog veel meer functionaliteiten. Ik ga hier niet in diepte behandelen wat Portainer allemaal kan voor meer informatie over portainer kun je terecht op https://www.portainer.io/. 
Portainer dient ook op een node welke een manager geïnstalleerd te worden omdat Portainer ook via de Docker api het cluster beheerd. Tevens is er Portainer agent beschikbaar welke als global service gedeployed dient te worden op alle nodes zodat Portainer ook weet heeft welke containers op welke nodes draaien.

Omdat ik Portainer gebruik als beheer tool kan ik het laatste punt afvinken van mijn lijstje:
- Het beheren van de containers moet gemakkelijk en overzichtelijk zijn. 

De combinatie van Docker Swarm, Traefik, Let's Encrypt en Portainer
---
We hebben het hierboven gehad over Docker Swarm, Traefik, Let's Encrypt en Portainer maar hoe ziet dat landschap er nu uit. 
In de volgende afbeelding heb ik een overzicht van het landschap zoals hierboven omschreven.

<img src="/images/Docker-Swarm-landschap.png" />

Het verkeer komt via het internet uit op de Azure Traffic Manager Het kan ook via een DNS Round-robin alleen de Azure Traffic Manager controleert of de docker node beschikbaar is. Zodra de node niet beschikbaar is wegens onderhoud of een storing zal er geen verkeer naar de node worden gestuurd.

Het verkeer komt binnen op de Traefik loadbalancer welke de Reverse proxy en de SSL-certificaten verzorgd. Traefik weet welk request er naar welke container gestuurd moet worden door middel van de labels welke zijn ingesteld bij het deployen van de stack\*\* of service\*. 

Zie hier een voorbeeld Docker-compose file om een Traefik container te deployen als stack\*\* op Docker Swarm.
``` yaml
version: '3.7'
services:
  traefik:
    image: traefik:1.7.13
    ports:
      - target: 80
        published: 80
        mode: host
      - target: 443
        published: 443
        mode: host
    command: >
      --api
      --acme
      --acme.storage=/certs/acme.json
      --acme.entryPoint=https
      --acme.httpChallenge.entryPoint=http
      --acme.onHostRule=true
      --acme.onDemand=false
      --acme.acmelogging=true
      --acme.email=${EMAIL}
      --docker
      --docker.swarmMode
      --docker.domain=${DOMAIN}
      --docker.watch
      --defaultentrypoints=http,https
      --entrypoints='Name:http Address::80'
      --entrypoints='Name:https Address::443 TLS'
      --logLevel=DEBUG
      --accessLog
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - traefik_certs:/certs
    configs:
      - source: traefik_htpasswd
        target: /etc/htpasswd
    networks:
      - public
    deploy:
      mode: global
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
      labels:
        - "traefik.docker.network=public"
        - "traefik.port=8080"
        - "traefik.backend=traefik"
        - "traefik.enable=true"
        - "traefik.frontend.rule=Host:traefik.${DOMAIN}"
        - "traefik.frontend.auth.basic.usersFile=/etc/htpasswd"
        - "traefik.frontend.headers.SSLRedirect=true"
        - "traefik.frontend.entryPoints=http,https"

configs:
  traefik_htpasswd:
    file: ./htpasswd

networks:
  public:
    driver: overlay
    name: public

volumes:
  traefik_certs: {}
```

Een kleine samenvatting wat er gebeurd in dit Docker-compose bestand:

- We maken een container aan op basis van traefik:1.7.13.
- We publiseren poort 80 en 443.
- We regelen de Docker swarm en acme configuratie in.
- We mounten 2 volumes.
- We koppelen het wachtwoord bestand vanuit de config.
- We zorgen dat het wordt uitgerold op elke manager in het cluster.
- We regelen de webui van Traefik in.
- We maken een netwerk aan genaamd Public van het type overlay. 

Om de stack* uit te rollen voeren we het volgende commando uit om een manager node: 
```
docker stack deploy -c docker-compose.traefik.yml proxy
```


Zie hier een voorbeeld Docker-compose file om een Portainer container en een Portainer agent te deployen als stack** op Docker Swarm.

``` yaml
version: '3.7'

services:
  agent:
    image: portainer/agent
    environment:
      AGENT_CLUSTER_ADDR: tasks.agent
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    networks:
      - private
    deploy:
      mode: global
  portainer:
    image: portainer/portainer
    command: -H tcp://tasks.agent:9001 --tlsskipverify
    volumes:
      - portainer-data:/data
    networks:
      - private
      - public
    deploy:
      placement:
        constraints:
          - node.role == manager
      labels:
        - traefik.frontend.rule=Host:portainer.${DOMAIN}
        - traefik.enable=true
        - traefik.port=9000
        - traefik.tags=public
        - traefik.docker.network=public
        - traefik.redirectorservice.frontend.entryPoints=http
        - traefik.redirectorservice.frontend.redirect.entryPoint=https
        - traefik.webservice.frontend.entryPoints=https

networks:
  private:
    driver: overlay
    name: private
  public:
    external: true

volumes:
  portainer-data: {}

```
Een kleine samenvatting wat er gebeurd in dit Docker-compose bestand:

- We maken een container aan op basis van portainer/portainer en portainer/agent.
- We publiceren poort 9000 voor de UI.
- We regelen de Traefik configuratie in.
- We mounten een volume voor de portainer data.
- We zorgen dat de Portainer container wordt uitgerold op een manager in het cluster en dat de agent op elke node in het cluster wordt uitgerold. 
- We maken een netwerk aan met de naam Private en koppelen dit aan de agent en de Portainer UI.
- We koppelen de Portainer UI ook aan het netwerk Public wat we hebben aangemaakt in de Traefik deploy.

Om de stack** uit te rollen voeren we het volgende commando uit om een manager node: 
```
docker stack deploy -c docker-compose.portainer.yml portainer
```

Vanaf nu kunnen we Portainer benaderen op de URL https://portainer.yourdomain.com en kunnen we hier vandaan de rest van de services\* en stacks\*\* deployen en beheren.


Conclusie
---
Ik vind dat de combinatie tussen de verschillende tools een hele goede basis zijn als applicatie landschap. Het is flexibel, schaalbaar en niet bijzonder complex. Docker Swarm staat ook bekend om zijn eenvoudigheid. Aangezien ik geen infrastructuur met honderden nodes en duizenden containers hoef te hosten is dit een hele geschikte oplossing. Docker Swarm heeft niet een hele steile leercurve dus je kunt er al snel mee aan de slag. Door de reverse proxy van Traefik kunnen virtuele hosts automatisch worden aangemaakt en is SSL meteen geregeld. Met Portainer als UI voor het beheer kun je in een mum van tijd een heel applicatie landschap optuigen zonder al te veel tijd en kosten te moeten investeren.

\* Een service is een image van een microservice in de context van een grotere toepassing.
\** Een stack is een Docker-compose file met services gedefinieerd welke in een keer uitgerold kan worden.