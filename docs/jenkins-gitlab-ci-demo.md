# Demo pratique : Jenkins vs GitLab CI

Ce guide accompagne la presentation "Jenkins vs GitLab CI" et s'appuie sur
**ce meme projet** (Angular + Docker + Keycloak + Postgres), deja pilote par
un pipeline GitHub Actions (`.github/workflows/pipeline-docker.yml`).

Les fichiers ajoutes reproduisent exactement les memes etapes
(install -> test -> build -> SonarQube -> Docker build/push -> notification
Slack) dans les deux outils, pour permettre une comparaison directe le jour
de l'examen :

| Fichier | Outil |
|---|---|
| `.github/workflows/pipeline-docker.yml` | GitHub Actions (deja existant) |
| `Jenkinsfile` | Jenkins |
| `.gitlab-ci.yml` | GitLab CI |
| `Dockerfile.jenkins` + `docker-compose.jenkins.yml` | Jenkins local pour la demo |

## 1. Demo Jenkins (en local, avec Docker)

Jenkins tourne dans un conteneur separe, avec acces au socket Docker de la
machine hote pour pouvoir builder/pusher l'image de l'appli.

```bash
# Depuis la racine du projet
docker compose -f docker-compose.jenkins.yml up -d --build
```

1. Ouvrir `http://localhost:8081`
2. Recuperer le mot de passe initial :
   ```bash
   docker exec jenkins_demo cat /var/jenkins_home/secrets/initialAdminPassword
   ```
3. Installer les plugins suggeres, puis ajouter les plugins :
   `Docker Pipeline`, `SonarQube Scanner`, `NodeJS`.
4. Configurer les credentials Jenkins (menu *Manage Jenkins > Credentials*) :
   - `dockerhub-credentials` (username/password Docker Hub)
   - `sonar-token` (secret text)
   - `slack-webhook-url` (secret text)
5. Creer un job **Pipeline** pointant sur ce depot Git, branche
   `main`, script path `Jenkinsfile`.
6. Lancer un build (*Build Now*) et montrer en direct les stages
   (Install, Tests, Build, SonarQube, Docker build, Docker push) dans la
   vue "Stage View".

Points a souligner en live :
- Jenkins doit etre installe/maintenu soi-meme (conteneur, plugins,
  credentials configures a la main).
- Le pipeline est un script Groovy (`Jenkinsfile`) versionne avec le code.

## 2. Demo GitLab CI

GitLab CI necessite un depot GitLab (le pipeline ne se declenche pas sur
GitHub). Le plus simple pour la demo : creer un depot miroir gratuit sur
`gitlab.com`.

1. Creer un projet vide sur gitlab.com, ex. `projet-docker-angular`.
2. Ajouter le remote et pousser le code (avec `.gitlab-ci.yml` deja present) :
   ```bash
   git remote add gitlab https://gitlab.com/<votre-compte>/projet-docker-angular.git
   git push gitlab main
   ```
3. Dans *Settings > CI/CD > Variables*, ajouter :
   - `SONAR_TOKEN`, `SONAR_HOST_URL`
   - `DOCKER_HUB_USERNAME`, `DOCKER_HUB_PASSWORD`
   - `SLACK_WEBHOOK_URL`
4. Aller dans *Build > Pipelines* : le pipeline se declenche automatiquement
   au push et affiche les stages (`install`, `test`, `build`, `quality`,
   `docker`, `notify`) directement dans l'interface GitLab, sans rien
   installer.

Points a souligner en live :
- Aucune installation : les runners partages GitLab.com sont utilises
  directement.
- Le fichier `.gitlab-ci.yml` est un simple YAML, tres proche de la syntaxe
  GitHub Actions deja utilisee dans ce projet.

## 3. Comparaison a montrer a l'ecran

Ouvrir cote a cote :
- L'onglet GitHub Actions du repo (pipeline existant)
- L'interface Jenkins locale (`localhost:8081`)
- L'interface GitLab CI (gitlab.com)

... pour la meme action (push sur `main`), et montrer que les 3 outils
executent la meme suite de taches, mais avec une experience et un cout
d'infrastructure tres differents (a relier au tableau comparatif de la
presentation).

## Nettoyage apres la demo

```bash
docker compose -f docker-compose.jenkins.yml down -v
```
