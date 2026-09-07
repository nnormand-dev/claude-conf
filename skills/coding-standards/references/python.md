# Règles spécifiques Python

À lire avant d'écrire du Python. Complète les principes universels du
SKILL.md.

## Gestion des ressources : les gestionnaires de contexte (`with`)

C'est l'équivalent Python du RAII.

- **Toute ressource (fichier, verrou, connexion, socket) s'utilise dans un
  `with`**, jamais ouverte/fermée à la main.
- Pour tes propres ressources, écris un gestionnaire de contexte
  (`@contextlib.contextmanager` ou `__enter__`/`__exit__`) plutôt que de
  laisser l'appelant gérer le nettoyage.

## Détecter au plus tôt : analyse statique

L'équivalent Python du « compile-time » est la vérification statique.

- **Annotations de type partout** (fonctions, attributs, variables), et un
  type-checker en mode strict (`mypy --strict` ou `pyright`) dans la CI.
  Ça déplace une classe entière de bugs du runtime vers l'analyse statique.
- **Types forts plutôt que primitifs** : `enum.Enum` pour les ensembles
  finis, `typing.NewType` pour distinguer des identifiants, `Literal` pour
  contraindre les valeurs.
- **Rends les états illégaux non-représentables** : `@dataclass(frozen=True)`,
  unions discriminées, plutôt que des dictionnaires fourre-tout ou des
  drapeaux corrélés.

## Immuabilité

- Préfère `@dataclass(frozen=True)`, les tuples et `frozenset` quand l'état
  ne doit pas changer.
- **Jamais d'argument par défaut mutable** (`def f(x=[])`) — utilise `None`
  et crée la valeur dans le corps.

## Robustesse

- **Fail fast** : valide les entrées tôt, lève des exceptions spécifiques.
- **Jamais de `except:` nu** ni de `except Exception` qui avale tout sans
  raison. Attrape l'exception la plus précise possible.
- N'utilise pas l'état global mutable.

## Style idiomatique

- Reste pythonique : compréhensions *lisibles*, `pathlib` plutôt que
  manipulation de chaînes de chemins, f-strings.
- Respecte PEP 8 ; laisse un formateur (`black` ou `ruff format`) et un
  linter (`ruff`) trancher le style — ne le fais pas à la main.
- Évite les imports circulaires ; garde des modules à responsabilité claire.
- Pour les logs, utilise le module `logging` plutôt que `print`, avec
  formatage paresseux (`logging.info("x=%s", x)`) plutôt qu'une f-string :
  l'argument n'est formaté que si le niveau de log est actif.

## Concurrence

- **Le GIL ne donne pas de parallélisme CPU** en CPython : threads pour
  l'I/O-bound, `multiprocessing` / `concurrent.futures` pour le CPU-bound.
- Le GIL ne **dispense pas** des verrous : une opération composée n'est pas
  atomique. Utilise `threading.Lock`, `queue.Queue`.
- Préfère `concurrent.futures` (`ThreadPoolExecutor` / `ProcessPoolExecutor`)
  à la gestion manuelle des threads.
- asyncio : mono-thread, **ne bloque jamais la boucle** (pas d'appel
  synchrone bloquant ; utilise les variantes `async`).
- Ne te repose pas sur le GIL pour l'atomicité : le CPython *free-threaded*
  (sans GIL) est en cours de déploiement. Vérifie le statut sur ta version
  Python cible.
