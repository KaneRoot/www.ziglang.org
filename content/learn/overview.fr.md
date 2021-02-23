---
title: "Dans le détail"
mobile_menu_title: "Détail"
toc: true
---
# Fonctionnalités
## Un langage simple

L'effort doit se faire sur la correction de votre application plutôt que sur la connaissance du langage.

La syntaxe de Zig est entièrement spécifiée dans [500 lignes de grammaire PEG](https://ziglang.org/documentation/master/#Grammar).

Rien n'est caché : ni **flots de contrôle**, ni allocations de mémoire.
Zig n'a pas de préprocesseur ni de macros.
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

Zig promeut la maintenance et la lisibilité du code en imposant tout flot de contrôle à être géré exclusivement avec des mots clés du langage et des appels de fonction.

## Performance ET sécurité

Zig a quatre [modes de compilation](https://ziglang.org/documentation/master/#Build-Mode), et ils peuvent être combinés jusqu'à une granularité aussi fine que le [bloc de code](https://ziglang.org/documentation/master/#setRuntimeSafety).

| Mode de compilation | [Debug](/documentation/master/#Debug) | [ReleaseSafe](/documentation/master/#ReleaseSafe) | [ReleaseFast](/documentation/master/#ReleaseFast) | [ReleaseSmall](/documentation/master/#ReleaseSmall) |
|-----------|-------|-------------|-------------|--------------|
Optimisations : + vitesse d'exécution, - debug, - durée de compilation | | -O3 | -O3| -Os |
Vérifications à l'exécution : - vitesse d'exécution, - taille, + plantage si comportement indéfini | On | On | | |

Voici ce à quoi ressemble un [dépassement d'entier](https://ziglang.org/documentation/master/#Integer-Overflow) à la compilation, peu importe le mode de compilation :

{{< zigdoctest "assets/zig-code/features/1-integer-overflow.zig" >}}

Voici ce à quoi ressemble l'exécution avec une compilation avec des vérifications de sécurité :

{{< zigdoctest "assets/zig-code/features/2-integer-overflow-runtime.zig" >}}

Ces [traces d'appels fonctionnent sur toutes les cibles](https://ziglang.org/#Stack-traces-on-all-targets), et également en [« freestanding » (binaire autonome, sans système d'exploitation)](https://andrewkelley.me/post/zig-stack-traces-kernel-panic-bare-bones-os.html).

Avec Zig il est possible de s'appuyer sur une compilation avec vérifications à l'exécution, tout en désactivant les vérifications seulement où les performances sont trop impactées.
Par exemple, l'exemple précédent pourrait être modifié comme cela :

{{< zigdoctest "assets/zig-code/features/3-undefined-behavior.zig" >}}

Zig détecte les [comportements indéfinis](https://ziglang.org/documentation/master/#Undefined-Behavior) à la compilation pour à la fois la prévention d'erreurs et l'amélioration des performances.

En parlant de performances, Zig est plus rapide que le C.

- L'implémentation de référence utilise LLVM comme un backend pour avoir l'état de l'art des optimisations.
- Ce que les autres projets appellent « Link Time Optimization », Zig l'a automatiquement.
- Pour les cibles natives, les fonctionnalités CPU avancées sont activées (-march=native) grâce à la [prise en charge de premier plan de la cross-compilation](https://ziglang.org/#Cross-compiling-is-a-first-class-use-case).
- Les comportements indéfinis sont soigneusement choisis.
Par exemple, en Zig les entiers signés et non signés ont un comportement indéfini lors d'un dépassement mémoire, contrairement à C où seul le dépassement d'entiers signés est indéfini.
Cela [facilite les optimisations qui ne sont pas disponibles en C](https://godbolt.org/z/n_nLEU).
- Zig expose directement le [type « vecteur SIMD ](https://ziglang.org/documentation/master/#Vectors), permettant d'écrire facilement du code vectorisé portable.

Important à noter, Zig n'est pas à considérer comme un langage *sécurisé*.
Pour ceux qui sont intéressés par suivre les histoires de sécurité dans Zig, abonnez-vous à ces fils de discussion :

- [enumerate all kinds of undefined behavior, even that which cannot be safety-checked](https://github.com/ziglang/zig/issues/1966)
- [make Debug and ReleaseSafe modes fully safe](https://github.com/ziglang/zig/issues/2301)

## Zig est en compétition avec le C, il n'en dépend pas

La bibliothèque standard de Zig intègre la libc, mais n'en dépend pas.
Voici un Hello World :

{{< zigdoctest "assets/zig-code/features/4-hello.zig" >}}

Quand ce code est compilé avec `-O ReleaseSmall`, les symboles de debug retirés, sur un seul fil d'exécution, cela produit un binaire static de 9.8 KiB pour la cible x86_64 :
```
$ zig build-exe hello.zig --release-small --strip --single-threaded
$ wc -c hello
9944 hello
$ ldd hello
  not a dynamic executable
```

Le binaire produit pour Windows est encore plus petit, seulement 4096 octets :
```
$ zig build-exe hello.zig --release-small --strip --single-threaded -target x86_64-windows
$ wc -c hello.exe
4096 hello.exe
$ file hello.exe
hello.exe: PE32+ executable (console) x86-64, for MS Windows
```

## Déclarations de premier niveau indépendantes de l'ordre

Les déclarations de premier niveau, comme les variables globales, sont indépendantes de l'ordre dans lequelle elles sont écrites et leur analyse est paresseuse.
Les valeurs d'initialisation des variables globales sont [évaluées à la compilation](https://ziglang.org/#Compile-time-reflection-and-compile-time-code-execution).

{{< zigdoctest "assets/zig-code/features/5-global-variables.zig" >}}
