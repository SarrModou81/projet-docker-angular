# Partie pratique — Jenkins vs GitLab CI

Ce projet expose désormais trois pipelines équivalents pour le même workflow
(build → test → analyse qualité SonarQube → build & push de l'image Docker) :

| Outil          | Fichier                          |
|----------------|-----------------------------------|
| GitHub Actions | `.github/workflows/pipeline-docker.yml` (existant) |
| Jenkins        | `Jenkinsfile`                     |
| GitLab CI      | `.gitlab-ci.yml`                  |

## 1. Démo Jenkins (local, via Docker)

```bash
docker volume create jenkins_home
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

> Utiliser `jenkins/jenkins:lts` (activement maintenue) plutôt que `jenkinsci/blueocean`,
> qui n'est plus mise à jour et dont le cœur Jenkins trop ancien fait échouer l'installation
> des plugins actuels.

L'image ne contient pas le client Docker par défaut ; le copier depuis l'image officielle
`docker:cli` :

```bash
docker create --name docker-cli-tmp docker:cli
docker cp docker-cli-tmp:/usr/local/bin/docker docker-binary
docker cp docker-binary jenkins:/usr/bin/docker
docker exec -u root jenkins chmod +x /usr/bin/docker
docker exec -u root jenkins chmod 666 /var/run/docker.sock
docker rm docker-cli-tmp
```

1. Récupérer le mot de passe admin initial : `docker logs jenkins` puis ouvrir `http://localhost:8080`.
2. Installer les plugins nécessaires : **NodeJS**, **Docker Pipeline**, **SonarQube Scanner**, **Slack Notification**.
3. Configurer un outil NodeJS nommé `NodeJS-22` (Manage Jenkins > Tools).
4. Ajouter les credentials `dockerhub-credentials` (Username/Password) et `sonar-token` (Secret text).
5. Créer un job **Pipeline** pointant vers ce dépôt (`Jenkinsfile` à la racine) et lancer un build.

## 2. Démo GitLab CI

Deux options pour la démo :

- **Miroir GitLab.com** : pousser ce dépôt vers un projet GitLab, définir les variables CI/CD
  (`DOCKER_HUB_USERNAME`, `DOCKER_HUB_PASSWORD`, `SONAR_TOKEN`, `SONAR_HOST_URL`, `SLACK_WEBHOOK_URL`)
  dans *Settings > CI/CD > Variables*, puis pousser sur `main` pour déclencher `.gitlab-ci.yml`.
- **GitLab local avec `gitlab-runner exec`** (sans serveur GitLab complet) pour tester un job isolément :
  ```bash
  gitlab-runner exec docker test
  ```

## 3. Ce qu'il faut montrer pendant la démo

- Le déclenchement du pipeline (push / webhook) côté Jenkins vs côté GitLab.
- La lecture du `Jenkinsfile` (Groovy, stages impératifs) vs `.gitlab-ci.yml` (YAML déclaratif).
- L'interface d'exécution : la **Stage View** de Jenkins vs l'onglet **Pipelines** intégré à GitLab.
- Le résultat final identique : image Docker publiée sur Docker Hub, notification Slack.

## 4. Points d'attention

- Les tests Angular (`ng test`) nécessitent Chrome/Chromium disponible sur l'agent d'exécution.
- Le job Docker de `.gitlab-ci.yml` utilise Docker-in-Docker (`services: [docker:27-dind]`) — à activer
  côté runner GitLab (`privileged = true` dans `config.toml`).
- Le Jenkinsfile suppose un agent Jenkins avec Docker CLI installé et le socket Docker monté.
