---
title: Kubernetes, Nginx ingress controller en Let's Encrypt op AKS
date: 2020-03-04 12:00:00
tags:
- Kubernetes
- Waarom kubernetes
- AKS
---

Case omschrijving
---

In deze blog ga ik uitleggen wanneer je kubernetes kan gebruiken, hoe je ermee kunt starten op AKS en hoe je een applicatie kunt uitrollen. Als voorbeeld gaan we een wordpress applicatie uitrollen op kubernetes.

Waarom heb ik Kubernetes nodig
---

Ik hoor vaak het argument waarom zou ik kubernetes gebruiken wij hebben alles via [Paas](https://en.wikipedia.org/wiki/Platform_as_a_service) en [Faas](https://en.wikipedia.org/wiki/Function_as_a_service). Nu is Kubernetes ook geen vervanging voor Paas of Faas maar het is een toevoeging aan je toolbox. 

<img src="/images/kubernetes_hamer.jpeg" />

__<p style="text-align: center;">"Als je alleen een hamer hebt, neig je ernaar elk probleem te zien als een spijker."</p>__
Je kan een hele hoop oplossingen kwijt in Paas en Faas maar niet voor alles is Paas of Faas de juiste oplossing bijv.

- Als je complexe architecturen hebt kan dit een uitdaging zijn.
- Als je volledige controle wilt hebben over je infrastructuur.
- Als je legacy applicaties wilt hosten.
- Als je geen vendor lock wilt hebben.
- Kubernetes is ook mogelijk in je eigen datacentrum.
- De applicaties zijn schaalbaar tot ... instanties.
- Je hebt een standaard deploy methode voor elke applicatie.

Wat is Kubernetes
---

Kubernetes, ook wel k8s genoemd, is kort gezegd een open-source systeem beheren van grote groepen containers en containerized applicaties. Met de software zijn containers te groeperen en eenvoudig(er) te beheren. Kubernetes kan je onder andere helpen met de volgende zaken:

#### Service discovery en loadbalancing

Kubernetes kan de load van applicaties verdelen over de verschillende instanties van de applicatie zodat de load verdeelt wordt over de verschillende instanties. Als je applicatie gaat schalen zullen de nieuwe instanties worden toegevoegd aan de interne loadbalancer en het binnenkomend verkeer wordt verdeelt over de nieuwe instanties.

<img src="/images/kubernetes_service_discovery.png" />

#### Storage orchestration

In Kubernetes heb je de mogelijkheid om een gedeelde storage in te stellen op je cluster. Stel je voor dat je een MySql server draait dan wil je niet dat als je MySql container stuk gaat dat je database weg is. Je kan hiervoor een persistant volume aanmaken en dit kun je koppelen aan een folder in je container. Zodoende als je container gaat schalen of hij werkt niet meer staan je database bestanden op een veilige plaats  
Er zijn veel providers welke ondersteuning bieden aan Kubernetes:
- awsElasticBlockStore
- azureDisk
- azureFile
- gcePersistentDisk
- iscsi
- local
- nfs

Voor een complete lijst kijk op [Types of Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes)

<img src="/images/kubernetes_storage.png" />

#### Geautomatiseerd applicaties updaten en terug rollen

In kubernetes kun je geautomatiseerd je applicaties updaten zonder downtime. Je kan opgeven hoeveel pods er offline mogen zijn tijdens een update van de applicatie. Je kunt ook opgeven dat er extra pods moeten worden aangemaakt tijdens de update zodoende blijft je applicatie op de gewenste hoeveelheid instanties. 
Mocht de nieuwe versie toch niet goed zijn is deze gemakkelijk terug te rollen. Kubernetes houd een history bij van de gedeployde versies.

<img src="/images/kubernetes_rollout.png" />

#### Automatische verdeling van resources

Kubernetes verdeeld automatisch de load over de nodes gebaseerd op de recourse requirements en de beschikbaarheid op de nodes. Nodes kunnen worden voorzien van labels zodat alleen bepaalde workloads daar mogen draaien.

<img src="/images/kubernetes_placements.png" />

#### Automatisch herstellend

Mocht er een workload niet meer goed functioneren dan kan Kubernetes zelf een nieuwe versie van de pod opstarten. Of de pod nog goed functioneert kan op meerdere manieren gecontroleerd worden.

In een Dockerfile is een ENTRYPOINT gedefinieerd zodra dit process niet meer beschikbaar is zal de pod gestopt worden.
```
FROM mcr.microsoft.com/dotnet/core/runtime:3.1
COPY --from=build-env /app/out .

# Start
ENTRYPOINT ["dotnet", "AuditlogService.dll"]
```

In een pod definitie kun je een livenessProbe instellen met bijv. een url welke gecontroleerd word als de response van de URL anders is dan een status code 200 zal de pod als unhealthy worden gezien.
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: auditlogservice
  name: auditlogservice
spec:
  containers:
  - image: marcoippel/auditlogservice:1.0
    name: auditlogservice
    resources: {}
    livenessProbe:
      httpGet:
        path: /health
    readinessProbe:
      httpGet:
        path: /ready
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

<p><img src="/images/kubernetes_self_healing.png" /></p>

#### Secret en configuration beheer

In kubernetes kun je secrets en configuraties aanmaken welke dan gebruikt kunnen worden in de applicaties. Deze objecten zijn op alle nodes beschikbaar en worden beheerd door kubernetes. Secrets en configuraties kunnen worden uitgelezen als environment variabelen of als een volume worden gemount in de pod.

<img src="/images/kubernetes_secret.png" />

Hoe begin je met Kubernetes
---

Het eenvoudigste is eigenlijk neem een managed instantie bij een cloud provider. Bij een managed instantie hoef je je niet meer druk te maken over de installatie en configuratie van het cluster. Het kost namelijk een hele hoop tijd om een goed werkend en een veilig cluster te bouwen. Je moet het cluster blijven monitoren of het nog goed werkt en zelf alerts instellen om op de hoogte gehouden te worden als het cluster niet goed functioneert. Omdat kubernetes zo uitgebreidt is kan er ook ontzettend veel misgaan en dan moet je het zelf troubleshooten en oplossen.

Er zijn heel veel varianten van kubernetes te krijgen enkele voorbeelden zijn bijv.

- Azure Kubernetes Service (AKS)
- Amazon Elastic Kubernetes Service (EKS)
- Google Kubernetes Engine (GKE)

Ik ga het hier verder over AKS hebben dit is de managed Kubernetes oplossing van Azure.


Wat is AKS
---
AkS staat voor Azure Kubernetes Service en is een managed service van Azure om je Kubernetes workload op te draaien. AKS is volledig in Azure geïntegreerd het maakt bijv. gebruik van azure monitoring en alerting. Je kunt bijv. zien wat de status is van je cluster en hoe je containers draaien. Je kunt op container niveau inloggen en de logfiles per container bekijken. Ook kun je door middel van een terminal direct inloggen op de container. Azure DevOps heeft een hele goede integratie met AKS wat het mogelijk maakt om gemakkelijk applicaties te deployen op je kubernetes cluster. In een andere blog zal ik in detail ingaan op devops in combinatie met kubernetes.

<img src="/images/Kubernetes_azure_monitor.png" />
Een overzicht van de monitoring in AKS.

Hoe installeer je AKS
---
AKS kun je volledig installeren door middel van de azure cli, voer de onderstaande commando's uit om AKS te installeren.

```bash
# maak een resource groep aan.
az group create --name kubernetesdemo --location west-europe

# maak een kubernetes cluster aan met twee workers.
az aks create --resource-group kubernetesdemo --name demo --node-count 2 --enable-addons monitoring --generate-ssh-keys

# installeer kubectl als deze nog niet id geinstalleerd
az aks install-cli

# haal de credentials op van je cluster en voeg deze toe aan je kubectl config
az aks get-credentials --resource-group kubernetesdemo --name demo

```

Nu ben je klaar om applicaties te deployen op Kubernetes.

Hoe deploy je applicaties op Kubernetes
---

Kubernetes bestaat uit verschillende componenten bijv.

- Een ingresscontroller is een soort van reverse proxy welke het verkeer op basis van de inkomende url het verkeer naar een bepaalde service kan sturen.
- Een service is een object wat een pod exposed naar buiten. De reden dat je hier een apart object voor gebruikt wordt is dat als je een pod connect op het ipadres en de pod wordt verplaatst naar een andere node dan krijg je een nieuw ipadress.
- Een pod is een wrapper om een of meerdere containers.
- Een persistant volume claim is een reservering op een persistant volume.
- Een persistant volume is een stuk storage wat beschikbaar is gesteld door een administrator.
- Een secret is een object waar je gevoelige informatie opslaan en beheren, zoals wachtwoorden, OAuth-tokens en ssh-sleutels.



Om componenten uit te rollen op kubernetes moeten we deze beschrijven in yaml. Deze yaml zal vervolgens worden geserialiseerd naar kubernetes objecten.
Om een voorbeeld wordpress applicatie uit te rollen hebben we de volgende yaml nodig:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: null
  name: wordpress
spec: {}
status: {}
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: demo 
  namespace: wordpress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: demo-kubernetes.westeurope.cloudapp.azure.com
    http:
      paths:
      - backend:
          serviceName: wordpress
          servicePort: 80
        path: /
---
apiVersion: v1
data:
  password: d2Vsa29tMDE=
kind: Secret
metadata:
  creationTimestamp: null
  name: mysql-pass
  namespace: wordpress
---
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
  namespace: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  labels:
    app: wordpress
  namespace: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
  namespace: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
---
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
  namespace: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pv-claim
  labels:
    app: wordpress
  namespace: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
  namespace: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:5.3.2-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wp-pv-claim
```

We kunnen deze yaml uitvoeren door het volgende commando uit te voeren:

```bash
kubectl apply -f wordpress.yaml
```

Dit commando zal de yaml naar de api sturen van kubernetes en de objecten aanmaken of updaten. Als alles goed is gegaan hebben we de onderstaande applicatie uitgerold:

<img src="/images/kubernetes_wordpress_overview.png" />

Moet ik al die yaml zelf typen ?
---
De wordpress applicatie bestaat uit ongeveer 171 regels yaml code welke ook nog eens de juiste manier moet inspringen moet je dat echt allemaal zelf typen? Nou nee gelukkig niet, je kunt het grootste deel van de yaml laten genereren. Als voorbeeld nemen we een deployment object.

```bash
kubectl create deployment my-dep --image=busybox -o yaml --dry-run > deployment.yaml
```

We maken hier een deployment aan met als naam my-dep en als image gebruiken we busybox. We doen een dry-run zodat we niets aanmaken en doen een output naar yaml. Dit alles schrijven we weg in een deployment.yaml bestand. Dit kunnen we doen voor al de objecten welke we nodig hebben.

Het resultaat ziet er als volgt uit:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: my-dep
  name: my-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-dep
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: my-dep
    spec:
      containers:
      - image: busybox
        name: busybox
        resources: {}
status: {}

```

Welke resources zijn handig om te bekijken
---

Ik luister graag in de auto naar podcasts en een hele goede podcast is die van Bret Fisher. Hij is een docker captain en hij heeft elke week een live stream op youtube en hij maakt hier ook podcasts van alles gaat over docker, docker swarm en kubernetes. https://www.bretfisher.com/podcast/

Als je wilt beginnen met kubernetes op je computer dan is k3d een hele goede optie. Het is een gestripte versie van kubernetes met een hele handige installer erbij. https://github.com/rancher/k3d

Als je aan de slag wilt met kubernetes en niet meteen een cluster op je pc wilt installeren dan kun je terecht op Katakoda. Het is een omgeving waar je voor een uur een tijdelijk cluster kunt starten. Er zijn ook korte cursussen aanwezig welke je dan kunt uitvoeren op het cluster. https://www.katacoda.com/

Als je iets langer wilt spelen met kubernetes dan kun je terecht op play with kubernetes. Dit is ook een gevirtualiseerde omgeving welke 4 uur beschikbaar is. https://labs.play-with-k8s.com/

Mocht je een cursus willen doen kan ik je echt de cursussen aanraden van KodeKloud op udemy. Dit zijn echt hele duidelijke cursussen en als bonus heb je toegang tot een online leeromgeving waar je allerlei opdrachten moet uitvoeren. https://www.udemy.com/course/certified-kubernetes-application-developer/


Conclusie
---
Ondanks dat kubernetes een enorme steile leercurve heeft is het eenmaal als je het door hebt een geweldig platform om je applicaties op te hosten. Je kunt er alle complexe applicaties op hosten maar je kunt er ook gewoon je Azure functions op hosten. Je hebt met Kubernetes een volledige gereedschapskist om al je oplossingen in te hosten. Met een hosted oplossing op bijv. AKS wordt al heel veel complexiteit uit handen genomen.