pipeline {                                   // Début du pipeline déclaratif Jenkins
  agent any                                  // Peut s'exécuter sur n'importe quel agent Jenkins (qui doit avoir docker et git)

  environment {                              // Variables d'environnement disponibles dans tout le pipeline
    DOCKERHUB_USER = 'omar926'                 // à adapter
                                             // Votre identifiant Docker Hub : préfixe des noms d'images
    REGISTRY_CREDS = credentials('dockerhub-credentials')     // user + access token
                                             // Lit le credential Jenkins "dockerhub-credentials" et crée automatiquement
                                             // REGISTRY_CREDS_USR (identifiant) et REGISTRY_CREDS_PSW (mot de passe/token)
    GIT_REPO       = 'github.com/VOTRE_COMPTE/VOTRE_DEPOT.git' // à adapter
                                             // Adresse du dépôt (sans "https://") : sert au git push avec le token
    IMAGE_TAG      = "${env.BUILD_NUMBER}"   // Tag des images = numéro du build Jenkins (1, 2, 3...), unique à chaque build
  }

  options { timestamps(); disableConcurrentBuilds() }
                                             // timestamps() : horodate chaque ligne de log
                                             // disableConcurrentBuilds() : un seul build à la fois (évite deux push Git simultanés)

  stages {                                   // Liste des étapes, exécutées dans l'ordre
    stage('Checkout') {                      // Étape 1 : récupérer le code
      steps { checkout scm }                 // Clone le dépôt Git configuré dans le job Jenkins (scm = "source code management")
    }

    stage('Build & tag') {                   // Étape 2 : construire et étiqueter les images
      steps {
        sh '''                               // Exécute un script shell (apostrophes simples : c'est le shell qui lit les $VARIABLES)
          SHA=$(git rev-parse --short HEAD)  # Récupère les 7 premiers caractères du hash du commit (ex. a1b2c3d)
          for pair in database:mysql backend:backend frontend:frontend; do
                                             # Boucle sur des paires "dossier:nom de l'image"
                                             # (le dossier database/ donne l'image "mysql")
            dir=${pair%%:*}; name=${pair##*:}   # %%:* garde ce qui est avant ":" (dossier) ; ##*: garde ce qui est après (nom)
            docker build -t $DOCKERHUB_USER/bijouterie-$name:$IMAGE_TAG \
                                             # Construit l'image avec le 1er tag : numéro de build (celui qui sera déployé)
                         -t $DOCKERHUB_USER/bijouterie-$name:$SHA \
                                             # 2e tag : le SHA du commit (permet de retrouver le code exact)
                         -t $DOCKERHUB_USER/bijouterie-$name:latest ./$dir
                                             # 3e tag : latest ; "./$dir" est le dossier contenant le Dockerfile
          done                               # Fin de la boucle : les 3 images sont construites
        '''
      }
    }

    stage('Push Docker Hub') {               // Étape 3 : envoyer les images sur Docker Hub
      steps {
        sh '''
          echo "$REGISTRY_CREDS_PSW" | docker login -u "$REGISTRY_CREDS_USR" --password-stdin
                                             # Connexion à Docker Hub ; le mot de passe passe par stdin (n'apparaît pas dans la ligne de commande)
          SHA=$(git rev-parse --short HEAD)  # Recalcule le SHA (chaque bloc sh est un nouveau shell : les variables ne sont pas conservées)
          for name in mysql backend frontend; do   # Boucle sur les 3 images
            docker push $DOCKERHUB_USER/bijouterie-$name:$IMAGE_TAG   # Pousse le tag numéro de build
            docker push $DOCKERHUB_USER/bijouterie-$name:$SHA         # Pousse le tag SHA
            docker push $DOCKERHUB_USER/bijouterie-$name:latest       # Pousse le tag latest
          done
        '''
      }
    }

    // CD GitOps : Jenkins ne touche pas au cluster, il met à jour Git ; Argo CD synchronise.
    stage('Update manifests (GitOps)') {     // Étape 4 : déclencher le déploiement en modifiant Git
      when { branch 'main' }                 // Uniquement sur la branche main (pas de déploiement depuis une branche de test)
      steps {
        withCredentials([usernamePassword(credentialsId: 'github-credentials',
                                          usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                                             // Charge le credential Jenkins "github-credentials" dans GIT_USER et GIT_TOKEN
                                             // pendant ce bloc seulement (les valeurs sont masquées dans les logs)
          sh '''
            sed -i "s|image: .*bijouterie-mysql:.*|image: $DOCKERHUB_USER/bijouterie-mysql:$IMAGE_TAG|"       k8s/02-mysql-deployment.yaml
                                             # Remplace dans le fichier la ligne "image: ...bijouterie-mysql:..." par la même image avec le nouveau tag
                                             # (sed -i modifie le fichier sur place ; "|" sert de séparateur car l'image contient des "/")
            sed -i "s|image: .*bijouterie-backend:.*|image: $DOCKERHUB_USER/bijouterie-backend:$IMAGE_TAG|"   k8s/06-backend-deployment.yaml
                                             # Même opération pour le backend
            sed -i "s|image: .*bijouterie-frontend:.*|image: $DOCKERHUB_USER/bijouterie-frontend:$IMAGE_TAG|" k8s/09-frontend-deployment.yaml
                                             # Même opération pour le frontend
            git config user.email "jenkins@ci.local"   # Identité du commit : adresse e-mail (obligatoire pour committer)
            git config user.name  "jenkins"            # Identité du commit : nom d'auteur
            git add k8s/                     # Ajoute les manifests modifiés au commit
            git commit -m "ci: images $IMAGE_TAG [skip ci]" || echo "rien à committer"
                                             # Commit avec un message clair ; "[skip ci]" évite de relancer le pipeline en boucle ;
                                             # si rien n'a changé, git échoue : "|| echo" empêche de faire échouer le build
            git push https://$GIT_USER:$GIT_TOKEN@$GIT_REPO HEAD:main
                                             # Pousse le commit sur la branche main, en s'authentifiant avec le token dans l'URL
                                             # Argo CD verra ce commit et déploiera
          '''
        }
      }
    }
  }

  post { always { sh 'docker logout || true' } }
                                             // Après le pipeline, qu'il réussisse ou non (always) : se déconnecte de Docker Hub
                                             // pour ne pas laisser les identifiants sur l'agent ; "|| true" ignore une éventuelle erreur
}