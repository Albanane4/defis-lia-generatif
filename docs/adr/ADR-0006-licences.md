# ADR-0006 — Licences MIT + Etalab 2.0 + CC-BY 4.0

- **Statut** : Accepted
- **Date** : 2026-05-12
- **Décideurs** : Thibault, Cowork
- **Contexte** : Sprint S1

## Contexte

Sobr.ia est open-source et alimenté par des données ouvertes. La cohérence des licences est cruciale pour permettre la réutilisation, la contribution, et la diffusion sur data.gouv.fr.

## Décision

| Artefact | Licence | Justification |
|----------|---------|---------------|
| Code source (Rust, TS, etc.) | **MIT** | Permissive, compatible Apache 2.0 et GPL, écosystème Rust |
| Dataset publié sur data.gouv.fr | **Etalab 2.0** | Standard data.gouv.fr, compatible CC-BY 4.0 |
| Documentation | **CC-BY 4.0** | Réutilisation libre avec attribution |
| Logo / identité visuelle | **CC-BY-SA 4.0** | Préserve l'identité du projet |

## Compatibilités vérifiées

- MIT ↔ Apache 2.0 : OK (la plupart des deps Rust).
- MIT ↔ GPL : compatible en redistribution GPL.
- Etalab 2.0 ↔ CC-BY 4.0 : reconnues équivalentes par la France et l'UE.
- Etalab 2.0 ↔ CC-BY-SA 4.0 : compatible avec attribution.

## Sources tierces — vérification par source

Chaque source ingérée dans le référentiel est vérifiée avant intégration. Catalogue dans `docs/sources/CATALOGUE-SOURCES.md` colonne « Licence ». Les sources incompatibles (ex: certains scrapes commerciaux) sont exclues ou citées sans intégration directe.

## Conséquences

**Positives** : maximisation de la diffusion et de la réutilisation.
**Négatives** : MIT laisse possibilité de fork propriétaire (acceptable, on garde la mission).

## Liens
- CDC §0, §12, NF-15.
