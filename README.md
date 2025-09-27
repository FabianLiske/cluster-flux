# Branching

## dev

Working Branch zum Basteln

## prod

Von Flux getrackter Branch, was hier steht landet im Cluster

# SOPS

Secrets werden nicht direkt im Repo getrackt, die kommen mit Age verschlüsselt ins Repo und dann über SOPS auf den Cluster. Dazu auf PC oder Rancher VM mit
```bash
sops --encrypt --in-place apps/<ordner>/<mein-secret>.yaml
```
verschlüsseln. Am besten nicht im Repo entschlüsseln, lieber das verschlüsselte Secret auf den Cluster bringen und im Racher UI den Inhalt checken.

# Neuinstallation (neue VM oder neuer Cluster)

## 0) Voraussetzungen
- Auf der Ops-VM (z. B. Rancher-VM): kubectl, flux, git, age, sops.
- Kubeconfig zeigt auf den Cluster (kubectl get nodes funktioniert).
- Repo existiert (z. B. github.com/<OWNER>/cluster-flux, Branch prod, Pfad clusters/picluster).

## 1) Variablen setzen

```bash
export OWNER="<OWNER>"
export REPO="cluster-flux"
export BRANCH="prod"
export PATH_IN_REPO="clusters/picluster"
export GITHUB_TOKEN="<PAT_mit_Contents+Administration_rw_auf_dieses_Repo>"
```

## 2) Flux im Cluster bootstrappen

```bash
flux bootstrap github \
  --owner=$OWNER \
  --repository=$REPO \
  --branch=$BRANCH \
  --path=$PATH_IN_REPO \
  --token-auth
```
  
Das legt/aktualisiert clusters/picluster/flux-system/* und startet die Controller in flux-system.

## 3) SOPS-Entschlüsselung wieder aktivieren

- Private age-Key (der zu deinem Repo-Public-Key aus .sops.yaml gehört) bereitstellen:
- Falls lokal vorhanden: Datei z. B. age.cluster.key.
- Falls nur im Backup/Passwortmanager: dort herholen.
- Secret anlegen (einmalig):

```bash
kubectl -n flux-system create secret generic sops-age \
  --from-file=age.agekey=./age.cluster.key
```
  
## 4) Status prüfen

```bash
kubectl -n flux-system get pods
kubectl -n flux-system get gitrepositories,kustomizations
```

## 5) End-to-End-Check

```bash
kubectl -n flux-system logs deploy/kustomize-controller | tail -n 50
```
