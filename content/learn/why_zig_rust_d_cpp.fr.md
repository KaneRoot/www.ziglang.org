---
title: "Pourquoi Zig quand il existe déjà C++, D, et Rust ?"
mobile_menu_title: "Pourquoi Zig..."
toc: true
---


## Pas de flot de contrôle caché

Si un code en Zig ne semble pas faire appel à une fonction, c'est qu'il ne le fait pas.
Cela veut dire que vous pouvez être sûr que le code suivant ne fait appel qu'à `foo()` puis `bar()`, et c'est garanti sans connaître aucun des types impliqués :

```zig
var a = b + c.d;
foo();
bar();
```

Exemples de flots de contrôle cachés :

- D a des fonctions `@property`, des méthodes qui ressemblent à de simples accès à des champs d'une structure, `c.d` peut être un appel à une fonction.
- C++, D, et Rust ont de la surcharge d'opérateurs, donc l'opérateur `+` peut appeler une fonction.
- C++, D, et Go ont des exceptions *throw/catch*, donc `foo()` peut lancer une exception, et empêcher `bar()` d'être appelé.
(Bien sûr, même en Zig `foo()` pourrait être bloquant et empêcher `bar()` d'être appelée, mais cela pourrait survenir dans n'importe-quel langage Turing-complet.)

L'objectif de cette conception est d'améliorer la lisibilité.

## Aucune allocation cachée

Zig ne gère pas lui-même les allocations mémoire sur le *tas*.
Il n'y a pas de mot clé `new` ou autre fonctionnalité qui utiliserait un allocateur de mémoire (comme un opérateur de concaténation de chaînes de caractères par exemple[1]).
Le concept de *tas* est géré par une bibliothèque ou le code de l'application, pas par le langage.

Exemples d'allocations cachées :

* En Go, `defor` alloue de la mémoire dans la pile de la fonction.
En plus d'être un flot de contrôle contre-intuitif, cela peut provoquer des erreurs d'allocation de mémoire si vous utilisez `defer` dans une boucle.
* Les coroutines C++ allouent de la mémoire sur le tas pour appeler une coroutine.
* En Go, un appel de fonction peut engendrer une allocation sur le tas car les *goroutines* allouent des petites piles qui finissent par être redimensionnées lorsque la pile est trop sollicitée.
* Les API standards de Rust plantent sur des conditions de manque de mémoire, et les API alternatives qui acceptent un allocateur de mémoire sont des réflexions après coup (voir [rust-lang/rust#29802 (EN)](https://github.com/rust-lang/rust/issues/29802)).

Presque tous les langages avec ramasse-miettes (*garbage collector*) ont des flots de contrôle cachés éparpillés, puisque le ramasse-miettes cache la libération de la mémoire.

Le principal problème avec les allocations de mémoire cachées est qu'elles empêchent la réutilisabilité du code, limitant le nombre d'environnements dans lequel il pourrait être déployé.
En des termes simples, il existe des cas nécessitant de n'avoir aucune allocation mémoire, donc le langage de programmation doit fournir cette garantie.

En Zig, il y a des fonctionnalités de la bibliothèque standard qui fournissent et fonctionnent avec des allocateurs de mémoire, mais elles sont optionnelles, pas incluses dans le langage de programmation lui-même.
Si vous n'allouez pas de mémoire dans le tas, vous pouvez être sûr que votre programme ne le fera pas.

Chaque fonctionnalité de la bibliothèque standard qui nécessite d'allouer de la mémoire sur le tas prennent un paramètre `Allocator`.
Cela veut dire que la bibliothèque standard de Zig prend en charge les cibles `freestanding` (qui ne nécessitent pas de système d'exploitation).
Par exemple, `std.ArrayList` et `std.AutoHashMap` peuvent être utilisées pour de la programmation matérielle !

Les allocateurs de mémoire sur-mesure rendent simple la gestion manuelle de la mémoire.
Zig a un allocateur de mémoire de debug qui vérifie les erreurs comme les utilisations après libération (*use-after-free*) et les double libérations (*double free*).
Il détecte automatiquement et affiche la trace de la pile en cas de fuite de mémoire.
Zig possède également un allocateur *arêne* permettant de multiples allocations de mémoire puis de toutes les libérer en une seule fois.
D'autres allocateurs, plus spécifiques, peuvent améliorer les performances ou l'usage de la mémoire pour des besoins spécifiques.

[1]: En réalité, il y a un opérateur de concaténation de chaînes de caractères (un opérateur de concaténation de tableaux, pour être plus précis), mais il fonctionne à la compilation.
Il n'y a donc pas d'allocation sur le tas à l'exécution.

## Prise en charge de Zig sans bibliothèque standard

Comme indiqué plus tôt, la bibliothèque standard de Zig est entièrement optionnelle.
Chaque API n'est compilée dans le programme que si elle est utilisée.
Zig a la même prise en charge avec ou sans libc.
Zig encourage son utilisation directement sur le matériel et le développement à haute performance.

Ceci est le meilleur des deux mondes.
Par exemple en Zig, les programmes WebAssembly peuvent utiliser les fonctionnalités habituelles de la bibliothèque standard, et quand même avoir de petits exécutables comparés aux autres langages prennant en charge WebAssembly.

## Un langage pour des bibliothèques portables

Le graal en programmation est de pouvoir réutiliser le code.
Malheureusement, en pratique, nous réécrivons les mêmes fonctions, les mêmes bibliothèques… et cela est souvent justifié.

* Si l'application nécessite du temps-réel, alors toute bibliothèque utilisant un ramasse-miettes ou tout autre comportement non déterministe est disqualifiée.
* Si le langage rend trop facile pour le développeur d'ignorer les erreurs, et qu'il faut s'assurer que la bibliothèque gère correctement les erreurs, il peut être tentant de mettre de côté la bibliothèque et de la réécrire.
Zig est conçu pour que la gestion correcte des erreurs soit la chose la plus paresseuse à faire pour le développeur, et que donc nous soyons confiant dans la bibliothèque.
* Pour le moment il est en pratique vrai que le C est le langage le plus versatile et portable.
Tout langage n'ayant pas de possibilité d'interagir avec le C risque de disparaître.
Zig est une tentative de devenir le nouveau langage portable pour les bibliothèques en rendant à la fois facile de se conformer à l'ABI C pour les fonctions exportées, et introduisant de la sécurité et une conception de langage qui empêche les erreurs les plus communes dans les implémentations.
