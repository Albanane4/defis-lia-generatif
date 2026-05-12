# ADR-0002 — SvelteKit + TypeScript pour le frontend

- **Statut** : Accepted
- **Date** : 2026-05-12
- **Décideurs** : Thibault, Cowork
- **Contexte** : Sprint S1

## Contexte

Tauri 2 ne contraint pas le frontend : n'importe quel framework web peut y être embarqué. Le choix doit optimiser : taille du bundle final, fluidité de la dataviz, accessibilité, et productivité solo sur 12 semaines.

## Options envisagées

### A — React + TanStack
- ✅ Écosystème massif, recrutable.
- ❌ Runtime ~45 Ko gzip minimum, plus lourd que les alternatives.
- ❌ Virtual DOM non optimal pour dataviz haute fréquence.

### B — SolidJS
- ✅ Signaux fins, performance brute.
- ✅ Bundle minuscule.
- ❌ Écosystème plus jeune, moins de ressources d'apprentissage.
- ❌ Moins d'intégrations dataviz prêtes à l'emploi.

### C — Vue 3
- ✅ DX agréable, écosystème mature.
- ❌ Runtime plus gros que Svelte.
- ❌ Pas un gain net face à Svelte pour ce projet.

### D — SvelteKit + TypeScript *(retenu)*
- ✅ Compilation à la build → quasi pas de runtime.
- ✅ Bundle final ultra-léger (cohérence frugale).
- ✅ Réactivité naturelle, syntaxe minimaliste.
- ✅ Excellent avec Observable Plot / D3.
- ✅ Stores Svelte typés, support TS premier rang.
- ❌ Communauté plus petite que React.
- ❌ SvelteKit côté SPA (sans SSR) est le mode d'emploi ici, certains patterns à adapter.

## Décision

**SvelteKit 2 + TypeScript strict** en mode SPA, embarqué dans Tauri.

## Conséquences

**Positives** :
- Bundle ≤ 200 Ko gzip réaliste.
- Dataviz fluide même sur appareils modestes.
- Productivité solo (moins de boilerplate).

**Négatives** :
- Moins de libs UI prêtes (compensé par Skeleton CSS custom).
- Recrutement futur moins large.

## Conventions associées

- Pas de framework UI lourd (pas de Material-UI, etc.). CSS custom + variables.
- Stores typés Svelte 5 (runes) si la version est stable au démarrage S1.
- Composants en kebab-case dossier, PascalCase fichier.
- Tests : Vitest unit + Playwright e2e.

## Liens
- ADR-0001, ADR-0008.
- CDC §7.1.
