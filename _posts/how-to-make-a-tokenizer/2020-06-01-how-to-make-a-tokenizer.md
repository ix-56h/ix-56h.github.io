---
title: EN - How to make an elegant tokenizer in C
date: 2020-06-01 12:45:00 +07:00
tags: [english, tokenizer, lexer, parser, shell, C]
description: Implementation of a tokenizer in a Shell Parser context in C.
---

## Definitions

Understanding the concepts of lexer, parser, and tokenizer can be challenging when first encountered. During my work on a shell parser project at 42, I encountered many different definitions of these terms that initially seemed overwhelming.

However, these concepts are simpler than they first appear. Let me break them down clearly.
##### Token

A "token" is a simplified representation of an element in the lexical context.  
Example with the shell language:
```bash
ls && pwd
```

Here, we have 3 tokens of 2 types:  
`ls` is `TOK_WORD`  
`&&` is `TOK_AND`  
`pwd` is also `TOK_WORD`  

`TOK_*` is an enum.  
This abstraction helps us avoid repeatedly analyzing the same word during processing.
##### Lexer

The lexer defines the rules for tokenization. Its role is to check the compatibility of the current context with the next one. At this step, we work at the character level, not at the token level (which is the parser's role).

For example, we might define these rules:  
A `WORD` token would be: `a-zA-Z[0-9]`  
An `OPERATOR` token would be: `&|><!;`  
(These are simplified examples for illustration.)

In simpler terms:
- A `WORD` can contain any lowercase letter (a-z), uppercase letter (A-Z), or digit (0-9)
- An `OPERATOR` can be any of these characters: `&`, `|`, `>`, `<`, `!`, `;`

These rules allow us to differentiate one token from another.  
For instance, `ls` is a `word`, while `>>` is an `operator` (more specifically, a `redirection`).
##### Tokenizer

The tokenizer is the code that uses the lexer's rules and will give us the full context.
##### Parser

The parser checks the compatibility between different contexts (at the token level).  
For more details on parser implementation, see "Recursive Descent Parser" in the references section.
## Implementation

Let's examine an example using shell grammar:  
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
Here we have our token types.

How do we recognize words efficiently? Instead of using string splitting and switch cases, we use a more elegant approach: CHR_CLASS.
#### Lexer + CHR_CLASS

We define enums to replace basic character recognition (like `s[i] == '='`):  
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

Now let's see how we achieve complete abstraction using these enums:  
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

Here, we define enums that allow us to abstract the character set. We can easily decide to treat a `space` as part of a word, or group characters by their meaning.  
For example, with `ls`:  
The word has 2 characters: `l` and `s`, both abstracted as `CHR_WORD`.

This eliminates verbose switch statements, resulting in clean and readable code.  
This is particularly useful compared to the traditional approach:
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
Now let's implement the lexer rules that will use these character classes.
#### Lexer

The lexer defines rules for the current context.  
At this point, "context" refers to the token type. We need to determine the current token before processing it.  

We use an array for this purpose:  
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

This array allows us to determine the token type from the first character:  
```cpp
unsigned int token_type = g_get_tok_type[g_get_chr_class[string[i]]];
```

Once we have the token type, we need to process the token as long as we remain in a valid context.  
Here are the rules that define valid character sequences for each token type:
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
With these rules defined, we can now iterate through the string while the context remains valid, then save the token.
### Tokenizer in Action

Once we have the token type (e.g., `TOK_WORD`), we iterate through the string until the context is no longer valid:  
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

You now have a working lexer and tokenizer implementation.  
For parser implementation, you can explore it on your own or check out my [Recursive Descent Parser](https://github.com/ix-56h/Recursive-Descent-Parser) for a complete example.
### References

[Finite State Machine](https://en.wikipedia.org/wiki/Finite-state_machine)  
[Lexical_analysis](https://en.wikipedia.org/wiki/Lexical_analysis)  
[Recursive-Descent-Parser](https://github.com/ix-56h/Recursive-Descent-Parser)  
