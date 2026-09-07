# Règles spécifiques C++

À lire avant d'écrire du C++. Complète les principes universels du SKILL.md.

## Gestion des ressources : RAII

- **Toute ressource est possédée par un objet dont la durée de vie la
  gère.** L'acquisition se fait à la construction, la libération au
  destructeur — automatique et sûre face aux exceptions.
- **Pas de `new`/`delete` manuels** dans le code applicatif. Utilise les
  smart pointers : `std::unique_ptr` par défaut, `std::shared_ptr`
  seulement quand la propriété est réellement partagée.
- **Pas de pointeur nu propriétaire.** Un pointeur nu peut observer, jamais
  posséder.
- **Verrous** : `std::scoped_lock` / `std::lock_guard`, jamais
  `lock()`/`unlock()` manuels.
- Pour les ressources externes (handles, fichiers C, descripteurs),
  enveloppe-les dans un petit type RAII dédié.

## Rule of Zero (et à défaut Rule of Five)

- **Préfère la Rule of Zero** : conçois tes types pour qu'ils n'aient
  *aucun* destructeur / constructeur de copie / d'affectation / de move
  écrit à la main — laisse les membres RAII s'en charger.
- Si tu dois en déclarer un, déclare les cinq de façon cohérente (Rule of
  Five). N'en oublie aucun.

## Détecter au plus tôt : compile-time plutôt que run-time

- **`static_assert`** pour les invariants vérifiables à la compilation.
- **Concepts (C++20)** plutôt que du SFINAE obscur pour contraindre les
  templates.
- **`constexpr` / `consteval`** pour déplacer le calcul à la compilation
  quand c'est naturel.
- **Types forts** : `enum class` plutôt que `enum` nu ; types distincts
  pour les unités/identifiants plutôt que des `int`/`std::string`
  interchangeables.
- **Rendre les états illégaux non-représentables** : `std::optional`,
  `std::variant`, types somme, plutôt que des sentinelles ou des flags
  booléens corrélés.
- **Garde-fou KISS** : tout ça oui, mais pas de métaprogrammation de
  templates illisible « pour le sport ». La garantie statique doit rester
  lisible.

## const-correctness et immuabilité

- `const` partout où c'est possible : paramètres `const&`, méthodes `const`,
  variables locales `const`.
- `constexpr` quand la valeur est connue à la compilation.
- Préfère les objets immuables ; limite l'état mutable partagé.

## Ne pas ignorer les résultats

- **`[[nodiscard]]`** sur les fonctions dont ignorer le retour est un bug
  (codes d'erreur, valeurs calculées coûteuses, objets à durée de vie).

## Sécurité mémoire et bonnes pratiques

- Évite l'*undefined behavior* : pas d'arithmétique de pointeurs nus, pas
  d'accès hors bornes.
- Préfère les conteneurs et vues standard (`std::vector`, `std::array`,
  `std::span`, `std::string_view`) aux tableaux C — mais attention aux
  durées de vie des vues (elles ne possèdent rien).
- `static_cast`/`reinterpret_cast`/`const_cast` explicites plutôt que les
  casts à la C.
- Préfère `constexpr`/`inline`/templates aux macros. Réserve les macros aux
  cas où il n'y a pas d'alternative.

## Gestion d'erreurs

- Sois cohérent avec la convention du projet : exceptions, ou
  `std::expected` (C++23) / `std::optional` pour les erreurs attendues.
- Ne mélange pas les deux styles de façon anarchique dans un même module.

## Organisation des fichiers : en-têtes minces

Un `.h` expose une interface ; il ne devrait pas imposer une implémentation
à tous ceux qui l'incluent. Garder le corps hors du `.h` réduit le couplage,
accélère la compilation et évite les recompilations en cascade au moindre
changement d'implémentation. Mais un template ou une fonction `inline` doit
rester **visible au point d'instanciation** : on ne peut pas le cacher dans
un `.cpp`. D'où la répartition :

- **`.h`** — déclarations seulement. Pas de corps de fonction/méthode, **à
  l'exception** d'un *one-liner* inline ou template trivial, qu'on tolère
  sur place tant qu'il tient en une ligne courte.
- **`.hxx`** — le corps des fonctions/méthodes **inline et templates**
  (celles qui doivent être visibles à l'instanciation). Le `.h` l'inclut,
  typiquement en fin de fichier. (Convention de nommage du projet ;
  équivalente aux `.tpp`/`.ipp`/`.inl` rencontrées ailleurs.)
- **`.cpp`** — le corps de tout le reste : fonctions/méthodes non-inline et
  non-templates.
- Ne mets jamais de `using namespace` au niveau global d'un `.h`/`.hxx` : il
  s'impose à toutes les unités qui incluent l'en-tête. Garde-le local à une
  fonction ou à un `.cpp`.
- **Donne une vue d'ensemble, même sans en-tête.** Quand un type ou un groupe
  de fonctions vit entièrement dans un `.cpp` (détail d'implémentation, non
  exposé), place quand même leurs déclarations en tête du fichier, corps
  définis plus bas. Cette « table des matières » rend le fichier lisible sans
  imposer l'implémentation dans un `.h`. Un `.cpp` où de longues définitions
  interdépendantes s'enchaînent sans déclaration préalable oblige le lecteur à
  tout parcourir pour saisir la structure. Ne remonte pas ces déclarations
  internes dans le `.h` juste pour les « exposer » : ça y ferait fuiter des
  détails (includes, méthodes privées) inutiles aux consommateurs.

## Concurrence

- Protège les accès partagés avec `std::mutex` + RAII (`scoped_lock`,
  `lock_guard`) ; `std::atomic` pour les compteurs/drapeaux simples.
- **`volatile` n'est PAS un outil de synchronisation** — c'est pour le MMIO,
  pas les threads.
- `std::jthread` (C++20) plutôt que `std::thread` : join automatique et
  `stop_token`.
- `std::condition_variable` **toujours avec un prédicat** (réveils
  intempestifs).
- Une *data race* est un **comportement indéfini**. Attention aux durées de
  vie capturées par référence dans un thread.
