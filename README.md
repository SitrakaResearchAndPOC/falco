# falco

Falco peut fonctionner dans un conteneur Docker tout en surveillant l'activité des conteneurs et du système hôte. Ce tutoriel vous guide pour exécuter Falco dans Docker et configurer son analyse sur un autre conteneur Docker.
# Falco IDS 
* Étape 1 : Préparer l’Environnement
Installer Docker sur Ubuntu (si ce n'est pas déjà fait) :
```
sudo apt update && sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
```
Vérifier l’installation de Docker :
```
docker --version
```
Installer Docker Compose (optionnel pour orchestrer facilement plusieurs conteneurs) :
```
sudo apt install -y docker-compose
```
* Étape 2 : Lancer Falco dans un Conteneur Docker
Télécharger et exécuter l’image Docker de Falco : Falco doit avoir accès au noyau de l’hôte pour analyser les événements.
```
docker run --rm -it \
    --name falco \
    --privileged \
    --pid=host \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v /dev:/host/dev \
    -v /proc:/host/proc:ro \
    -v /boot:/host/boot:ro \
    -v /lib/modules:/host/lib/modules:ro \
    -v /usr:/host/usr:ro \
    falcosecurity/falco:latest
```
Testé par : 
```
docker run --rm -it \
    --name falco \
    --privileged \
    --pid=host \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v /dev:/host/dev \
    -v /proc:/host/proc:ro \
    -v /boot:/host/boot:ro \
    -v /lib/modules:/host/lib/modules:ro \
    -v /usr:/host/usr:ro \
    falcosecurity/falco:0.39.2
```

Vérifier que Falco fonctionne : Une fois le conteneur en cours d'exécution, vous verrez des logs indiquant que Falco surveille les événements du système.

* Étape 3 : Configurer Falco pour Surveiller les Conteneurs Docker
Falco surveille par défaut l’activité Docker en analysant les événements du socket Docker. Voici comment tester cela. </br>
Créer un conteneur Docker à analyser : Lancez un conteneur NGINX comme exemple : </br>
```
docker pull nginx
```
```
docker run --rm -d --name nginx-example nginx
```
Simuler un comportement suspect dans le conteneur : Accédez à une session interactive dans le conteneur NGINX :
```
docker exec -it nginx-example /bin/bash
```
Ensuite, tentez de lire un fichier sensible, par exemple :
```
cat /etc/shadow
```
Observer les logs dans Falco : Dans le conteneur Falco, vous devriez voir une alerte comme :
```
21:30:15.123456 Detected suspicious activity: Attempt to read /etc/shadow inside container nginx-example
```
* Étape 4 : Ajouter une Règle Personnalisée pour les Conteneurs
Créer un fichier pour les règles personnalisées : Montez un fichier local pour ajouter des règles personnalisées au conteneur Falco :
```
nano custom_rules.yaml
```
Exemple de règle personnalisée pour détecter l’accès à un fichier spécifique dans un conteneur :

```
- rule: Detect Access to Sensitive File in Containers
  desc: "Détecte les accès à /etc/shadow dans n'importe quel conteneur"
  condition: >
    container and evt.type=open and fd.name = "/etc/shadow"
  output: >
    "Accès à /etc/shadow détecté dans le conteneur (conteneur=%container.name utilisateur=%user.name fichier=%fd.name)"
  priority: WARNING
  tags: [docker, sensitive]
```
Relancer Falco avec les règles personnalisées : Montez le fichier custom_rules.yaml lors du lancement de Falco :
```
docker run --rm -it \
    --name falco \
    --privileged \
    --pid=host \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v /dev:/host/dev \
    -v /proc:/host/proc:ro \
    -v /boot:/host/boot:ro \
    -v /lib/modules:/host/lib/modules:ro \
    -v /usr:/host/usr:ro \
    -v $(pwd)/custom_rules.yaml:/etc/falco/falco_rules.local.yaml \
    falcosecurity/falco:latest
```
Tester la règle personnalisée : Accédez au conteneur NGINX et tentez d’accéder à /etc/shadow. Falco devrait déclencher l’alerte définie.

* Étape 5 : Surveiller et Configurer les Logs
Accéder aux logs Falco : Vous pouvez consulter les logs directement dans le terminal ou monter un volume pour les exporter :
```
-v /var/log/falco:/var/log/falco
```
Configurer des intégrations : Modifiez le fichier /etc/falco/falco.yaml pour ajouter des intégrations (Slack, webhooks, etc.).
Avec ce guide, vous avez installé Falco dans un conteneur Docker et configuré sa surveillance pour détecter des comportements anormaux dans d'autres conteneurs. Vous pouvez étendre cette configuration en créant des règles personnalisées ou en intégrant Falco avec des outils de gestion d’alertes comme Prometheus ou Grafana.
