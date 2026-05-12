# ADR-0009 — Architecture médaillon (Copper / Silver / Gold) pour le traitement de la donnée

- **Statut** : Accepted
- **Date** : 2026-05-12
- **Décideurs** : Thibault, Cowork
- **Contexte** : Sprint S2 (avant tout travail d'ingestion)

## Contexte et énoncé du problème

Sobr.ia agrège des données hétérogènes (ADEME, RTE, Hugging Face, EcoLogits, CodeCarbon, ML.Energy, papers PDF…) de qualité, fréquence, format et licence variables. Sans discipline, on tombe dans le piège classique du *spaghetti ETL* : transformations *ad hoc*, recalculs non reproductibles, perte de traçabilité scientifique.

Or notre projet a deux exigences contradictoires :
1. **Souveraineté scientifique** — pouvoir remonter de chaque chiffre publié jusqu'à la donnée brute et justifier chaque transformation.
2. **Performance applicative** — l'app et la doc doivent lire vite, sans rejouer les transformations.

L'architecture médaillon (popularisée par Databricks, généralisée dans le monde data engineering) répond aux deux : on conserve le brut, on raffine en étapes nommées, on ne lit que la couche utile.

## Décision

Adopter une **architecture médaillon à 3 couches**, implémentée **automatiquement** via un trait Rust unique appliqué à chaque source. Pipeline orchestré par DVC, exécutable d'une seule commande.

### Couches

#### 🟫 Copper Layer — *raw, immutable, source-of-truth*

- Données brutes telles que reçues de la source.
- Format d'origine préservé (JSON, CSV, PDF, XML, HTML scrap).
- **Append-only** : on n'écrase jamais, on ajoute un snapshot daté.
- Métadonnées obligatoires : URL, timestamp UTC, hash SHA-256, headers HTTP, signature mTLS si dispo, licence détectée.
- Aucun nettoyage, aucune validation autre que l'intégrité du transfert.

```
data/copper/
├── ademe-base-empreinte/
│   ├── 2026-05-12/
│   │   ├── factors-electricity.csv
│   │   └── manifest.json   ← URL, hash, ts, license
│   └── 2026-06-15/
│       └── ...
├── rte-eco2mix/
├── hf-energy-score/
├── ecologits-models/
├── codecarbon-runs/
├── ml-energy-leaderboard/
├── papers/
│   ├── luccioni-2023-bloom.pdf
│   └── manifest.json
└── geolite2/
```

#### 🥈 Silver Layer — *cleaned, conformed, validated*

- Format unifié : **Parquet** (colonnaire, compressé Zstd, fast scan).
- Une entité (table) par source : `models.parquet`, `datacenters.parquet`, `emission_factors.parquet`…
- Schémas figés et versionnés (`schemas/silver/<entity>-v<n>.json`).
- Validation à l'écriture (Rust : `arrow-schema` + JSON Schema via `schemars`).
- Déduplication, normalisation des unités SI, harmonisation des codes pays (ISO 3166), conversion des dates en UTC.
- **Lignée (lineage)** conservée : chaque ligne Silver pointe vers son hash Copper d'origine.
- Pas de jointure inter-source à ce stade : chaque source est cantonnée à son propre Parquet.

```
data/silver/
├── ademe/
│   ├── electricity_factors.parquet
│   └── hardware_factors.parquet
├── rte/
│   └── mix_hourly.parquet
├── hf/
│   └── model_energy_score.parquet
├── ecologits/
│   └── models.parquet
├── codecarbon/
│   └── training_runs.parquet
├── ml_energy/
│   └── inference_benchmarks.parquet
├── papers/
│   └── extracted_measures.parquet
└── geolite2/
    └── ip_to_zone.parquet
```

#### 🥇 Gold Layer — *business-ready, consumption-optimized*

- **Deux artefacts cible** :
  - `gold/referentiel.sqlite` → lu par l'app Tauri (ACID, indexé, FTS5 pour recherche).
  - `gold/analytics.parquet` → lu par DuckDB pour scénarios macro et notebook Quarto.
- Jointures, déduplications inter-sources, résolution des conflits (règles documentées par champ).
- Vues matérialisées orientées usage : `model_overview`, `scenario_inputs`, `time_series_mix`, `comparison_matrix`.
- Datasheet (Gebru et al. 2018) embarquée comme JSON-LD `datasheet.jsonld`.
- Signature d'intégrité : `MANIFEST.sha256` + signature GPG du mainteneur.

```
data/gold/
├── referentiel.sqlite              ← lu par l'app
├── analytics.parquet               ← lu par DuckDB
├── datasheet.jsonld                ← métadonnées scientifiques
└── MANIFEST.sha256                 ← intégrité (signé GPG)
```

## Implémentation automatique

### Trait unique par source

Chaque source du référentiel implémente le trait `DataLayer` exposé par `sobria-ingest` :

```rust
// crate sobria-ingest, src/layer.rs

#[async_trait]
pub trait DataLayer: Send + Sync {
    /// Identifiant stable et unique de la source.
    fn id(&self) -> &'static str;

    /// Métadonnées (licence, URL, fréquence de mise à jour…).
    fn meta(&self) -> SourceMeta;

    /// Étape 1 — récupération brute → Copper layer.
    /// Doit être idempotente : un même snapshot doit produire le même hash.
    async fn ingest_copper(&self, ctx: &Context) -> Result<CopperSnapshot>;

    /// Étape 2 — promotion vers Silver.
    /// Reçoit le snapshot Copper, écrit un (ou plusieurs) Parquet validé.
    async fn promote_silver(
        &self,
        snapshot: &CopperSnapshot,
        ctx: &Context,
    ) -> Result<Vec<SilverEntity>>;

    /// Étape 3 — contribution à Gold.
    /// Retourne les fragments SQL/SQL-Arrow à intégrer au merge Gold.
    async fn contribute_gold(
        &self,
        silver: &[SilverEntity],
        ctx: &Context,
    ) -> Result<GoldContribution>;
}
```

### Registry et orchestrateur

```rust
// crate sobria-ingest, src/registry.rs

pub struct LayerRegistry {
    sources: Vec<Box<dyn DataLayer>>,
}

impl LayerRegistry {
    pub fn default() -> Self {
        Self {
            sources: vec![
                Box::new(AdemeBaseEmpreinteSource::new()),
                Box::new(RteEco2MixSource::new()),
                Box::new(ElectricityMapsSource::new()),
                Box::new(HuggingFaceEnergyScoreSource::new()),
                Box::new(EcoLogitsSource::new()),
                Box::new(CodeCarbonSource::new()),
                Box::new(MlEnergyLeaderboardSource::new()),
                Box::new(PapersSource::new()),
                Box::new(GeoLite2Source::new()),
            ],
        }
    }

    pub async fn run_full_pipeline(&self, ctx: &Context) -> Result<PipelineReport> { … }
}
```

### Commande unique

```bash
# Récupère, valide, normalise, agrège, produit referentiel.sqlite + analytics.parquet
cargo run -p sobria-ingest -- pipeline run

# Ou en mode incrémental (ne ré-ingère que ce qui a changé)
cargo run -p sobria-ingest -- pipeline run --incremental

# Source unique (debug / dev)
cargo run -p sobria-ingest -- pipeline run --source rte-eco2mix
```

### Orchestration DVC

`dvc.yaml` à la racine déclare chaque transition comme une *stage* :

```yaml
stages:
  copper:
    cmd: cargo run -p sobria-ingest -- copper --all
    deps:
      - crates/sobria-ingest/src/sources
    outs:
      - data/copper:
          cache: true
          push: true

  silver:
    cmd: cargo run -p sobria-ingest -- silver --all
    deps:
      - data/copper
      - schemas/silver
    outs:
      - data/silver

  gold:
    cmd: cargo run -p sobria-ingest -- gold
    deps:
      - data/silver
      - schemas/gold
    outs:
      - data/gold/referentiel.sqlite
      - data/gold/analytics.parquet
      - data/gold/datasheet.jsonld
      - data/gold/MANIFEST.sha256
```

`dvc repro` rejoue uniquement les étages dont les inputs ont changé. La CI nocturne exécute `dvc repro && dvc push` automatiquement.

## Garanties produites par l'architecture

| Garantie | Mécanisme |
|----------|-----------|
| Traçabilité scientifique de bout en bout | Lineage (`copper_hash` propagé jusqu'au Gold) |
| Reproductibilité | DVC + seeds + builds déterministes |
| Idempotence | `ingest_copper` doit produire le même hash pour un même snapshot |
| Validation des schémas | JSON Schema + arrow-schema à chaque écriture Silver |
| Pas de régression silencieuse | Tests `proptest` + golden files sur chaque transformation |
| Lecture rapide en prod | Gold = SQLite indexé + Parquet colonnaire |
| Versionnage temporel | Snapshots Copper datés permettent le *time travel* |

## Conséquences

**Positives** :
- Discipline ETL professionnelle, défendable devant n'importe quel jury data.
- Chaque chiffre Sobr.ia est traçable jusqu'à la donnée brute originale.
- Le code d'ingestion devient mécaniquement uniforme entre sources.
- Onboarding d'une nouvelle source = implémenter un seul trait.
- Le notebook Quarto peut citer les hashes Copper comme références bibliographiques.

**Négatives** :
- Coût de stockage triple (mais Copper est compressible et purgable au-delà de N snapshots).
- Discipline d'ingénierie plus forte requise.
- Plus de cérémonie au démarrage de chaque nouvelle source.

**Neutres** :
- L'app finale ne « voit » que la couche Gold — l'utilisateur final n'est pas exposé à la complexité.

## Politique de rétention Copper

- Snapshots des **30 derniers jours** : conservation complète.
- Au-delà : conservation **mensuelle** (premier jour du mois) pendant 2 ans.
- Au-delà : conservation **annuelle** indéfiniment.
- DVC garbage collection automatisé en CI hebdo.

## Évolutions futures (hors v1.0)

- Couche **Platinum** pour données dérivées scientifiquement (résultats Monte-Carlo agrégés, projections).
- Materialized views incrémentales (DuckDB 1.x supportera mieux).
- Plugin Quarto pour citer automatiquement les sources Copper dans le notebook.

## Liens

- ADR-0003 (SQLite + DuckDB).
- ADR-0007 (DVC).
- CDC §5.1 (Module M1 Référentiel), §8 (Sources).
- Référence : Bronze/Silver/Gold pattern (Databricks, 2020+). Notre nomenclature « Copper » remplace « Bronze » par préférence projet.
