---
title: How to make a elegant tokenizer in C
date: 2020-06-01 12:45:00 +07:00
tags: [tokenizer, lexer, parser, shell, C]
description: Implementation of a tokenizer in a Shell Parser context in C.
---

## Definition

Avant de s'attaquer au code, il est impératif de connaitre les différentes définitions liées aux Lexer, parser et tokenizer.

Durant le développement de mon projet de fin de cercle à 42Paris, j'ai été confronté aux mille et une définitions de ces termes.  
Certains définissent le lexer comme étant une couche avant le Parser, d'autres définissent le lexer comme étant les règles régissant le lexique analysé et certains pensent que ce n'est qu'un autre mot pour désigner le parsing.

Enfin bref, c'était un bordel sans nom.

Cependant, il est relativement simple de comprendre les différentes définitions :

##### Token

Un "token" représente un "mot", le champ lexical du language analysé est composé de différents mots.

Exemple pour le language SHELL :

```bash
ls && pwd
```

Ici, nous avons 3 tokens de 2 types différents :  
`ls` de type `TOK_WORD`  
`&&` de type `TOK_AND`  
`pwd` de type `TOK_WORD`  

Ici, `TOK_*` serait un enum, il est plus intuitif d'abstraire nos "mots" plutôt que d'analyser en continu le résultat d'un `split`.

##### Lexer

Le lexer, lui, correspond à la définition des règles lexicales.

Ainsi, imaginons les règles suivantes :  
Un token `MOT` peut être composé de : `a-zA-Z[0-9]`  
Un token `OPERATOR` peut être composé de : `&|><!;`  
Ne vous fiez pas à l'exactitude de la regexp, je liste juste les caractères acceptés dans la composition du mot.

Ces règles nous permettent donc d'identifier un `ls` d'un `>>`, ou encore un `echo hello word` d'un `||`.  

(ici, `echo hello world` est composé de 3 tokens "MOT", cependant, `hello` et `world` ne seront pas identiques à `echo`)

##### Tokenizer

Le tokenizer est la "couche" qui formera le token, il se fie aux règles du Lexer afin de découper les différents mots qui composent notre commande.

##### Parser

Le parser, quant à lui, s'occupe de vérifier la compatibilité des mots entre eux.  
Cela est très grossier, mais c'est la définition la plus simple. Ça reste un parser, merde !


## Définition des tokens

Nous parlons de shell depuis le début, donc on va utiliser les tokens de la grammaire SHELL !

{% highlight C %}
typedef enum		e_toktype {
	TOK_ERROR,
  ...
	TOK_WORD,
	TOK_AND_IF,
	TOK_OR_IF,
  ...
	TOK_MAX
}					t_toktype;
{% endhighlight %}

Bien, ici, nous avons nos tokens.

Désormais, laissez moi vous poser une question :  
Comment comptez-vous découper les mots d'une commande ? Un split ? Tss.

La réponse : CHR_CLASS.

#### Lexer + CHR_CLASS

Avant tout, définissons des enums qui définiront le type d'un caractère sans avoir à faire `s[i] == '=';` :  
```C
typedef enum		e_chr_class {
	CHR_ERROR,
	CHR_SP,
	CHR_DASH,
	CHR_BANG,
	CHR_AND,
	CHR_SEMI,
	CHR_WORD,
  ...
	CHR_MAX
}					t_chr_class;
```

Mais comme ça, ca ne sert pas à grand chose... Ce n'est juste qu'un tas d'enum.

Vous vous demandez surement à quoi cela va nous servir ?  
Il est temps de vous montrer ce que j'entends par "abstraction totale des caractères" :

```C
static t_chr_class		g_get_chr_class[255] =
{
	[' '] = CHR_SP,
	['\t'] = CHR_SP,
	[';'] = CHR_SEMI,
	['$'] = CHR_DOL,
	['#'] = CHR_WORD,
	['A'...'Z'] = CHR_WORD,
	['a'...'z'] = CHR_WORD,
	['0'...'9'] = CHR_DIGIT,
  ...
};
```

Je vais vous expliquer ce qu'il se passe :  
Ici, nous avons définis des enums qui nous permettront d'abstraire les caractères et ainsi les regroupper dans des "types" de caractères.  

Reprennons l'exemple de `ls` :  
Le mot comprend 2 caractères : `l` et `s`, ces deux caractères sont abstraits en `CHR_WORD`.

Cela revient à éviter de faire :  
```C
switch (s[i]) {
  case 'A':
    return (CHR_WORD);
  case 'B':
    return (CHR_WORD);
  ...
  default:
    return (CHR_ERROR);
}
```

A partir de ça, nous pouvons reconnaitre les caractères de notre commande, mais cela ne sert à rien sans Lexer.

#### Lexer

Le lexer va définir ce qui est autorisé dans le contexte actuel.  
Le "context" correspond au token qu'on est en train de construire, il nous faut donc déterminer le potentiel type du token courrant.

Pour cela, j'utilise un array :  
```C
static t_toktype		g_get_tok_type[CHR_MAX] = {
	[CHR_SP] = TOK_SP,
	[CHR_WORD] = TOK_WORD,
	[CHR_ESCAPE] = TOK_WORD,
	[CHR_DASH] = TOK_WORD,
	[CHR_DIGIT] = TOK_WORD,
	[CHR_BANG] = TOK_BANG,
  ...
};
```

Cet array me permettra de recuperer le potentiel contexte (le potentiel TOKEN si vous préférez) à partir du premier caractère de celui-ci :  
```C
unsigned int token_type = g_get_tok_type[g_get_chr_class[string[i]]];
```

Bien, désormais, nous avons le context, il nous faut maintenant continuer la tokenization jusqu'à rencontrer un cas qui mettra fin au contexte actuel.  
Il est temps d'implémenter les règles ! Pour cela, on utilise un array à 2 dimensions :  
```C
static int				g_token_chr_rules[TOK_MAX][CHR_MAX] =
{
	[TOK_SP] = {
		[CHR_SP] = 0
	},
	[TOK_WORD] = {
		[CHR_WORD] = 1,
		[CHR_DIGIT] = 1,
		[CHR_ESCAPE] = 1,
		[CHR_SQUOTE] = 1,
		[CHR_DQUOTE] = 1,
		[CHR_BQUOTE] = 1,
		[CHR_LPAREN] = 1,
		[CHR_RPAREN] = 0,
		[CHR_LBRACE] = 1,
		[CHR_RBRACE] = 0,
		[CHR_DOL] = 1,
		[CHR_DASH] = 1
	},
  ...
};
```

Ici, nous avons déclaré les règles des tokens.  
Comme vous pouvez le voir, cela permet d'abstraire la totalité de la grammaire, ici, nous pouvons choisir de redéfinir les caractéristiques d'un `WORD` ou d'un `OPERATOR`.  
Vous voulez définir le caractère `%` comme étant un espace ? Pas de problème.  
Vous voulez définir le caractère `|` comme étant le premier caractère d'un `TOK_WORD` ? Modifiez donc `g_get_tok_type` !

La question maintenant est de savoir comment parcourir tout cela... Rien de bien compliqué.

### Tokenizer in action !

While we get the state of the current token, ex : `TOK_WORD`, we need to browse the current string until the contexte state are changed.  
```c+
unsigned int i = 1;
unsigned int token_type = g_get_tok_type[g_get_chr_class[string[0]]];
while (g_token_chr_rules[tok_type][g_get_chr_class[s[i]])
{
  ...
  i++;  
}
// here you can now save the token with your favorite data structure.
```

So, you have made a Lexer & Tokenizer.  
For the parser, you can do it by yourself, if you want to have a look on my own, you can see my [Recursive Descent Parser](https://github.com/ix-56h/Recursive-Descent-Parser).

### References

[Finite State Machine](https://en.wikipedia.org/wiki/Finite-state_machine)  
[Lexical_analysis](https://en.wikipedia.org/wiki/Lexical_analysis)  
[Recursive-Descent-Parser](https://github.com/ix-56h/Recursive-Descent-Parser)  
