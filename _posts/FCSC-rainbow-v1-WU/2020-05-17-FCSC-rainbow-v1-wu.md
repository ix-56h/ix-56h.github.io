---
title: FCSC 2020 Write-Up - Rainbow v1 - GraphQL Introspection Exploit
date: 2020-05-17 10:32:00 +07:00
tags: [english, FCSC, CTF, writeup, web, security, GraphQL, introspection]
description: Exploiting GraphQL introspection to extract sensitive data from an insecure API endpoint in FCSC 2020's Rainbow v1 challenge.
---

## Challenge Overview

Rainbow v1 presented a dynamic search interface for finding chefs. Upon inspection, the site uses a JavaScript frontend that formats search queries and sends them to `index.php?search=`.

## Initial Reconnaissance

### Discovering GraphQL

The first step was analyzing the search parameter. I noticed it was Base64-encoded, which immediately suggested something interesting was being hidden. After decoding, i discovered an unusual JSON-like structure:

```graphql
{ 
  allCooks (filter: { firstname: {like: "%%"}}) { 
    nodes { 
      firstname, 
      lastname, 
      speciality, 
      price 
    }
  }
}
```

This wasn't standard JSON or SQL, it was **GraphQL**. Having not worked extensively with GraphQL at the time, i started researching common vulnerabilities and exploitation techniques for GraphQL APIs.

### Understanding the Vulnerability

The critical issue here is that the application allows users to construct and execute arbitrary GraphQL queries. There's no query validation or restriction on what fields can be accessed. This is a common misconfiguration in GraphQL implementations.

## Exploitation

### GraphQL Introspection

GraphQL has a powerful feature called **introspection** that allows clients to query the schema itself. This is incredibly useful for development and documentation, but when left enabled in production, it becomes a security liability.

I used the following introspection query to dump the entire schema:

```graphql
{
  __schema {
    types {
      name,
      fields {
        name
      }
    }
  }
}
```

**What this does:**
- `__schema`: The root introspection query
- `types`: Lists all types defined in the schema
- `fields`: Shows all fields available on each type

This revealed the complete database structure, including a particularly interesting field: `allFlags`.

### Extracting the Flag

With the schema exposed, i crafted a new query to retrieve all flags:

```graphql
{ 
  allFlags (filter: { flag: {like: "%%"}}) { 
    nodes { 
      flag
    }
  }
}
```

The wildcard filter `%%` ensures all records are returned. Executing this query immediately revealed the flag.

## Conclusion

This challenge wasn't particularly complex, the vulnerability was pretty straightforward. The application allowed users to format and execute arbitrary GraphQL queries, which is a fundamental security flaw. Combine that with introspection enabled in production, and you've essentially given attackers a complete map of your database schema.

## References

- [API Hacking: GraphQL](https://medium.com/@ghostlulzhacks/api-hacking-graphql-7b2866ba1cf2)
- [GraphQL Security Best Practices](https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html)
