# Règles spécifiques Bash

À lire avant d'écrire du Bash. Complète les principes universels du
SKILL.md.

## Mode strict d'abord (le « fail fast » du shell)

- Commence chaque script par :

  ```bash
  set -euo pipefail
  ```

  - `-e` : arrête au premier échec de commande.
  - `-u` : erreur sur variable non définie.
  - `-o pipefail` : un échec dans un pipe fait échouer le pipe entier.

- Ajuste `IFS` si tu traites des entrées séparées par des espaces/lignes.

### `-e` ne te protège pas autant qu'il en a l'air

C'est un bon défaut, pas une garantie. Connais ses exemptions et ses
pièges, sinon tu te crois couvert à tort :

- **Contexte testé = exempté (et c'est voulu).** Une commande dans la
  condition d'un `if`/`while`/`until`, après `!`, ou à gauche d'un
  `&&`/`||`, n'arrête pas le script si elle échoue : son code de retour est
  testé. `if grep -q motif f; then …` est donc sûr.
- **`-e` est désactivé dans une fonction appelée en contexte testé.**
  `if f; then …` : à l'intérieur de `f`, `-e` ne s'applique plus, et
  l'exécution continue après une commande échouée. Piège majeur.
- **`local`/`declare` masquent le code de retour** : `local x=$(cmd)` prend
  le statut de `local`, jamais celui de `cmd`. Sépare : `local x; x=$(cmd)`.
- **Arithmétique** : `((count++))` renvoie 1 quand le résultat est 0 → `-e`
  tue le script. Utilise `: $((count++))` ou `count=$((count+1))`.
- Pour une logique conditionnelle fine, **teste explicitement** les codes
  de retour plutôt que de te reposer sur `-e`.

## Quoting systématique

- **Mets toujours les variables entre guillemets** : `"$var"`,
  `"${array[@]}"`, `"$(cmd)"`. L'absence de quotes est la source n°1 de
  bugs en shell (séparation de mots, expansion de globs).
- Utilise `"${var:?message}"` pour exiger qu'une variable soit définie.
- Utilise `[[ … ]]` plutôt que `[ … ]` : pas de découpage de mots ni
  d'expansion de glob sur les variables, et opérateurs plus sûrs (`==`,
  `=~`, `&&`).

## Gestion des ressources : `trap` (l'équivalent RAII)

- Crée les fichiers temporaires avec `mktemp`.
- Nettoie-les via un `trap` sur la sortie, quel que soit le chemin pris :

  ```bash
  tmp="$(mktemp)"
  trap 'rm -f "$tmp"' EXIT
  ```

  Ça garantit le nettoyage même en cas d'erreur ou d'interruption.

## Détecter au plus tôt : analyse statique

- **Passe `shellcheck`** sur chaque script et traite les avertissements.
  C'est l'équivalent du compilateur/linter pour le shell.
- Vérifie les codes de retour ; ne chaîne pas des commandes en supposant
  leur succès.

## Style et structure

- Préfère des **fonctions** nommées et `local` pour leurs variables.
- Noms de variables en minuscules pour le local, MAJUSCULES réservées à
  l'environnement.
- Garde les scripts courts. **Au-delà d'une certaine complexité (boucles
  imbriquées, structures de données, parsing), réécris en Python** plutôt
  que d'empiler du Bash fragile.

## Sécurité

- Méfie-toi de l'injection : évite `eval` ; ne passe jamais une variable
  non quotée à une commande.
- Quote les chemins (les espaces dans les noms de fichiers cassent tout).
- Valide / assainis les entrées externes avant de les utiliser.

## Concurrence

- Concurrence = process en arrière-plan (`&` + `wait`), `xargs -P`, GNU
  `parallel`. Pas de mémoire partagée : le risque se déplace sur les
  **fichiers**.
- Écris de façon atomique (écriture dans un temporaire + `mv`), sérialise
  avec `flock`, crée les temporaires avec `mktemp`.
- **Le besoin de vraie synchronisation est un signal de réécrire en
  Python** (cf. la règle sur la complexité plus haut).
