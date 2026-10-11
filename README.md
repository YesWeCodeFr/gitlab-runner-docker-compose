
# Installer un runner GitLab avec Docker Compose

Ce guide accompagne la vidéo consacrée à l'installation d'un runner GitLab dans Docker Compose.

L'objectif est d'ajouter un runner Docker à une instance GitLab déjà auto-hébergée. Il exécutera les jobs CI/CD dans des conteneurs Docker temporaires, puis nous déclencherons un premier pipeline pour vérifier son fonctionnement.

Cette procédure part de l'architecture mise en place dans les vidéos précédentes : GitLab est installé dans `/srv/gitlab`, accessible en HTTPS derrière NGINX, et connecté au réseau Docker externe `gitlab-network`.

> Les commandes sont prévues pour un serveur Ubuntu ou Debian. Exécutez-les avec un utilisateur qui possède les droits `sudo` et qui peut utiliser Docker.

> Dans ce guide, remplacez `git.example.com` par le nom d'hôte de votre instance GitLab.

## Ressources liées

- YouTube — Installer un runner GitLab avec Docker Compose : lien à ajouter
- GitHub — [GitLab auto-hébergé avec Docker Compose](https://github.com/YesWeCodeFr/gitlab-docker-compose-self-hosted)
- GitHub — [NGINX avec Docker Compose et Let's Encrypt](https://github.com/YesWeCodeFr/nginx-docker-compose-multisite)
- LinkedIn — [me suivre sur LinkedIn](https://www.linkedin.com/in/dominique-korzeczek-695033188)

## Prérequis

- Une instance GitLab fonctionnelle, accessible en HTTPS depuis le serveur qui hébergera le runner.
- Docker Engine et le plugin Docker Compose installés.
- L'installation GitLab existante dans `/srv/gitlab`.
- Le réseau Docker externe `gitlab-network`, déjà utilisé par GitLab.
- Un compte administrateur GitLab pour créer le runner.
- Des projets et des utilisateurs de confiance : un runner exécute le code défini dans les pipelines.

Vérifiez Docker et Docker Compose :

```bash
docker --version
docker compose version
```

> La vidéo précédente configure GitLab avec un certificat Let's Encrypt. Si votre instance utilise un certificat autosigné, le conteneur du runner devra également faire confiance à l'autorité de certification avant son enregistrement.

## 1. Créer le dossier de configuration du runner

Le fichier de configuration du runner `config.toml` créé lors de l'enregistrement du runner doit rester présent lorsque le conteneur est recréé. Créez donc un dossier persistant :

```bash
sudo mkdir -p /srv/gitlab/config-gitlab-runner
```

## 2. Ajouter le runner à Docker Compose

Ouvrez le fichier Docker Compose de GitLab :

```bash
cd /srv/gitlab/compose
sudo nano docker-compose.yml
```

Ajoutez le service `gitlab-runner` dans la section `services`. Le réseau `gitlab-network` est déjà déclaré par l'installation GitLab ; conservez sa déclaration à la fin du fichier.

Voici le fichier complet correspondant à l'installation des vidéos précédentes :

```yaml
services:
  gitlab:
    image: gitlab/gitlab-ce:19.4.1-ce.0
    container_name: gitlab
    hostname: git.example.com
    restart: unless-stopped
    stop_grace_period: 5m
    shm_size: "256m"

    volumes:
      - /srv/gitlab/config:/etc/gitlab
      - /srv/gitlab/data:/var/opt/gitlab
      - /srv/gitlab/logs:/var/log/gitlab

    ports:
      - "2222:22"

    networks:
      - gitlab-network

    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url "https://git.example.com"
        nginx['listen_port'] = 80
        nginx['listen_https'] = false
        letsencrypt['enable'] = false
        gitlab_rails['gitlab_shell_ssh_port'] = 2222

    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://localhost/-/readiness?all=1 >/dev/null || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 30
      start_period: 5m

  gitlab-runner:
    image: gitlab/gitlab-runner:v19.4.1
    container_name: my-gitlab-runner
    restart: unless-stopped
    stop_grace_period: 1m

    depends_on:
      gitlab:
        condition: service_healthy

    networks:
      - gitlab-network

    volumes:
      - /srv/gitlab/config-gitlab-runner:/etc/gitlab-runner
      - /var/run/docker.sock:/var/run/docker.sock

networks:
  gitlab-network:
    external: true
```

Adaptez les valeurs suivantes :

- `hostname` et `external_url` doivent correspondre au nom d'hôte public de GitLab ;
- les volumes GitLab existants doivent rester inchangés ;
- le volume `/srv/gitlab/config-gitlab-runner` conserve la configuration du runner ;
- le socket `/var/run/docker.sock` permet au runner, avec l'exécuteur Docker, de créer les conteneurs temporaires des jobs.

## 3. Démarrer le conteneur du runner

Vérifiez la syntaxe de la configuration, puis démarrez uniquement le runner :

```bash
docker compose config
docker compose up -d gitlab-runner
```

Contrôlez son état et ses journaux :

```bash
docker compose ps
docker compose logs --tail=50 gitlab-runner
```

Le conteneur démarre avant d'être enregistré, mais il ne peut pas encore récupérer de jobs.

## 4. Créer un runner dans GitLab

Dans l'administration GitLab, ouvrez `CI/CD` → `Runners`, puis créez un **runner d'instance**.

Lors de sa création :

1. Sélectionnez le système d'exploitation Linux.
2. Ajoutez le tag `docker`.
3. Ne cochez pas l'option autorisant l'exécution des jobs sans tag.
4. Créez le runner.

GitLab affiche alors un token d'authentification qui commence par `glrt-`. Copiez-le : il sera demandé une seule fois par la commande d'enregistrement.

> Ne publiez jamais ce token dans un dépôt Git, une capture d'écran ou un fichier de configuration versionné.

## 5. Enregistrer le conteneur du runner

Depuis le dossier `/srv/gitlab/compose`, lancez l'enregistrement interactif :

```bash
docker compose exec gitlab-runner gitlab-runner register
```

Répondez aux questions affichées :

- URL GitLab : `https://git.example.com`
- Token : le token `glrt-` copié à l'étape précédente
- Exécuteur : `docker`
- Image Docker par défaut : `alpine:latest`

L'enregistrement crée le fichier `/srv/gitlab/config-gitlab-runner/config.toml` sur l'hôte.

## 6. Vérifier la connexion du runner

Retournez dans `CI/CD` → `Runners` dans l'administration GitLab, puis actualisez la page.

Le runner doit apparaître avec le statut **en ligne** et le tag `docker`.

Vous pouvez aussi vérifier la connexion depuis le serveur :

```bash
docker compose exec gitlab-runner gitlab-runner verify
```

Le runner est maintenant prêt à exécuter les jobs portant le tag `docker`.

## 7. Créer un projet de test et son pipeline

Dans GitLab, créez un projet de test nommé par exemple `test-runner-gitlab`. Initialisez-le avec un fichier `README.md` afin de créer le premier commit.

À la racine du projet, créez ensuite le fichier `.gitlab-ci.yml` suivant :

```yaml
hello:
  tags:
    - docker
  image: alpine:latest
  script:
    - echo "Hello World depuis le runner GitLab"
```

Créez le fichier. Le commit déclenche automatiquement un pipeline.

Le tag `docker` est important : GitLab affecte le job au runner qui possède ce tag. Comme l'exécution des jobs sans tag a été désactivée, un job sans ce tag resterait en attente.

## 8. Vérifier le pipeline et le relancer

Dans le projet, ouvrez `Build` → `Pipelines`.

Le pipeline associé au commit doit se terminer avec le statut **réussi**. Ouvrez le job `hello` pour consulter sa trace : elle contient le message suivant :

```text
Hello World depuis le runner GitLab
```

Pour vérifier qu'un commit déclenche systématiquement un pipeline, modifiez le fichier `README.md`, créez un nouveau commit, puis retournez dans `Build` → `Pipelines`. Un second pipeline apparaît et exécute le même job.

## Récapitulatif rapide des commandes

```bash
# Créer le dossier de configuration persistant
sudo mkdir -p /srv/gitlab/config-gitlab-runner

# Vérifier et démarrer le runner
cd /srv/gitlab/compose
docker compose config
docker compose up -d gitlab-runner
docker compose ps
docker compose logs --tail=50 gitlab-runner

# Enregistrer et vérifier le runner
docker compose exec gitlab-runner gitlab-runner register
docker compose exec gitlab-runner gitlab-runner verify
```

## Points d'attention

- Le runner d'instance peut exécuter les jobs des projets auxquels il est affecté. Utilisez-le uniquement avec des projets et des comptes de confiance, ou préférez un runner de projet pour isoler les usages.
- Le socket Docker est nécessaire au runner pour créer les conteneurs de jobs. Ne le transmettez pas automatiquement aux conteneurs de jobs : cela donnerait aux scripts du pipeline un contrôle étendu sur le Docker de l'hôte.
- `alpine:latest` est adapté à cette démonstration. Pour des pipelines de production, utilisez une image et une version explicitement définies afin de rendre les builds reproductibles.
- Surveillez l'espace disque du serveur : les images, conteneurs, volumes et caches produits par les jobs Docker peuvent s'accumuler.

## Aller plus loin

- Remplacez le job `hello` par des tests automatisés.
- Construisez et publiez une image Docker depuis un pipeline.
- Créez des runners dédiés selon les projets ou les environnements.
- Ajoutez des limites de ressources et une stratégie de nettoyage adaptée à votre serveur.
