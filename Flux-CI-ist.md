# Flux-Setup Ist-Zustand

- Cluster: ein K3s-Cluster („picluster“), angebunden an Flux
- Management: Flux läuft als Controller im Namespace flux-system
- Bootstrap via flux bootstrap github gegen GitHub-Repo cluster-flux, Branch prod
- Zugriff auf das Repo erfolgt über ein Fine-grained GitHub-Token (Repo-Scope)
- Repo-Struktur:

```bash
cluster-flux/
├── clusters/
│   └── picluster/
│       ├── flux-system/           # vom Bootstrap angelegte Manifeste
│       ├── cluster-root.yaml      # Flux-Kustomization CR für Root
│       └── kustomization.yaml     # Kustomize-File, das flux-system + cluster-root einbindet
└── apps/
    ├── kustomization.yaml         # Aggregator, listet aktive Workload-Ordner
    └── …                          # (noch leer, außer evtl. Tests)
```

## Root-Kustomization (cluster-root):

- überwacht Repo-Pfad ./apps
- interval: 1m, prune: true, wait: true
- versieht alle Ressourcen mit Label gitops=flux
- Decryption aktiviert:

```bash
decryption:
  provider: sops
  secretRef:
    name: sops-age
```

## Secrets-Management:

- SOPS + age im Einsatz
- .sops.yaml im Repo-Root mit Regel: alle apps/**/…-secret.yaml → Felder data/stringData werden verschlüsselt
- sops-age Secret im Namespace flux-system enthält den privaten age-Key des Clusters
- Verschlüsselte Secrets werden im Repo versioniert, Flux entschlüsselt sie im Cluster zur Laufzeit

## Git-Workflow:

- Branch prod ist die „Source of Truth“ für den Cluster
- Änderungen an Konfiguration oder neuen Workloads → Commit/Merge in prod
- Flux reconciled automatisch, bringt Änderungen in den Cluster und entfernt Ressourcen, die aus dem Repo entfernt wurden (prune: true)
- Test: ein Dummy-Secret wurde erfolgreich per SOPS verschlüsselt, von Flux entschlüsselt und als K8s-Secret ins Cluster deployed → End-to-End-Funktion geprüft.

---

Damit ist ein funktionierendes Flux+SOPS-Gerüst vorhanden: Git-Repo + Cluster sind verbunden, Secrets-Entschlüsselung funktioniert, prune ist aktiv. Workloads können nun schrittweise als Unterordner unter apps/ eingebracht werden.