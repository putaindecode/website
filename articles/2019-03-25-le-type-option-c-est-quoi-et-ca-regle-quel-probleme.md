---
date: 2019-03-25
title: Le type option, c'est quoi et ça règle quel problème ?
author: bloodyowl
slug: le-type-option-c-est-quoi-et-ca-regle-quel-probleme
---

La plupart des langages populaires aujourd'hui ont une valeur particulière appelée `null`. Elle représente l'absence délibérée de valeur. JavaScript possède aussi `undefined`, qui fonctionne à peu près de la même façon mais pour d'autres significations.

Un des problèmes souvent rencontré dans ces langages est que `null` est implicitement accepté comme valeur possible de n'importe quelle variable. Il est donc assez facile de se trouver avec un `null is not an object` ou une célèbre `NullPointerException` avec une stacktrace qui ne vous dira pas d'où est sorti ce `null`.

`null` est une valeur importante pour la conception de programme : on n'a pas toujours de valeur, et il faut être en mesure de l'exprimer dans notre code. Pourtant, la plupart des langage fonctionnels statiquement typés n'ont pas de concept de `null`.

Comment gèrent-ils ça ? Avec un **type option** (aussi appelé **type maybe** dans certains langages), qui est un **petit conteneur** qui wrap la valeur (ou la non-valeur).

## Le type

Le type option est un **variant**, qui peut de loin s'apparenter à un type d'union.

```reason
type option('value) =
  /* on définit les différentes valeurs possibles,
    une valeur du type option sera forcément d'une des deux listées ci-dessous */
  | None /* pas de valeur */
  | Some('value); /* une valeur du type `'value`*/
```

`'value` est ici ce qu'on appelle un **paramètre de type**, ça permet au type d'être «generique»: il se fout du type de la valeur contenue, et vous laisse le spécifier à l'usage ou laisse l'inférence de type le deviner.

```reason
let isMyself = fun
  | Some("Matthias") => true
  | Some(_) | None => false;
```

Ici, la fonction aura la signature suivante :

```reason
let isMyself: option(string) => bool;
                  /* ^ le compiler a compris qu'il s'agissait d'une chaîne de caractères! */
```

Cette généricité fait de l'option une abstraction générale pour représenter la présence ou l'absence de n'importe quel type de valeur. Cela nous permet par exemple de créer une fonction `map` :

```reason
let map = (opt, f) =>
  switch (opt) {
  | Some(x) => Some(f(x))
  | None => None
  };
```

Et cette fonction pourra être utilisée pour **n'importe quelle option**. Jetons un œil à sa signature :

```reason
let map: (option('a), 'a => 'b) => option('b);
```

On peut lire cette signature de cette façon :

- on a une fonction map
- elle prend une option contenant une valeur de type `a`
- elle prend une fonction prenant une valeur de type `a` et retourne une valeur de type `b`
- elle retourne une option contenant une valeur de type `b`

```reason
Some(2)->map(x => x * 3) // Some(6)
None->map(x => x * 3) // None
```

Un autre exemple de fonction utile est `flatMap`:

```reason
let flatMap = (opt, f) =>
  switch (opt) {
  | Some(x) => f(x)
  | None => None
  };
/* let flatMap: (option('a), 'a => option('b)) => option('b); */
/* `get` retourne une option */
get("profile")
->flatMap(a => a->get("address"))
->flatMap(b => b->get("zipCode"))
```

## Le problème résolu par le type option

Prenons pour exemple la fonction `Array.prototype.find` de JavaScript :

```js
let item = array.find(item => item === undefined || item.active);
```

Cet code va retourner:

- un objet s'il a un champ `active` ayant une valeur évaluée comme vraie
- `undefined` si un item de `array` est `undefined`
- `undefined` si rien n'est trouvé

Avec cette implémentation naïve, nous sommes incapables de savoir dans quel cas nous nous trouvons : soit on a trouvé un item `undefined`, soit on a rien trouvé.

_Notez que le problème se pose ici avec `undefined` mais qu'il en serait de même un tableau contenant des `null` et une fonction `find` d'une bibliothèque retournant `null` dans le cas où elle ne trouve rien_

Si l'on veut être capable de faire la différence entre les deux derniers cas, on doit utiliser une autre fonction: `findIndex`:

```js
let index = array.findIndex(item => item === undefined || item.active);
if (index == -1) {
  // not found
} else {
  // found
  let item = array[index];
}
```

Le code est plus lourd, moins lisible, et manque d'expressivité. `find` ne nous donne pas assez d'information au travers de la valeur retournée: `undefined` est "applati", et requiert une logique supplémentaire (ici `index`, si un item est trouvé, il sera supérieur à `-1`, sinon il sera égal à `-1`)

Le problème ne vient pas de la fonction `find` elle même mais de la façon dont `null` et `undefined` sont traités. `null` **est** la valeur, il la **remplace**. `option` **l'enrobe**: c'est un conteneur.

```reason
/* `getBy` est l'equivalent de `find` */
let result = array->Belt.Array.getBy(
  fun
    | None => true
    | Some({active}) => active
);
```

D'abord, `array` a le type suivant:

```reason
let array: array(option(value));
```

Et `getBy` celui-ci:

```reason
let getBy: (array('a), 'a => bool) => option('a);
```

Si on remplace les paramètres de type par le type vraiment utilisé dans notre cas précis, on se retrouve avec ça :

```reason
let getBy:
  (
    array(option(value)),
    option(value) => bool
  ) => option(option(value));
```

`result` aura donc le type suivant :

```reason
let item: option(option(value));
```

C'est une `option` d'`option` de `value`. Et ça signifie qu' **on peut extraire l'information qui nous intéresse** de la valeur de retour:

- si le résultat est `Some(Some(value))` : on a trouvé une valeur `true` pour le champ `active`
- si le résultat est `Some(None)` : on a trouvé une valeur `None`
- si le résultat est `None` : on n'a rien trouvé dans le tableau

Le type `option` a éliminé par _design_ certains problèmes inhérents à `null` et `undefined` en se comportant comme un conteneur plutôt qu'un substitut.

La nature des `option` dans les langages statiquement typés permet d'éviter de nombreuses erreurs de conception. Les fonctions ne sont plus juste autorisées à retourner `null` et à vous laisser la responsabilité implicite de le gérer à grand coup de `if(value == null) { a } else { b }`, elles retournent un type `option` qui vous **force à prendre en compte l'optionalité de la valeur**.

Bisous.