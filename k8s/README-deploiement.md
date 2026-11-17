# Déploiement : Kubernetes + Jenkins + Argo CD

Flux : `git push` → Jenkins (build, tag, push Docker Hub, met à jour les tags dans `k8s/`) → Argo CD détecte le commit et synchronise le cluster.

## Étape 1 : manifests (manuel)
    kubectl apply -f k8s/
    # http://localhost:30080 (ou: kubectl -n bijouterie port-forward svc/frontend 8080:80)

## Étape 2 : Jenkins
Agent avec docker + git. Credentials : `dockerhub-credentials` (Username/Password, access token), `github-credentials` (Username/Token, droit push).
Le commit de Jenkins contient `[skip ci]` pour éviter les boucles (à configurer côté plugin/webhook si besoin).

## Étape 3 : Argo CD
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    kubectl apply -f argocd/application.yaml
Interface : `kubectl -n argocd port-forward svc/argocd-server 8443:443`.
Mot de passe admin : `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`.
