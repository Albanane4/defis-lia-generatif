# ADR-0004 — Propagation d'incertitude par Monte-Carlo N=10⁴

- **Statut** : Accepted
- **Date** : 2026-05-12
- **Décideurs** : Thibault, Cowork (+ relecture mentor Ecolab à venir)
- **Contexte** : Sprint S4

## Contexte

L'évaluation environnementale d'un usage GenAI repose sur une chaîne de paramètres tous incertains : énergie par token, PUE du datacenter, facteur d'émission de l'électricité, embodied carbon hardware, durée d'amortissement, etc. Présenter une valeur ponctuelle (« 3.14 gCO₂eq ») est trompeur et scientifiquement discutable.

AFNOR SPEC 2314 demande explicitement la transparence sur les hypothèses et la prise en compte de l'incertitude.

## Options envisagées

### A — Estimation ponctuelle (P50 seul)
- ✅ Simple, rapide.
- ❌ Trompe l'utilisateur, scientifiquement faible, non conforme SPEC 2314.

### B — Intervalles min/max déterministes
- ✅ Plus honnête, peu coûteux.
- ❌ Compose mal sur 5+ paramètres (intervalles deviennent énormes ou irréalistes).
- ❌ N'exprime pas la distribution réelle.

### C — Propagation analytique (séries de Taylor)
- ✅ Exact pour distributions simples.
- ❌ Très lourd à implémenter pour distributions composées.
- ❌ Ne gère pas bien les non-linéarités et corrélations.

### D — Monte-Carlo N=10⁴ *(retenu)*
- ✅ Robuste, généraliste, gère toutes distributions et corrélations.
- ✅ Restitue P5, P25, P50, P75, P95.
- ✅ Suffisamment rapide en Rust (cible < 200 ms par estimation).
- ❌ Légère stochasticité d'exécution (mitigée par seed).

## Décision

**Monte-Carlo N=10⁴ par estimation, seed déterministe** (`SOBRIA_SEED`, défaut 42). Implémenté dans `sobria-estimator`.

### Distributions par paramètre (par défaut, ajustables)

| Paramètre | Distribution | Justification |
|-----------|--------------|---------------|
| Énergie/token | Log-normale | Mesures Luccioni, Patterson : variance multiplicative |
| PUE | Uniforme bornée | Plage [1.05, 1.6] par datacenter |
| Facteur émission | Discrète (mix réel) | Lecture RTE / Electricity Maps |
| Embodied/req | Log-normale | Forte incertitude amortissement |
| Tokens entrée | Déterministe ou uniforme | Selon entrée utilisateur |
| Tokens sortie | Log-normale autour estimation | Variance documentée |

## Performance attendue

- N=10⁴ × 6 paramètres = 60 000 ops scalaires en Rust → cible < 50 ms en release mode.
- Génération RNG : `rand` + `rand_distr` (xorshift / pcg).
- Parallélisable par rayon `par_iter` si plusieurs estimations en batch.

## Conséquences

**Positives** :
- Conformité méthodologique forte.
- Restitution honnête au public.
- Robuste aux changements de paramètres.

**Négatives** :
- Code plus complexe que l'estimation ponctuelle.
- Output UI plus riche → effort dataviz supplémentaire.

## Liens
- CDC §9.
- ADR-0001 (Rust pour la perf).
- Référence : AFNOR SPEC 2314, ISO/IEC GUM (Guide to expression of Uncertainty in Measurement).
