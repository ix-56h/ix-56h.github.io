---
title: FR - FCSC 2020 - Write Up - Rainbow v1
date: 2020-05-17 10:32:00 +07:00
tags: [francais, FCSC, CTF, WU, Web, php, GraphQL]
description: Write Up du challenge Rainbow v1 du FCSC 2020.
---

## Rainbow v1
Le site se présente comme un formulaire dynamique de recherche de chefs.  
On y découvre rapidement un script JS qui formate la requête puis l'envoi à `index.php?search=`.

##### Premiers pas
On remarque rapidement que le paramètre est en Base64, en décodant, on découvre un drôle de json...  
```js
{ allCooks (filter: { firstname: {like: "%%"}}) { nodes { firstname, lastname, speciality, price }}}
```
Je ne connaissais pas ce format, en cherchant les erreurs que j'ai fait proc sur google, je découvre `GraphQL` (oui je sais, je n'ai pas fait de web depuis assez longtemps).  
Ni une ni deux, je suis parti à la recherche d'exploit de GraphSQL.

## Exploit basique
Je trouve ceci :  
```js
{__schema{types{name,fields{name}}}}
```  
Une fois mon payload exécuté, je découvre le schéma complet de la DB, j'y découvre le champ `allFlags`...  
Je refais une requête pour lister tout ce qui se trouve dans `allFlags`.  
```js
{ allFlags (filter: { flag: {like: "%%"}}) { nodes { flag}}}
```

## Conclusion

Il n'y avait rien de très compliqué dans ce challenge, la faille étant assez basique (l'utilisateur ayant la possibilité de formater la requête à sa guise).
### References
[api-hacking-graphql](https://medium.com/@ghostlulzhacks/api-hacking-graphql-7b2866ba1cf2)
