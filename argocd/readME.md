# Cluster bootstrap

Alt her kjøres manuelt én gang per cluster. ArgoCD kan ikke deploye sin
egen første Application, og plattformkomponentene installeres med Helm.

Etter bootstrap lever ressursene i clusteret, og alle videre endringer
til appen skjer via git.

## Forutsetninger

kind, kubectl og helm installert.

Guiden forutsetter et rent cluster. Vil du starte på nytt:

    kind delete cluster --name thunderbolt-cluster

## 1. Cluster og ArgoCD

    kind create cluster --name thunderbolt-cluster
    kubectl get nodes

    kubectl create namespace argocd
    kubectl apply -n argocd --server-side --force-conflicts \
      -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    kubectl wait --for=condition=available --timeout=300s \
      deployment/argocd-server -n argocd

ArgoCD-UI (valgfritt, men nyttig for feilsøking). La stå i eget vindu:

    kubectl port-forward svc/argocd-server -n argocd 8080:443

Brukernavn `admin`, passord:

    kubectl -n argocd get secret argocd-initial-admin-secret \
      -o jsonpath='{.data.password}' | base64 -d

Åpne https://localhost:8080 og godta sertifikatadvarselen.

## 2. Policy Controller

Admission webhook som verifiserer attester. Må være oppe før neste
steg, siden trust-policies trenger CRD-ene den installerer
(ClusterImagePolicy, TrustRoot).

    helm upgrade policy-controller --install --atomic \
      --create-namespace --namespace cosign-system \
      oci://ghcr.io/sigstore/helm-charts/policy-controller \
      --version 0.10.5

    kubectl get pods -n cosign-system

Vent til den viser 1/1 Running.

## 3. Trust policies

Selve regelen: images fra vår organisasjon må ha gyldig
SLSA-attestasjon. Verdiene ligger i attestations/.

    helm upgrade trust-policies --install --atomic \
      --namespace cosign-system \
      oci://ghcr.io/github/artifact-attestations-helm-charts/trust-policies \
      --version v0.7.0 \
      -f attestations/trust-policies-values.yml

    kubectl get clusterimagepolicy

## 4. Slå på håndheving i namespacet

Uten denne labelen verifiseres ingenting. Labelen hører til namespacet,
ikke til chartene.

    kubectl label namespace default policy.sigstore.dev/include=true

## 5. Appen

    kubectl apply -f argocd/application.yml
    kubectl get application thunderbolt -n argocd

## Verifiser at det virker

Negativ test — image utenfor policyen skal avvises:

    kubectl run test --image=nginx:latest

Forventet:

    Error from server (BadRequest): admission webhook "policy.sigstore.dev"
    denied the request: no matching policies

Ingen pod opprettes

Positiv test — vårt attesterte image skal kjøre:

    kubectl get pods

Se appen i nettleseren:

    kubectl port-forward svc/thunderbolt 8081:80

## Feilsøking

Se hva Policy Controller faktisk gjør:

    kubectl logs -n cosign-system -l app.kubernetes.io/name=policy-controller --tail=50

Se hvorfor en pod ikke starter:

    kubectl describe pod -l app=thunderbolt | grep -A15 Events

Tving ArgoCD til å lese repoet på nytt:

    kubectl patch application thunderbolt -n argocd \
      --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'

Tøm ArgoCDs repo-cache:

    kubectl rollout restart deployment argocd-repo-server -n argocd

## Merk

Policy Controller kjører i cosign-system, som ikke har labelen — den
verifiserer altså ikke sitt eget image. Derfor er chart-versjonene
pinnet, og kan verifiseres manuelt:

    gh attestation verify --owner github \
      oci://ghcr.io/github/artifact-attestations-helm-charts/trust-policies:v0.7.0