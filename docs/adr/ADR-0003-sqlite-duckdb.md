# ADR-0003 — Dual-store SQLite (ACID) + DuckDB (OLAP)

- **Statut** : Accepted
- **Date** : 2026-05-12
- **Décideurs** : Thibault, Cowork
- **Contexte** : Sprint S1

## Contexte

Sobr.ia doit gérer deux profils de charge antagonistes :

1. **Référentiel + audit ledger** : écritures atomiques, intégrité forte, lecture aléatoire, taille modeste (< 100 Mo). ACID nécessaire pour la traçabilité réglementaire (CSRD).
2. **Scénarios macro + agrégations** : lecture massive, jointures lourdes, agrégations colonne, datasets Parquet potentiellement larges (> 1 Go).

Un seul moteur impose un compromis. Deux moteurs spécialisés cohabitent très bien en Rust.

## Options envisagées

### A — SQLite seul
- ✅ Simplicité.
- ❌ Pas optimisé pour OLAP, ralentissements sur les agrégations larges.

### B — DuckDB seul
- ✅ Excellent OLAP.
- ❌ Pas conçu pour writes concurrents transactionnels intensifs.
- ❌ Stockage moins universel que SQLite.

### C — PostgreSQL embarqué
- ❌ Complexité énorme pour app desktop.
- ❌ Pas embarquable proprement.

### D — SQLite + DuckDB *(retenu)*
- ✅ SQLite pour référentiel + audit ledger ACID.
- ✅ DuckDB pour scénarios + lecture Parquet en zéro-copie.
- ✅ Les deux ont des crates Rust matures (`rusqlite`, `duckdb-rs`).
- ✅ Pas de réseau, pas de service à gérer.
- ❌ Deux moteurs = deux schémas à maintenir.

## Décision

**SQLite (mode WAL) + DuckDB**, accédés via Rust :
- `sobria-referentiel` : SQLite (référentiel modèles, datacenters, facteurs).
- `sobria-audit` : SQLite (ledger, journal chaîné SHA-256).
- `sobria-estimator` (mode batch / scénarios) : DuckDB pour requêtes OLAP, lit le SQLite ou des Parquet.

## Schéma directeur (esquisse)

```
SQLite — référentiel
├── models (id, name, provider, params, …)
├── datacenters (id, provider, region, pue, wue, …)
├── emission_factors (id, country, year, gCO2_per_kWh, source_id, …)
├── sources (id, url, doi, license, fetched_at, hash, …)
└── methodology_assumptions (id, key, distribution, params_json, …)

SQLite — audit
└── estimations (id, ts, params_hash, result_json, prev_hash, sig)

DuckDB — analytique (vues sur Parquet + lectures SQLite)
├── view_scenarios (matérialisée à la demande)
└── view_model_compare
```

## Conséquences

**Positives** :
- Garanties ACID strictes là où elles comptent.
- Performance OLAP excellente sans serveur.
- Migration simple (export Parquet possible).

**Négatives** :
- Maintenance double schéma.
- Onboarding développeur légèrement plus complexe.

## Liens
- ADR-0001.
- CDC §7.1, EF-M7-*.
