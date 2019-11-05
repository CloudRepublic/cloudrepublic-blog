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
Niet lang geleden heb ik besloten om mij eens goed te gaan verdiepen in Docker. Je hoort het overal en het wordt steeds populairder en ik hoor steeds vaker dat het een goed alternatief is voor de huidige manier van werken namelijk Faas en Paas in een public cloud hierdoor ben ik nieuwsgierig geworden en wil ik eens kijken wat hier nu de voordelen van zijn. Ik ben begonnen om mij te certificeren, ik heb een cursus gevolgd en mijn examen met succes behaald. Ik vind het altijd belangrijk om te weten hoe iets in elkaar zit en waarom het zo in elkaar zit. Mijn idee is om een of meerdere productie applicaties op een Docker Swarm cluster te gaan draaien. In deze blog ga ik laten zien hoe ik een goede infrastructuur bouw om de applicaties te hosten.

De omgeving moest aan de volgende voorwaarden voldoen:

- Er moeten meerdere web applicaties op kunnen draaien welke op poort 80 en 443 benaderd kunnen worden.
- Alle web applicaties moeten draaien op SSL, dit moet mij zo min mogelijk werk kosten.
- De oplossing moet schaalbaar zijn en bestand zijn tegen het uitvallen en/of offline gaan van servers.
- Ik wil de oplossing kunnen hosten bij een Cloud provider, Hosting provider of lokaal om te kunnen testen.
- Het beheren van de containers moet gemakkelijk en overzichtelijk zijn.
- Aangezien ik een echte Hollander ben moeten de kosten ook een beetje beperkt blijven.

De oplossing welke ik heb gemaakt is een combinatie van Docker Swarm, Traefik, Let's Encrypt en Portainer.

Wat is Docker Swarm
---
Docker Swarm is een tool waarmee je Docker-containers kunt beheren en schalen. Als je Docker installeert dan krijg je daar meteen Swarm bij. Docker Swarm is een standaard product van Docker wat bij elke installatie van Docker wordt meegeleverd. Met Docker Swarm kun je een cluster bouwen van verschillende virtuele machines deze worden hierna nodes genoemd. Hierop kunnen de Docker-containers worden gedeployed als een stack\*\* of een service\*. Als je een cluster wilt hebben wat een hoge beschikbaarheid heeft wordt aangeraden om minstens 3 manager nodes te hebben. Docker Swarm maakt namelijk gebruik van het Raft Consensus Algoritme, 1 manager is de leider van de swarm en de status van de manager wordt gesynchroniseerd over de overige managers. Mocht de leider niet meer beschikbaar zijn om wat voor reden dan ook kan een andere manager zijn taken over nemen.

Om te berekenen hoeveel managers er mogen uitvallen voordat het cluster niet meer kan functioneren wordt de volgende berekening gebruikt: (N-1/2).
Bij dus een cluster van 3 manager mag er 1 manager uitvallen en zal het cluster nog steeds functioneren. Docker adviseert om niet meer dan 7 managers te gebruiken om performance issues met het synchroniseren te voorkomen. Voor meer informatie zie https://docs.docker.com/engine/swarm/raft/

Om een Docker Swarm cluster op te zetten kun je de volgende stappen uitvoeren:
- Installeer 5 virtuele machine met bijv. Ubuntu server. Voor de 3 managers hebben we niet hele zware virtuele machines nodig. Een basic A1 1.75 GB RAM volstaat al voor een manager node. 
Voor de 2 worker nodes zou ik kiezen voor een virtuele machine met 8 GB RAM. 
- Installeer Docker community edition op alle 5 de servers. Volg de handleiding op https://docs.docker.com/install/linux/docker-ce/ubuntu/. 
- Op 1 van de manager servers voer het volgende commando uit om Docker Swarm te initialiseren.
```
$ docker swarm init --advertise-addr "public ipadres van de server"
Swarm initialized: current node (bvz81updecsj6wjz393c09vti) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-3pu6hszjas19xyp7ghgosyx9k8atbfcr8p2is99znpy26u2lkl-1awxwuwd3z9j1z3puu7rcgdbx \
    172.17.0.2:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
- Voer het bovenstaande docker swarm join token uit op de 2 nodes. 
- Voer het comamndo docker swarm join-token manager uit en voer het join commando uit op de overige 2 managers.

Als dit klaar gedaan is heb je een Docker Swarm cluster gemaakt zoals in het onderstaande overzicht is weergegeven.

<img src="/images/Docker-Swarm-overview.png" />
Mocht je meer details willen hebben over het opzetten van een Docker Swarm cluster kijk dan op: https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm/

Docker Swarm zal niet automatisch schalen als de load op je applicatie hoger wordt, wil je meer containers uitrollen van je applicatie kun je dit doen door meer replica's aan te maken. Als je bijv. de service api_api wilt opschalen naar 5 instanties kun je dit doen door middel van het volgende commando:

```
$ docker service scale api_api=5 
```
Je kunt de schaling ook regelen in de UI van Portainer.
<img src="/images/Docker-Swarm-scaling.png" />

Mocht je nu toch te weinig capaciteit hebben kun je eenvoudig een nieuwe virtuele machine inrichten met ubuntu en docker erop installeren. Hierna voer je het Docker Swarm join commando uit op de server en deze zal het bestaande cluster uitbreiden met de extra capaciteit. 
```
docker swarm join \
    --token SWMTKN-1-3pu6hszjas19xyp7ghgosyx9k8atbfcr8p2is99znpy26u2lkl-1awxwuwd3z9j1z3puu7rcgdbx \
    172.17.0.2:2377
```
Mocht je Docker Swarm op Azure hebben uitgerold kun je gebruik maken van een vm scale sets om automatisch nodes bij je cluster te zetten. Ik heb hier verder nog geen ervaring mee maar ga hier zeker mee aan de slag.

Omdat ik Docker Swarm gebruik kan ik de volgende punten afvinken van mijn lijstje:
- De oplossing moet schaalbaar zijn en bestand zijn tegen het uitvallen of offline gaan van servers. 
    -- In het cluster zitten 3 managers dus er kan er 1 offline gaan volgens de berekening: "3 managers - 1 = 2  2/2 = 1. 
    -- Ook hebben we 2 workers welke de containers kunnen hosten. Mocht er 1 offline gaan worden de containers opnieuw gestart op de andere node.
    -- De manager heeft onder andere als taak om er voor te zorgen dat de containers draaien op 1 of meerdere nodes.
- Ik wil de oplossing kunnen hosten bij een Cloud provider, Hosting provider of lokaal om te kunnen testen.
    -- Omdat Docker Swarm standaard in elke Docker installatie zit kan het zonder verdere installatie van tools gebruikt worden overal waar je Docker hebt geïnstalleerd.  

Wat is Let's Encrypt
---
Let's Encrypt is een certificaatautoriteit opgericht op 16 april 2016. Het geeft X.509 certificaten uit voor het Transport Layer Security (TLS) encryptie-protocol, zonder dat dit kosten met zich meebrengt. De certificaten worden uitgegeven via een geautomatiseerd proces dat is ontworpen om het tot nu toe complexe proces van handmatige validatie, ondertekening, installatie en hernieuwing van certificaten voor beveiligde websites te elimineren. (Wikipedia)

Voor meer informatie over Let's Encrypt zie https://letsencrypt.org/

Wat is Traefik
---
Traefik is een opensource router welke speciaal is ontworpen voor container oplossingen. Traefik wordt als global service op elke manager gedeployed op het cluster. Dit wil zeggen elke node met als rol manager krijgt een Traefik container. De reden dat Traefik op de manager nodes gedeployed dient te worden is dat de Docker api wordt uitgelezen. Zodra er een container bij komt en deze is geconfigureerd met de Traefik labels kan Traefik de labels van de container uitlezen en een virtuele host aanmaken voor de container en een SSL-certificaat aanvragen bij Let's Encrypt. Zodoende is de container beschikbaar voor de buitenwereld met een SSL-certificaat.

Zie hier een voorbeeld Docker-compose file om een Traefik container te deployen als stack\*\* op het Docker Swarm cluster.
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

Om de stack* uit te rollen voeren we het volgende commando uit om een manager node: 
```
docker stack deploy -c docker-compose.traefik.yml proxy
```

Een kleine samenvatting wat er gebeurt in dit Docker-compose bestand:

- We maken een container aan op basis van traefik:1.7.13.
- We publiseren poort 80 en 443.
- We regelen de Docker swarm en acme configuratie in.
- We mounten 2 volumes.
- We koppelen het wachtwoord bestand vanuit de config.
- We zorgen dat het wordt uitgerold op elke manager in het cluster.
- We regelen de webui van Traefik in.
- We maken een netwerk aan genaamd Public van het type overlay. 


Voor meer informatie over Traefik zie https://traefik.io/

Omdat ik Traefik gebruik als reverse proxy kan ik de volgende punten afvinken van mijn lijstje:
- Er moeten meerdere web applicaties op kunnen draaien welke op poort 80 en 443 benaderd kunnen worden.
    -- Doordat Traefik de labels van containers kan uitlezen maakt het automatisch virtuele hosts aan.
- Alle webapplicaties moeten draaien op ssl, dit moet mij zo min mogelijk werk kosten.
    -- Traefik ondersteund out of the box Let's Encrypt SSL-certificaten.

Wat is Portainer
---
Portainer is een opensource web interface om je Docker te beheren zowel lokaal als remote. Met Portainer kun je de volgende Docker concepten beheren:  
- Containers
- Images
- Networks
- Volumes
- Services
- Swarm Cluster

<img src="/images/Docker-Swarm-portainer-dashboard.png" />

Portainer dient ook op een manager node geïnstalleerd te worden omdat Portainer ook via de Docker api het cluster beheerd. Tevens is er Portainer agent beschikbaar welke als global service gedeployed dient te worden op alle nodes zodat Portainer ook weet heeft welke containers op welke nodes draaien.

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
Om de stack** uit te rollen voeren we het volgende commando uit om een manager node: 
```
docker stack deploy -c docker-compose.portainer.yml portainer
```

Vanaf nu kunnen we Portainer benaderen op de URL https://portainer.yourdomain.com en kunnen we hier vandaan de rest van de services\* en stacks\*\* deployen en beheren.


Een kleine samenvatting wat er gebeurt in dit Docker-compose bestand:

- We maken een container aan op basis van portainer/portainer en portainer/agent.
- We publiceren poort 9000 voor de UI.
- We regelen de Traefik configuratie in.
- We mounten een volume voor de Portainer data.
- We zorgen dat de Portainer container wordt uitgerold op een manager in het cluster en dat de agent op elke node in het cluster wordt uitgerold. 
- We maken een netwerk aan met de naam Private en koppelen dit aan de agent en de Portainer UI.
- We koppelen de Portainer UI ook aan het netwerk Public wat we hebben aangemaakt in de Traefik deploy.

Voor meer informatie over Portainer kijk op: https://www.portainer.io/. 

Omdat ik Portainer gebruik als beheer tool kan ik het laatste punt afvinken van mijn lijstje:
- Het beheren van de containers moet gemakkelijk en overzichtelijk zijn. 

De combinatie van Docker Swarm, Traefik, Let's Encrypt en Portainer
---
We hebben het hierboven gehad over Docker Swarm, Traefik, Let's Encrypt en Portainer maar hoe ziet dat landschap er nu uit. 
In de volgende afbeelding heb ik een overzicht van het landschap zoals hierboven omschreven.

<img src="/images/Docker-Swarm-landschap.png" />

Ik heb gekozen om Azure Traffic Manager te gebruiken als loadbalancer voor om het verkeer te verdelen tussen de 3 verschillende managers. De gedachte hier achter is grotendeels dat het een erg goedkope service is in Azure en het werkt ook met externe endpoints. Je bent dus niet verplicht om je virtuele machines op Azure te moeten draaien dit geeft je weer de flexibiliteit om de machines overal te hosten.

Ik heb 3 endpoints gedefineerd in de Azure Traffic Manager 1 endpoint voor elke manager.
Ik heb Azure Traffic Manager ingeregeld dat hij voor de beste performance kiest. Je kunt een protocol, poort en eventueel een pad opgeven wat hij moet controleren. Mocht het endpoint niet meer beschikbaar zijn zal er geen verkeer meer naar toe gestuurd worden. Dus als er een storing of een update is van een manager zal er geen downtime zijn in de applicaties. Als er een manager niet bereikbaar is hebben we nog 2 andere managers over welke het werk overnemen.

<img src="/images/Docker-Swarm-traffic-manager-configuration.png" />


Hierna komt het verkeer binnen op de Traefik loadbalancer welke de Reverse proxy en de SSL-certificaten verzorgd. Traefik weet welk request er naar welke container gestuurd moet worden door middel van de labels welke zijn ingesteld bij het deployen van de stack\*\* of service\* en docker maakt intern gebruikt van zijn eigen DNS server zodat er bekend is welke container er op welke node draait. 

Zie hier een voorbeeld Docker-compose file met labels om een Docker container met een webapplicatie te deployen als stack** op een Docker Swarm cluster.
```yml
  version: '3.7'

services:
  web:
    image: marcoippel/web:0.1
    environment: 
      - ASPNETCORE_URLS=http://+:5000
    networks:
      - public
    deploy:
      placement:
        constraints:
          - node.role == worker
      labels:
        - traefik.frontend.rule=Host:test.${DOMAIN}
        - traefik.enable=true
        - traefik.port=80
        - traefik.tags=public
        - traefik.docker.network=public
        - traefik.redirectorservice.frontend.entryPoints=http
        - traefik.redirectorservice.frontend.redirect.entryPoint=https
        - traefik.webservice.frontend.entryPoints=https

networks:
  public:
    external: true
```

Het Docker-compose bestand kan in Portainer als stack gedeployed worden op het cluster.

Conclusie
---
Ik vind dat de combinatie tussen de verschillende tools een hele goede basis zijn als applicatie landschap. Het is flexibel, schaalbaar en niet bijzonder complex. Docker Swarm staat ook bekend om zijn eenvoudigheid. Aangezien ik geen infrastructuur met honderden nodes en duizenden containers hoef te hosten is dit een hele geschikte oplossing. Docker Swarm heeft niet een hele steile leercurve dus je kunt er al snel mee aan de slag. Door de reverse proxy van Traefik kunnen virtuele hosts automatisch worden aangemaakt en is SSL meteen geregeld. Met Portainer als UI voor het beheer kun je in een mum van tijd een heel applicatie landschap optuigen zonder al te veel tijd en kosten te moeten investeren.

Om het laatste puntje van mijn lijstje af te kunnen vinken heb ik hier een klein overzicht met een kosten indicatie van 3 verschillende providers waar je een docker Swarm cluster kan hosten. De totaal kosten welke hieronder staan zijn gebaseerd op 3 manager nodes en 2 worker nodes. De Manager nodes hebben allemaal 2 gig geheugen en de workers hebben 8 gig geheugen.

| Provider                                                                                                | Manager VM            | Worker VM         | Totaal        |
| :--------------                                                                                         |:--------------        | :--------------   |:------------- |
| [Azure](https://azure.microsoft.com/nl-nl/pricing/details/virtual-machines/ubuntu-advantage-standard/)  | B1MS      €32/mo      | B2MS      €76/mo  | €248/mo       |
| [Digital Ocean](https://www.digitalocean.com/pricing/#Compute)                                          | Standard  $10/mo      | Standard  $40/mo  | $110/mo       |  
| [Strato](https://www.strato.nl/server/vps-linux/)                                                       | Linux V10 €5/mo       | Linux V30 €15/mo  | €45/mo        |

De kosten voor de Azure Traffic Manager zijn: Eerste 1 miljard DNS-query's/maand	€0,456 per miljoen query's. https://azure.microsoft.com/nl-nl/pricing/details/traffic-manager/

Zoals je ziet in het overzicht 3 providers waarvan Digital Ocean en Azure echt serieuze cloud providers zijn met veel meer services dan alleen virtuele machines. Azure biedt een uptime van 99,9% en Digital Ocean een uptime van 99,99% dit is ook iets waar je voor betaald. Zo zie je dat voor iedereen zijn portemonnee een oplossing is. 

Mocht je toch niet tevreden zijn met de service van je hosting provider dan is het heel gemakkelijk om je infrastructuur op te pakken en deze gewoon bij een andere provider te hosten. 


\* Een service is een image van een microservice in de context van een grotere toepassing.
\** Een stack is een Docker-compose file met services gedefinieerd welke in een keer uitgerold kan worden.