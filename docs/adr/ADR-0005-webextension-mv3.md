# ADR-0005 — Extension WebExtension Manifest V3

- **Statut** : Accepted
- **Date** : 2026-05-12
- **Décideurs** : Thibault, Cowork
- **Contexte** : Sprint S8

## Contexte

Le module M11 (extension navigateur) capture en vie réelle l'usage des LLMs sur Chrome et Firefox. Il faut choisir le format d'extension.

## Options envisagées

### A — Chrome Extension MV2
- ❌ Déprécié, retiré du Chrome Web Store en 2024-2025.

### B — Bookmarklet
- ❌ Pas de background, pas de persistance, UX dégradée.

### C — Application native + injection
- ❌ Sécurité dégradée, intrusif, refusée par les navigateurs.

### D — WebExtension Manifest V3 *(retenu)*
- ✅ Standard moderne supporté par Chrome 120+ et Firefox 120+ (parité MV3).
- ✅ Service worker, content scripts, storage API.
- ✅ Permissions granulaires, audit de sécurité strict.
- ❌ Pas de `eval`, pas de remote code — adapté à notre principe « tout local ».
- ❌ Quelques différences subtiles Chrome/Firefox (mitigées par `webextension-polyfill`).

## Décision

**WebExtension Manifest V3** avec TypeScript et `webextension-polyfill`. Build via `vite-plugin-web-extension` pour produire un seul codebase → Chrome `.crx` + Firefox `.xpi`.

## Permissions demandées (justifiées)

| Permission | Justification |
|------------|---------------|
| `activeTab` | Lire le DOM des interfaces LLM |
| `storage` | Sauvegarder agrégations locales |
| `notifications` | Bilan hebdo (opt-in) |
| host_permissions limitées | chatgpt.com, claude.ai, mistral.ai, chat.mistral.ai, gemini.google.com, chat.lechat.ai |

Pas de `tabs`, pas de `webRequest`, pas de `<all_urls>`.

## Liens
- CDC §5.11, NF-04, NF-10.
- ADR-0006 (licences).
