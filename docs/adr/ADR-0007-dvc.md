# ADR-0007 — DVC pour le versionnage du référentiel

- **Statut** : Accepted
- **Date** : 2026-05-12
- **Décideurs** : Thibault, Cowork
- **Contexte** : Sprint S2

## Contexte

Le référentiel Sobr.ia agrège des données qui évoluent (mix électrique RTE temps réel, nouveaux modèles LLM publiés, mises à jour facteurs ADEME). Le SQLite peut peser ≥ 100 Mo après agrégation. Git n'est pas conçu pour des fichiers binaires lourds. Il faut **versionner le dataset comme on versionne le code**.

## Décision

Adopter **DVC** (Data Version Control) :
- Métadonnées (`.dvc` files) commitées dans Git.
- Données stockées sur un remote (S3-compatible, Cloudflare R2 envisagé pour la gratuité).
- Pipeline `dvc.yaml` pour reproductibilité de bout en bout (sources → couches Copper → Silver → Gold).
- CalVer (YYYY.MM.DD) pour les snapshots du référentiel.

## Pourquoi pas Git LFS ?

- Quotas GitHub LFS strictes (1 Go gratuit), payant rapidement.
- Pas de pipelines reproductibles natifs.
- Pas d'orientation data science.

## Conséquences

**Positives** : reproductibilité scientifique, séparation propre code/données.
**Négatives** : DVC ajoute une dépendance au workflow contributeur.

## Liens
- CDC §8, §12, ADR-0009 (medallion).
