# ADR-0008 — Observable Plot + D3 pour la dataviz

- **Statut** : Accepted
- **Date** : 2026-05-12
- **Décideurs** : Thibault, Cowork
- **Contexte** : Sprint S7-S8

## Contexte

La dataviz est un livrable visible et différenciant. Elle doit être : déclarative quand possible, customisable quand nécessaire, accessible (a11y), exportable (PNG/SVG), et fluide.

## Options envisagées

### A — Chart.js
- ✅ Simple.
- ❌ Limité pour visualisations complexes (Sankey, treemap, ridge).

### B — Plotly.js
- ✅ Interactivité forte.
- ❌ ~3 Mo gzip, antinomique frugalité.

### C — D3 pur
- ✅ Customisation totale.
- ❌ Trop bas niveau pour chaque graphique.

### D — Observable Plot + D3 *(retenu)*
- ✅ Plot = grammar of graphics déclarative.
- ✅ D3 disponible pour custom (Sankey énergétique, treemap critique).
- ✅ Bundle modeste (~100 Ko gzip pour Plot core).
- ✅ Exports SVG natifs.
- ❌ Documentation un peu jeune (mais maintenue par Observable).

## Décision

**Observable Plot pour 80 % des graphes, D3 pour les 20 % exotiques** (Sankey énergie, treemap critique, animations custom).

## Inventaire des graphes prévus

| Graphe | Lib | Module |
|--------|-----|--------|
| Barres comparaison modèles | Plot | Comparateur M5 |
| Aire scénario temporel | Plot | Simulateur M4 |
| Bande d'incertitude P5-P95 | Plot | Estimateur M2 |
| Sankey flux énergétiques | D3 | Estimateur M2 |
| Treemap consommation par catégorie | D3 | Workbench M3 |
| Carte choroplèthe datacenter | Plot (geo) | Géoloc M9 |
| Histogramme distribution Monte-Carlo | Plot | Estimateur M2 |
| Heatmap matrice modèles × indicateurs | Plot | Comparateur M5 |

## a11y

- Tous les graphes : titres + description textuelle (aria-label).
- Mode tableau alternatif (`<table>`) accessible par lecteurs d'écran.
- Palettes daltoniens (Viridis, Cividis).

## Liens
- CDC §5, NF-12.
- ADR-0002.
