# Runtime OCI automerge policy

**Stato:** Draft

**Data:** 2026-08-17

## Missione

Automatizzare la promozione GitOps delle immagini OCI proprietarie che rappresentano workload runtime continuativi, riusando Renovate come unico updater già autorevole e senza introdurre workflow cross-repository, webhook, PAT o chiamate imperative ad Argo CD.

## Contesto verificato

- `default.json` è la fonte autorevole dell'inventario dei package OCI proprietari con CalVer `YYYY.MM.DD_<run_number>`.
- `automerge.json` è il preset condiviso che governa l'automerge dei consumer che lo estendono.
- Homelab usa Renovate per aggiornare i riferimenti Kubernetes e Argo CD riconcilia automaticamente `main`.
- I package one-shot o migration non devono essere promossi automaticamente solo perché è disponibile una nuova immagine.

## Decisione approvata

Abilitare l'automerge Renovate esclusivamente per gli aggiornamenti Docker dei seguenti runtime proprietari, sia con nome GHCR diretto sia con il path consumer Harbor `private-ghcr`:

- `developer-workspace`
- `aeris`
- `skunklabs`
- `baialupo.com`
- `iwant`
- `club-aviazione-popolare-web`
- `club-aviazione-popolare-cms`

Restano esclusi dall'automerge automatico:

- `iwant-migrator`
- `prosignal`

Le esclusioni sono intenzionali perché questi artifact sono associati a job one-shot, migration o flussi che richiedono una decisione di promozione distinta dal semplice aggiornamento di un workload runtime continuativo.

## Implementazione prevista

Aggiungere in `automerge.json` una `packageRule` finale e più specifica che:

- seleziona `datasource: docker`;
- seleziona esplicitamente solo i package runtime proprietari sopra elencati;
- abilita `automerge: true`;
- mantiene `automergeType: pr`;
- non modifica cooldown, versioning CalVer, manager ownership o altre policy condivise.

La regola deve essere più specifica della regola Docker generica che oggi disabilita l'automerge, così da riabilitarlo soltanto per questo insieme chiuso di package.

## Flusso risultante

1. Il repository producer pubblica un nuovo CalVer immutabile dopo i propri gate di build/test/security.
2. Renovate rileva il nuovo CalVer nel manifest Homelab.
3. Renovate crea la PR di aggiornamento.
4. I check richiesti dal repository consumer devono risultare soddisfatti.
5. Renovate esegue l'automerge della PR.
6. Argo CD riconcilia `main` e aggiorna il workload runtime.

Nessun producer scrive direttamente nel repository Homelab e nessun workflow del producer invoca il deploy.

## Verifiche

La modifica è accettabile solo se è verificabile che:

- `automerge.json` resta JSON valido;
- ogni runtime proprietario approvato compare nella nuova regola nelle due forme attese, GHCR e `private-ghcr`;
- `iwant-migrator` e `prosignal` non compaiono nella regola di automerge runtime;
- nessun package esterno o OCI proprietario non approvato riceve automerge per effetto della modifica;
- le policy esistenti di minimum release age e versioning CalVer restano invariate;
- non vengono aggiunti workflow, credenziali, webhook o componenti custom.

## Alternative scartate

### Regola locale soltanto in Homelab

Funzionale, ma duplicherebbe nel consumer un'informazione di ownership già centralizzata nel preset condiviso.

### Automerge di tutti i package Docker proprietari

Più semplice sintatticamente ma troppo ampio: includerebbe artifact one-shot/migration per cui l'aggiornamento del riferimento non equivale a una promozione runtime sicura.

### Workflow cross-repository dal producer

Scartato perché introdurrebbe credenziali e automazione custom duplicando Renovate e violando il principio di semplicità e unicità della fonte operativa.

## Criterio di chiusura

La missione è completata quando il preset condiviso abilita l'automerge soltanto per i runtime proprietari approvati, le esclusioni one-shot restano effettive e la configurazione risulta verificata senza introdurre nuovi meccanismi di delivery.