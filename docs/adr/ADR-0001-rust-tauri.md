# ADR-0001 — Choisir Rust + Tauri 2 comme stack native

- **Statut** : Accepted
- **Date** : 2026-05-12
- **Décideurs** : Thibault, Cowork
- **Contexte** : Sprint S1 (cadrage technique)

## Contexte et énoncé du problème

Sobr.ia doit être une application multi-plateforme (Windows, macOS, Linux, web, mobile à terme) qui :
- mesure et visualise l'impact environnemental de l'IA générative,
- incarne par sa propre frugalité le sujet qu'elle traite,
- traite des volumes de données significatifs (référentiel, audit ledger, scénarios macro).

Il faut donc choisir un **runtime applicatif** qui soit simultanément multi-plateforme, performant, frugal en ressources, et adapté à la livraison d'un binaire signé.

## Options envisagées

### A — Electron + Node.js + React
- ✅ Écosystème massif, recrutable, dataviz JS facile.
- ❌ Binaire 100-150 Mo, RAM 300-500 Mo : antinomique avec le sujet.
- ❌ Une instance Chromium par app = caricature de gaspillage.
- ❌ Politique anti-Electron explicite du projet (cohérence frugale).

### B — Native (C++ / Qt ou C# / WPF)
- ✅ Performance maximale.
- ❌ Cross-platform difficile, courbe d'apprentissage hétérogène.
- ❌ Pas de cible web facile.
- ❌ Toolchain plus lourde, moins moderne.

### C — Flutter
- ✅ Cross-platform mobile + desktop + web.
- ❌ Dart : écosystème data science / scientifique pauvre.
- ❌ Binaire moins compact que Rust.
- ❌ Skia rendering — joli mais lourd.

### D — Rust + Tauri 2 *(retenu)*
- ✅ Binaire 5-15 Mo, RAM 30-80 Mo.
- ✅ Backend Rust = performance brute + sûreté mémoire.
- ✅ Tauri 2 supporte desktop + Android + iOS + Wasm.
- ✅ Webview système (pas de Chromium embarqué).
- ✅ Cohérence idéologique parfaite avec le pitch frugal.
- ❌ Courbe d'apprentissage Rust significative.
- ❌ Webviews varient légèrement entre OS (WebView2, WKWebView, WebKitGTK).
- ❌ Tauri mobile encore jeune (mais stable en 2.0+).

## Décision

**Rust + Tauri 2** est retenu. La cohérence sujet/médium et la frugalité priment sur la courbe d'apprentissage, qui est mitigée par l'allocation d'une semaine S0 de revue/montée en compétence.

## Conséquences

**Positives** :
- Empreinte mémoire et binaire minimales → argument fort dans le dossier.
- Sécurité mémoire native.
- Toolchain unique (Cargo) côté backend.
- Tauri 2 = un seul codebase pour 6 cibles.

**Négatives** :
- Cycle de feedback initial plus lent (compilation Rust).
- Tests des webviews à automatiser par cible (Playwright).
- Mobile en bonus, pas une garantie absolue.

**Neutres** :
- Choix du frontend reste ouvert (ADR-0002).

## Liens
- CDC §7.1, §7.2, NF-01 à NF-04.
- ADR-0002 (frontend), ADR-0003 (storage).
