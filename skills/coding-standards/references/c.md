# Règles spécifiques C pur

À lire avant d'écrire du C. Complète les principes universels du SKILL.md.

## Gestion des ressources : pas de RAII natif

Le C standard n'a pas de destructeurs. La gestion sûre des ressources
repose donc sur une discipline stricte :

- **Pattern `goto cleanup` / sortie unique.** Sur les fonctions qui
  acquièrent plusieurs ressources, libère-les dans une section de nettoyage
  atteinte par `goto` sur chaque chemin d'erreur, dans l'ordre inverse de
  l'acquisition. C'est l'idiome reconnu (et celui du noyau Linux), pas un
  anti-pattern. C'est aussi le **repli portable** quand l'extension
  `cleanup` ci-dessous n'est pas disponible.
- **Une fonction qui alloue documente qui doit libérer.** Conventions de
  nommage cohérentes par paires : `x_create` / `x_destroy`, `x_init` /
  `x_free`.
- **Remets à `NULL` après `free` les pointeurs qui survivent à leur
  libération** (membres de structure, pointeurs réutilisés) pour éviter
  use-after-free et double-free. Inutile pour un pointeur local en fin de
  portée ; en débogage, laisser le pointeur tel quel aide l'ASan à
  signaler les double-free.

### `__attribute__((cleanup(...)))` — l'analogue RAII (GCC/Clang)

Quand tu compiles avec GCC ou Clang, applique-le dès que possible : c'est le
mécanisme C qui s'approche le plus de RAII. Appliqué à une variable locale,
il fait appeler la fonction de nettoyage avec un pointeur vers cette
variable dès qu'elle sort de portée — sur **tous** les chemins (return,
break, goto, fin de bloc). La libération cesse de dépendre de la discipline
du développeur.

```c
static inline void freep(void *p) { free(*(void **)p); }
#define _cleanup_free_ __attribute__((cleanup(freep)))

void f(void) {
    _cleanup_free_ char *buf = malloc(1024);
    if (!buf) return;        // buf libéré automatiquement ici
    /* ... usage ... */
}                             // et ici sur le chemin nominal
```

- Écris un nettoyeur par type de ressource (`_cleanup_free_`,
  `_cleanup_fclose_`, `_cleanup_close_`…), sur le modèle de systemd.
- Le nettoyeur doit être **idempotent et sûr sur valeur nulle** (ex. tester
  le descripteur, accepter `free(NULL)`), car il s'exécute même si la
  ressource n'a jamais été acquise.
- **Attention au transfert de propriété** : si la ressource doit survivre à
  la portée (valeur de retour), neutralise le nettoyage (`TAKE_PTR` :
  remettre le pointeur à `NULL` après l'avoir transmis).
- C'est une **extension non standard** : si la portabilité vers d'autres
  compilateurs est requise, reviens au `goto cleanup`.

### `constructor` / `destructor` — autre sujet, pas du RAII

`__attribute__((constructor))` et `__attribute__((destructor))` exécutent
une fonction **une seule fois** au chargement/déchargement du programme ou
de la bibliothèque partagée (avant `main()` / après sa sortie). C'est de
l'init/teardown **global** de module, utile pour ça, mais sans rapport avec
la gestion de ressources liée à une portée. Ne les confonds pas avec
`cleanup`.

## Vérifie tout

- Vérifie **chaque** valeur de retour pouvant échouer : `malloc`, `realloc`,
  `fopen`, `read`/`write`, appels système. Gère l'échec, ne suppose jamais
  le succès.
- Initialise les variables à la déclaration ; ne lis jamais une variable
  non initialisée.

## Détecter au plus tôt

- Compile avec `-Wall -Wextra -Werror` (et `-Wpedantic` si possible).
- **`_Static_assert` (C11)** pour les invariants vérifiables à la
  compilation.
- Pour les constantes entières, utilise `enum` (pas `#define`, pas `const`) :
  seul `enum` produit une constante de compilation utilisable comme taille de
  tableau, label de `case` ou dans `_Static_assert`. Réserve `const` aux
  valeurs non entières et aux objets en lecture seule.
- Utilise les types à largeur explicite de `<stdint.h>` (`uint32_t`, etc.)
  quand la taille compte.

## Sécurité

- Bannis les fonctions sans borne : `strcpy`, `strcat`, `sprintf`, `gets`.
  Préfère `snprintf` et des copies bornées avec terminaison gérée
  explicitement.
- Attention aux **débordements d'entiers** (notamment dans les calculs de
  taille d'allocation : `n * size`).
- Valide les indices et les tailles avant tout accès mémoire.
- N'utilise pas de tableau à taille variable (`int buf[n]` avec `n`
  dynamique) : taille de pile non bornée et non vérifiable. Alloue sur le tas
  avec une taille validée, ou utilise une borne fixe.

## Organisation des fichiers : en-têtes minces

Comme en C++ (mais sans la question des templates) : un `.h` expose une
interface, pas une implémentation. Garder le corps hors du `.h` réduit le
couplage et les recompilations en cascade.

- **`.h`** — déclarations seulement (prototypes, types, `extern`). **À
  l'exception** des fonctions `static inline`, qui doivent être visibles
  dans chaque unité de traduction qui les utilise et vivent donc dans le
  `.h` (idiome du noyau Linux), même non triviales.
- **`.c`** — le corps de tout le reste.
- **Vue d'ensemble en tête du `.c`.** Pour les fonctions `static` internes à
  un `.c`, place un bloc de prototypes `static` en tête du fichier (table des
  matières), corps définis plus bas — plutôt qu'une longue suite de
  définitions interdépendantes sans déclaration préalable. Même bénéfice qu'un
  `.h`, sans exposer ces internes.
- **Struct opaque pour cacher les détails.** Quand un type ne doit pas exposer
  ses champs, déclare `typedef struct Foo Foo;` dans le `.h` et définis
  `struct Foo { ... }` dans le `.c` : les consommateurs manipulent un pointeur
  opaque, l'implémentation reste privée (équivalent C du PIMPL).

## Style

- Limite la portée des macros ; préfère les fonctions `static inline`.
- Garde les fonctions courtes et à responsabilité unique (cf. SKILL.md).
- Limite la portée des variables au plus près de leur usage.
- **État partagé → struct contexte.** Concrétisation C du principe du
  SKILL.md (« repère l'état qui se propage ») : quand plusieurs fonctions se
  threadent les mêmes paramètres, regroupe-les dans une `struct` passée par
  pointeur (`struct ctx *`) plutôt que de multiplier les arguments.

## Concurrence

- `_Atomic` / `<stdatomic.h>` pour les accès partagés simples ; verrous
  `pthread_mutex_t` (ou C11 `mtx_t`, au support inégal — à vérifier selon la
  cible).
- **`volatile` n'est PAS de la synchronisation** : il sert au MMIO ; pour
  les signaux, c'est `sig_atomic_t`.
- Libère le verrou sur **tous** les chemins de sortie — relie ça au pattern
  `cleanup` / `goto cleanup` plus haut.
