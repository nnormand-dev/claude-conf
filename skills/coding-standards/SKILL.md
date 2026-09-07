---
name: coding-standards
description: >
  Garde-fous de qualité à appliquer dès que l'utilisateur demande de
  générer, écrire, implémenter, refactorer ou corriger du code. Couvre
  KISS, DRY, YAGNI, SOLID, lisibilité, fonctions courtes, détection précoce
  des erreurs, gestion sûre des ressources et conventions du codebase, plus
  des règles spécifiques par langage (C++, C, Python, Bash). À utiliser pour
  toute production de code, même si l'utilisateur ne mentionne pas
  explicitement « qualité » ou « standards ».
---

# Standards de génération de code

Quand tu écris, refactores ou corriges du code, applique ces principes.
Ils priment sur la rapidité : un code clair et juste vaut mieux qu'un code
livré vite.

## Spécificités par langage — à lire AVANT d'écrire

Ces principes sont universels. Chaque langage a en plus ses règles
idiomatiques. Selon le langage du code que tu produis, lis le fichier de
référence correspondant avant d'écrire :

- **C++** → lis `references/cpp.md` (RAII, compile-time, Rule of Zero,
  const-correctness, smart pointers…).
- **C pur** → lis `references/c.md` (discipline de libération, `goto
  cleanup`, propriété explicite…).
- **Python** → lis `references/python.md` (context managers, type hints +
  mypy, immuabilité…).
- **Bash** → lis `references/bash.md` (`set -euo pipefail`, quoting,
  `trap` de nettoyage, shellcheck…).

## Avant d'écrire la moindre ligne

- **Cherche l'existant d'abord.** Avant d'implémenter une fonctionnalité,
  vérifie si elle existe déjà dans le codebase (recherche par mots-clés,
  grep, exploration des modules utilitaires). Ne réinvente pas ce qui est
  déjà là.
- **Bonne frontière, bon emplacement.** Demande-toi si ce code relève d'une
  bibliothèque réutilisable (générique, sans dépendance au domaine métier)
  plutôt que du programme principal. Si oui, propose-le explicitement avant
  d'écrire.
- **Comprends l'intention.** Si la demande est ambiguë, pose une question
  plutôt que de partir sur une hypothèse silencieuse. Ne devine pas en
  fonçant.
- **Suis les conventions en place.** Aligne-toi sur le style, les patterns,
  le nommage et les outils déjà présents dans le projet. La cohérence prime
  sur tes préférences personnelles.

## Principes de conception

- **KISS** — Choisis la solution la plus simple qui résout le problème
  réel. Pas d'abstraction spéculative, pas de cleverness inutile.
- **YAGNI** — N'implémente que ce qui est demandé maintenant. Pas de
  paramètres, hooks ou points d'extension « au cas où ».
- **DRY** — Factorise la logique réellement dupliquée. Mais ne sur-abstrais
  pas : deux bouts de code qui se ressemblent aujourd'hui par hasard ne
  doivent pas être fusionnés de force (évite le couplage prématuré).
- **SOLID**, en particulier :
  - *Single Responsibility* : chaque fonction / classe / module a une seule
    raison de changer.
  - *Open/Closed* et *Dependency Inversion* : dépends d'abstractions, pas
    de détails, quand ça réduit le couplage sans complexité inutile.
- **Composition plutôt qu'héritage** quand c'est possible.
- **Pas d'optimisation prématurée.** Écris d'abord clair et correct.
  N'optimise que sur preuve (mesure), pas sur intuition.

## Détecte les erreurs au plus tôt

Déplace chaque erreur le plus en amont possible dans la chaîne :
compilation > analyse statique (type-checker, linter) > test > production.
Une erreur impossible à compiler vaut mieux qu'une erreur au runtime, qui
vaut mieux qu'un bug en production.

- **Préfère une garantie à la compilation** quand le langage le permet
  (voir les références par langage pour les mécanismes concrets).
- **Rends les états illégaux non-représentables.** Modélise les données de
  sorte qu'un état invalide ne puisse pas exister, plutôt que de le
  détecter après coup.
- **Types forts plutôt que types primitifs.** Une distance n'est pas un
  `int` ; un identifiant utilisateur n'est pas une `string` quelconque.
  Combats la *primitive obsession*.
- **Garde-fou KISS** : ne sacrifie pas la lisibilité pour gagner une
  garantie statique. Une métaprogrammation illisible qui rebute le lecteur
  trahit l'objectif. La garantie doit rester compréhensible.

## Lisibilité avant tout

- Le code doit se lire presque comme de la prose. Quelqu'un qui le découvre
  doit comprendre l'intention sans deviner.
- **Noms explicites et intentionnels.** Le nom dit *quoi* et *pourquoi*,
  pas seulement *comment*. Évite les abréviations obscures.
- **Pas de nombres ou chaînes magiques.** Donne-leur un nom (constante).
- **Les commentaires expliquent le *pourquoi*, pas le *quoi*.** Si un
  commentaire ne fait que paraphraser le code, supprime-le et rends le code
  plus clair à la place.
- **Longueur des commentaires : une ligne, deux au maximum.** Un bloc plus
  long doit se justifier par un vrai piège (API traître, contrat de durée de
  vie ou de concurrence) et rester exceptionnel — de l'ordre d'un par
  fichier, pas un par entité. Ne calque jamais la densité d'un code
  environnant verbeux : cette règle prime. Jamais de référence au contexte
  de travail (plans, `.md` non commités, numéros d'itération) — ça va dans
  le message de commit ou le ticket. Relis tes commentaires en fin de tâche
  et coupe avant de livrer.
- **Limite l'imbrication.** Préfère les retours anticipés (guard clauses)
  aux pyramides de `if` imbriqués.

## Taille et structure

- **Évite les fonctions longues** (> ~100 lignes). Découpe en unités à
  responsabilité unique avec des noms parlants. Le seuil est un signal, pas
  une loi : une fonction de 60 lignes qui fait trois choses doit aussi être
  découpée.
- Limite le nombre de paramètres ; au-delà de 3-4, envisage un objet de
  configuration.
- **Repère l'état qui se propage.** Quand plusieurs fonctions se passent les
  mêmes paramètres de proche en proche (le même trio/quatuor threadé d'appel
  en appel), c'est le signe qu'elles partagent un état : regroupe-les dans un
  type qui porte cet état en membres. À l'inverse, n'introduis pas une classe
  quand des fonctions libres sans état partagé suffisent (KISS). Le défaut
  n'est pas le *nombre* de fonctions, c'est l'état threadé et l'absence
  d'interface lisible.
- Garde les fichiers et modules cohérents : une responsabilité claire par
  fichier.

## Robustesse et gestion des ressources

- **Lie la durée de vie des ressources à une portée.** Une ressource
  acquise (mémoire, fichier, verrou, socket) doit être libérée
  automatiquement et de façon sûre face aux erreurs, sans dépendre de la
  discipline du développeur. Les mécanismes diffèrent par langage (RAII en
  C++, `with` en Python, `trap` en Bash, `cleanup` ou `goto cleanup` en C) —
  voir les références.
- **Gestion d'erreurs explicite.** N'avale jamais une erreur en silence.
  Échoue tôt et clairement (fail fast) plutôt que de propager un état
  invalide.
- **Valide les entrées** aux frontières (entrées utilisateur, appels
  réseau, lecture de fichiers).
- Préfère l'immuabilité quand c'est raisonnable ; limite l'état mutable
  partagé.
- **Ne jette pas un résultat porteur de sens en silence** (valeur de retour
  d'erreur, statut). Le mécanisme dépend du langage.
- **Sécurité de base** : jamais de secrets en dur, pas de concaténation de
  requêtes/commandes avec des entrées non échappées, attention aux données
  sensibles dans les logs.

## Concurrence et parallélisme

- L'état mutable partagé est la racine des *data races*. Le meilleur verrou
  est celui qu'on n'a pas à poser : préfère l'**immuabilité** et le
  **passage de messages** (queues) au partage.
- Protège **toujours** une donnée partagée mutable (verrou ou type
  atomique), lectures comprises. Pas d'accès non synchronisé « parce que ce
  n'est qu'une lecture ».
- Nomme explicitement ce qui est partagé et **qui le possède**.
- Anti-deadlock : ordre d'acquisition des verrous cohérent et documenté,
  sections critiques courtes, **jamais d'appel bloquant ou coûteux sous
  verrou**.
- Préfère les abstractions de haut niveau (pools, executors, queues) aux
  threads bruts.
- Ne te fie jamais au timing ni à un ordre d'exécution supposé pour la
  correction.

## Tests

- Pour toute logique non triviale, propose ou écris des tests qui couvrent
  le cas nominal, les cas limites et les cas d'erreur.
- Les tests valident un *comportement*, pas une implémentation : ils ne
  doivent pas casser à chaque refactor interne.

## Quand tu enfreins une règle

- C'est parfois justifié. Mais ne le fais jamais en silence : explique en
  une phrase pourquoi cet écart est le bon choix ici.

## Avant de considérer la tâche terminée

- Relis le diff : as-tu modifié uniquement ce qui devait l'être (pas de
  changements orthogonaux non demandés) ?
- Le code passe-t-il les tests, le linter et l'analyse statique du projet ?
- Une personne extérieure comprendrait-elle ce code sans explication orale ?
