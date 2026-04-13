### Flux-Status im Cluster prüfen

#### Kustomizations anzeigen

```bash
kubectl get kustomizations -n flux-system
```

Zeigt alle Flux-Kustomizations mit Status `READY` (True/False) und ggf. Fehlermeldung.

#### Details zu einer fehlgeschlagenen Kustomization

```bash
kubectl describe kustomization infrastructure -n flux-system
kubectl describe kustomization cert-issuers -n flux-system
kubectl describe kustomization apps -n flux-system
```

Im `Events`- und `Status`-Abschnitt steht die genaue Fehlermeldung.

#### HelmReleases prüfen

```bash
kubectl get helmreleases -A
```

Zeigt alle Helm-Deployments mit `READY`-Status.

#### Flux-Logs ansehen

```bash
kubectl logs -n flux-system deploy/kustomize-controller
kubectl logs -n flux-system deploy/helm-controller
kubectl logs -n flux-system deploy/source-controller
```

#### GitRepository-Status (Sync vom Git)

```bash
kubectl get gitrepositories -n flux-system
```

Hier sieht man, ob Flux das Git-Repo erfolgreich liest.

#### Flux CLI (empfohlen, falls installiert)

```bash
flux get all -n flux-system
flux get kustomizations
flux get helmreleases -A
```

Gibt eine übersichtliche Zusammenfassung aller Flux-Ressourcen mit Status und Fehlermeldungen.

#### Manuellen Sync erzwingen

```bash
flux reconcile kustomization infrastructure --with-source
```

Erzwingt sofortigen Sync ohne auf das Intervall zu warten – nützlich nach Git-Änderungen.

---

### Typische Fehlerquellen

| Problem | Erkennbar an |
|---|---|
| Falscher `path` in Kustomization | `kustomization.yaml not found` in Events |
| Abhängigkeit nicht bereit | `dependency not ready` in Status |
| HelmRelease schlägt fehl | `READY: False` bei `kubectl get helmreleases -A` |
| Git nicht erreichbar | `GitRepository` zeigt `READY: False` |
| Secret fehlt | `secret not found` in Helm-Controller-Logs |