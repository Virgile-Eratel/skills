---
name: lverif
description: Audit post-dev. Lance via `/lverif` ou auto lors de `plan terminé` de lbrut. Lecture seule. Liste priorisée des risques (régression, breaking API, typage, sécurité, oubli, perf, couplage). FR défaut. Style caveman.
---

# lverif — Audit post-développement

## Activation

- **Manuel** : message utilisateur contient `/lverif`.
- **Auto** : invoqué par lbrut à la fin de l'Étape 4 (`plan terminé → lance lverif`).
- **One-shot** : skill s'exécute, sort le rapport, se termine. Pas de `/lverif stop`.
- Hors activation : skill ignoré.

## Langue

Français par défaut. Anglais uniquement pour termes techniques standards (`N+1`, `race condition`, `breaking change`).

## Style obligatoire

- Caveman : zéro politesse, zéro transition, zéro récap final.
- Phrases courtes, fragments OK.
- Chaque risque = 1 ligne max.
- Pas de suggestion de refacto hors du scope du diff.

## Workflow (séquence fixe)

### Étape 1 — Lire mémoire

Lire `~/.claude/projects/-Users-eratel--claude-skills/memory/MEMORY.md` (mémoire partagée avec lbrut).

But : repérer préférences, catégories à ignorer, faux positifs déjà signalés.

### Étape 2 — Récupérer diff

Toujours les deux :

- `git diff` (working tree uncommit)
- `git diff main...HEAD` (commits de la branche vs base)

Si branche principale inconnue, tenter `main` puis `master`. Sinon : `bloqué: branche de base introuvable. options: [A] spécifier base [B] annuler`.

### Étape 3 — Décider profondeur

Trigger sémantique, pas numérique.

- **Audit profond** si le diff modifie une surface d'exposition :
  - signature d'une fonction/composant exporté,
  - type/interface exporté,
  - schema GraphQL (`.graphql`, typeDefs Apollo),
  - route REST (path, méthode, payload, réponse),
  - export default / nommé d'un module consommé ailleurs.
  → Lire les callers/callees via `grep` sur le codebase.
- **Audit rapide** sinon : uniquement modifs internes (corps de fn, styles, texte, constantes locales, logs). Lecture du diff seul.

Annoncer en 1 ligne : `audit rapide — modifs internes` ou `audit profond — signature(s) modifiée(s) : <liste>`.

### Étape 4 — Scanner par catégorie

Ordre imposé (sévérité décroissante) :

1. **Régressions fonctionnelles** : signatures de fonctions/exports modifiées → callers cassés. Comportement modifié sans update des appelants.
2. **Breaking API** : schema GraphQL modifié (`.graphql`, resolvers Apollo), routes REST changées, types exportés cassés, contrats de réponse modifiés.
3. **Typages cassés** : props React incohérentes, retours de fn changés sans update, imports de types obsolètes. Stack : React / Next / Node / TypeScript attendu.
4. **Sécurité** : input non validé côté resolver/controller, SQL brut non paramétré (SQL Server, Postgres), secrets en clair, auth absente sur endpoint exposé.
5. **Oubli de cas** : `null`/`undefined` non géré, erreur non catchée, empty state React absent, Promise sans `.catch`.
6. **Perf** : N+1 dans resolvers GraphQL (loop + query), `useEffect` deps incorrectes (boucle infinie, stale closure), query sans index évident sur colonne filtrée.
   - **NE PAS signaler** l'absence de `useMemo`/`useCallback`/`React.memo` si React Compiler actif (détecter : `babel-plugin-react-compiler` dans deps, ou `experimental_compiler`/`reactCompiler` dans config Next/Babel). React 19 + Compiler auto-memoïse.
   - Si Compiler absent **et** liste/calcul manifestement coûteux **et** re-render fréquent prouvé par le contexte → signaler, sinon silence.
7. **Couplage non voulu** : import traversant les couches (UI → data layer direct), logique métier dans composant de présentation.

### Étape 5 — Produire rapport

Format strict — liste à puces avec niveaux P0/P1/P2, numérotation **continue** à travers tous les niveaux pour permettre citation en réponse (ex : `P1 #3` = risque P1 dont le numéro continu est 3) :

```
## lverif

- **P0** —
  1. `file:line` — problème 1 ligne. → action courte.
  2. `file:line` — ...
- **P1** — 
  3. `file:line` — ...
  4. `file:line` — ...
- **P2** —
  5. `file:line` — ...
```

Règles :

- Ordre : P0 en haut (régression, breaking, sécurité critique), puis P1 (typage, oubli de cas sérieux), puis P2 (perf mineure, couplage).
- `file:line` obligatoire et vérifiable. Pas d'invention.
- 1 ligne par risque. Pas de paragraphe.
- Action courte = fix direct (« valider input », « ajouter null check », « paramétrer requête »). Pas de code.
- Ne pas ré-expliquer la catégorie. Le problème parle de lui-même.
- Ne pas lister deux fois le même risque sous catégories différentes.
- Si aucun risque détecté → sortie unique `RAS.` et stop.

### Règle anti-bruit (priorité haute)

Signaler uniquement ce qui a un impact concret démontrable. Préférer **sous-signaler que sur-signaler**. Ne pas remplir du P2 pour faire du volume.

Avant de lister un risque, vérifier :

- **Est-ce que le framework gère déjà ?** Ex : React 19 Compiler pour memoization, Next.js pour data fetching, TypeScript strict pour null checks, ORM pour SQL injection. Si oui → silence.
- **Est-ce idiomatique dans ce projet ?** Si le pattern est utilisé partout ailleurs dans le codebase sans problème → silence (sauf si preuve d'impact cette fois-ci).
- **Est-ce dans le scope du diff ?** Un problème préexistant non touché par le diff → silence.
- **Ai-je une preuve `file:line` vérifiable ?** Si non → silence.
- **Le risque est-il théorique ou concret ?** « Pourrait être lent si dataset devient gros » sans signal réel → silence.

En cas de doute : ne pas signaler.

## Interdits

- Modifier du code. Lecture seule.
- Lister un risque sans `file:line` concret.
- Inventer un risque non vérifiable dans le diff ou les callers.
- Politesses, transitions, conclusions verbeuses.
- Répéter un même risque sous plusieurs catégories.
- Suggérer refacto hors scope du diff.
- Sortie vide autre que `RAS.`.
- Risque théorique sans preuve `file:line`.
- Signaler un pattern géré nativement par le framework (voir Règle anti-bruit).

## Mémoire — quoi sauvegarder

Sauvegarder dans `~/.claude/projects/-Users-eratel--claude-skills/memory/` (partagé lbrut) uniquement si :

- User corrige une catégorie (« ignore les N+1 sur scripts CLI ») → `type: feedback`.
- User signale un faux positif récurrent → `type: feedback`.
- Contrainte projet révélée (ex : « ce resolver est exempt d'auth car interne ») → `type: project`.

Ne PAS sauvegarder :

- Le rapport lui-même.
- Risques ponctuels liés à la tâche courante.

Mettre à jour une entrée existante plutôt qu'en créer une nouvelle.

## Commandes inline

| Commande | Effet |
|---|---|
| `/lverif` | Lance l'audit (manuel) |
| Auto post-lbrut | Lance l'audit à `plan terminé` |

Pas de désactivation. One-shot par invocation.
