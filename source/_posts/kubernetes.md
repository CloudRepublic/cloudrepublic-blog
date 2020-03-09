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

Ik hoor vaak het argument waarom zou ik kubernetes gebruiken wij hebben alles via [Paas](https://en.wikipedia.org/wiki/Platform_as_a_service) en [Faas]
(https://en.wikipedia.org/wiki/Function_as_a_service). Nu is Kubernetes ook geen vervanging voor Paas of Faas maar het is een toevoeging aan je toolbox. 

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

Er zijn heel veel varianten van kubernetes te krijgen enkele voorbeelden zijn bijv.

- Azure Kubernetes Service (AKS)
- Amazon Elastic Kubernetes Service (EKS)
- Google Kubernetes Engine (GKE)

Ik ga het hier verder over AKS hebben dit is de managed Kubernetes oplossing van Azure.


Wat is AKS
---
AkS staat voor Azure Kubernetes Service en is een managed service van Azure om je Kubernetes workload op te draaien. AKS is volledig in Azure geïntegreerd het maakt bijv. gebruik van azure monitoring en alerting. Je kan bijv. zien hoe je cluster erbij staat en hoe je containers draaien. Je kunt op container niveau inloggen en de logs bekijken. Ook kun je doormiddel van een terminal direct inloggen op de container. Azure DevOps heeft een hele goede integratie met AKS.

<img src="/images/Kubernetes_azure_monitor.png" />

Hoe installeer je AKS
---
AKS kun je volledig installeren doormiddel van de azure cli, voer de onderstaande commando's uit om aks te installeren.

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
- Een service is een object wat een pod exposed naar buiten. De reden dat je hier een appart object voor gebruikt wordt is dat als je een pod connect op het ipadres en de pod wordt verplaatst naar een andere node dan krijg je een nieuw ipadress.
- Een deployment is de desired state configuratie van de pod.
- In een replicaset staat gedefineerd hoeveel pods er moeten draaien.
- Een pod is een wrapper om een of meerdere containers.

Om dus een applicatie te deployen zijn er meerdere componenten nodig. Als voorbeeld nemen we een wordpress applicatie welke er als volgt uit ziet.

<img src="/images/kubernetes_wordpress_overview.png" />

hiervoor is de volgende yaml nodig:

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


Conclusie
---
