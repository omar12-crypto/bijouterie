pipeline {
  // Pod agent éphémère lancé par Jenkins dans le cluster : client docker + démon docker (dind)
  agent {
    kubernetes {
      yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: docker
      image: docker:27-cli
      command: [cat]
      tty: true
      env:
        - name: DOCKER_HOST
          value: tcp://localhost:2375
    - name: dind
      image: docker:27-dind
      securityContext:
        privileged: true
      env:
        - name: DOCKER_TLS_CERTDIR
          value: ""
'''
    }
  }

  environment {
    DOCKERHUB_USER = 'omar926'                  // à adapter
    REGISTRY_CREDS = credentials('cb1d760b-d56f-48a8-a69a-97c0443ff7a3')      // user + access token
    GIT_REPO       = 'github.com/omar12-crypto/bijouterie.git' // à adapter
    IMAGE_TAG      = "${env.BUILD_NUMBER}"
  }

  options { disableConcurrentBuilds() }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Préparer') {
      steps {
        container('docker') {
          sh '''
            apk add --no-cache git
            git config --global --add safe.directory '*'
            until docker info >/dev/null 2>&1; do echo "attente du démon docker..."; sleep 2; done
          '''
        }
      }
    }

    stage('Build & tag') {
      steps {
        container('docker') {
          sh '''
            SHA=$(git rev-parse --short HEAD)
            for pair in database:mysql backend:backend frontend:frontend; do
              dir=${pair%%:*}; name=${pair##*:}
              docker build -t $DOCKERHUB_USER/bijouterie-$name:$IMAGE_TAG \
                           -t $DOCKERHUB_USER/bijouterie-$name:$SHA \
                           -t $DOCKERHUB_USER/bijouterie-$name:latest ./$dir
            done
          '''
        }
      }
    }

    stage('Push Docker Hub') {
      steps {
        container('docker') {
          sh '''
            echo "$REGISTRY_CREDS_PSW" | docker login -u "$REGISTRY_CREDS_USR" --password-stdin
            SHA=$(git rev-parse --short HEAD)
            for name in mysql backend frontend; do
              docker push $DOCKERHUB_USER/bijouterie-$name:$IMAGE_TAG
              docker push $DOCKERHUB_USER/bijouterie-$name:$SHA
              docker push $DOCKERHUB_USER/bijouterie-$name:latest
            done
          '''
        }
      }
    }

    // CD GitOps : Jenkins ne touche pas au cluster, il met à jour Git ; Argo CD synchronise.
    stage('Update manifests (GitOps)') {
      when { branch 'main' }
      steps {
        container('docker') {
          withCredentials([usernamePassword(credentialsId: 'a8405b8a-21ed-4b4f-b12e-1a6c55e539e4',
                                            usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
            sh '''
              sed -i "s|image: .*bijouterie-mysql:.*|image: $DOCKERHUB_USER/bijouterie-mysql:$IMAGE_TAG|"       k8s/02-mysql-deployment.yaml
              sed -i "s|image: .*bijouterie-backend:.*|image: $DOCKERHUB_USER/bijouterie-backend:$IMAGE_TAG|"   k8s/06-backend-deployment.yaml
              sed -i "s|image: .*bijouterie-frontend:.*|image: $DOCKERHUB_USER/bijouterie-frontend:$IMAGE_TAG|" k8s/09-frontend-deployment.yaml
              git config user.email "jenkins@ci.local"
              git config user.name  "jenkins"
              git add k8s/
              git commit -m "ci: images $IMAGE_TAG [skip ci]" || echo "rien à committer"
              git push https://$GIT_USER:$GIT_TOKEN@$GIT_REPO HEAD:main
            '''
          }
        }
      }
    }
  }
}