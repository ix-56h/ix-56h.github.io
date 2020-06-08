---
title: EN - How to make an elegant tokenizer in C
date: 2020-06-01 12:45:00 +07:00
tags: [english, tokenizer, lexer, parser, shell, C]
description: Implementation of a tokenizer in a Shell Parser context in C.
---

## Definitions

In the past, it was really hard for me to understand what exactly lexer/parser/tokenizer is.  
The reason for that is the complexity (i guess ?) that seems for my comrades (o/).  
During the development of the last project of the first part of my 42 cursus, i was in front of (too) many definitions of what "Lexer/Parser" is.

Anyway, it is really simple to understand what it is.
##### Token

A "token" is like a simplified representation of an element in the lexical context.  
Example with the shell language :
```bash
ls && pwd
```

Here, we have 3 tokens of 2 types :  
`ls` is `TOK_WORD`  
`&&` is `TOK_AND`  
`pwd` also is `TOK_WORD`  

`TOK_*` is an enum.  
This is helpful to abstract our elements rather than browse many times the word we currently process.
##### Lexer

The lexer would be the "Rules Maker". (What a shitty name, sorry...)  
His role is to check the compatibility of the current context with the next one (at this step, we say "context" to the char level, not at the token level, that's the parser's role).

So, define these rules :  
A `WORD` token would be : `a-zA-Z[0-9]`  
A `OPERATOR` token would be : `&|><!;`  
(don't care about the Regexp, this is an example.)

Theses rules let us differentiate a token to another one.  
`ls` would be different to `>>`, ls is a `word`, `>>` would be a `operator` (most exactly a `redirection`).
##### Tokenizer

The tokenizer is the code that uses the lexer's rules and will give us the full context.
##### Parser

The parser will check the compatibility between the different contexts.  
This is basically a parser, so, if you want to learn more about it, check "Recursive Descent Parser" in references.
## It's time to tokenizing !

Let's see an example with the Shell grammar :  
```cpp
typedef enum		e_toktype {
	TOK_ERROR,
  ...
	TOK_WORD,
	TOK_AND_IF,
	TOK_OR_IF,
  ...
	TOK_MAX
}					t_toktype;
```
Here we have our tokens.

Let me ask you : How would you recognize words ? With a split and switch case ? Hm, no.  
Answer : CHR_CLASS.
#### Lexer + CHR_CLASS

We will define some enums which replaces the basic char recognization (`s[i] == '=';`) :  
```cpp
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

"This is just some enums, what you want to do with that ?"  
It's time to show you what i mean by "total abstraction" :  
```cpp
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

Here, we define all enums who's let us to abstract the charset. We can easily decide to set a `space` as an entire word. That's let us group the chars and their meaning.  
Ok, let see an example with `ls` :  
The word have 2 chars : `l` and `s`, these chars are abstracted as "CHR_WORD".

No switch case, no expensive lines, just a clean and fast-to-read code.  
That's really usefull in this case :
```cpp
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
So, what's next ? The lexer, cause' rules are nothing without Judge ! (wtf i'm sayin ?)
#### Lexer

The lexer defines rules on the current context.  
At this point, we call "context" the token. We need to determine the current token before process him.  

For that purpose, i use array :  
```cpp
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

This array let us to get the current context by the first char.  
```cpp
unsigned int token_type = g_get_tok_type[g_get_chr_class[string[i]]];
```

Good, we have the context, now we need to process the current token as long as we are in a valid context.  
It's time to code rules !
```cpp
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
So, we have rules, now we need to browse as long as we are in a valid context and save the token. Easy to go !
### Tokenizer in action !

While we get the state of the current token, ex : `TOK_WORD`, we have to browse the current string until the current context state is no longer valid.  
```cpp
unsigned int i = 1;
unsigned int token_type = g_get_tok_type[g_get_chr_class[string[0]]];
while (g_token_chr_rules[token_type][g_get_chr_class[string[i]])
{
  ...
  i++;  
}
// here you can now save the token with your favorite data structure.
```
## Conclusion

So, you have made a Lexer & Tokenizer.  
For the parser, you can do it by yourself, if you want to have a look on my own, you can see my [Recursive Descent Parser](https://github.com/ix-56h/Recursive-Descent-Parser).
### References

[Finite State Machine](https://en.wikipedia.org/wiki/Finite-state_machine)  
[Lexical_analysis](https://en.wikipedia.org/wiki/Lexical_analysis)  
[Recursive-Descent-Parser](https://github.com/ix-56h/Recursive-Descent-Parser)  
