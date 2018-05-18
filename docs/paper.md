---
title: SPARQL Query Transformation for Graph-based Access Control
author:
- Nathaniel Rudavsky-Brody
- Aad Versteden
abstract: In a microservice-based architecture, reuse can be complicated by the fact that different applications might use different authorization schemes. In the context of Linked Data applications, we propose a technique for abstracting access rules via SPARQL query transformation, allowing access rights to be modeled in the database and individual microservices to remain authorization-agnostic. By consolidating authorization logic almost at the database level, we are able to simplify each microservice's SPARQL queries, and get authorization-aware caching almost for free.
---

# Introduction

`mu.semte.ch` is a platform for "building state-of-the-art web applications fueled by Linked-Data aware microservices."[^mu] It has been successfully used to build complex, production-ready applications in the fields of data management,...

[^mu]: See Versteden and Pauwels, "State-of-the-art Web Applications using Microservices and Linked Data" in [ref].

One of the advantages of such a platform is the ability to reuse microservices between applications. However, reuse is complicated by the fact that different applications can use very different authorization models and graph structures. The problem becomes acute when we different users are given access to different subsets of the data, and these access rights need to be computed dynamically.

Here we propose a general solution to this problem, based on dynamic query transformation. The Mu Query Rewriter is a proxy service for enriching and constraining SPARQL queries before they are sent to the database. It allows for the abstract expression of complex authorization schemes, greatly simplifying what each microservice needs to know about graph structure and access rules.

Furthermore, it provides a system of query annotations to facilitate integration with `mu.semte.ch`'s cache service, allowing the implementation of authorization-aware caching and cache-clearing.

# Basic Architecture

The Mu Query Rewriter runs as a proxy service in front of the database, handling SPARQL queries sent by the application's microservices. These microservices are expected to pass on the HTTP `mu-session-id` header provided by the Mu Identifier[^muid] which, in conjunction with a log-in service, identifies the URI of the current user.

[^muid]: https://github.com/mu-semtech/mu-identifier

Conceptually speaking, the access rules are expressed as a dynamic *constraint* expressed as a standard SPARQL `CONSTRUCT` query. The constraint is thought of as constructing an intermediate *constraint graph* on which incoming SPARQL queries are run. Since the constraint can depend on `mu-session-id` header, the constraint graph will be different for each user, and represents a hypothetical personal graph which that user is authorized to read and/or update (see Figure 1).

In practice, an incoming query is optimally rewritten to a form which, when run against the full database, is equivalent to running the original query against the hypothetical constraint graph, i.e., the subset of data which the current user has permission to query or update. 

![Basic mu.semte.ch architecture with the Query Rewriter and conceptual constraint graph](../rewriter.png)

As a simple example, here is a constraint describing a database where triples whose subject have type `<Car>` are stored in the `<cars>` graph, and similarly `<Bike>`s in the `<bikes>` graph. The `<auth>` graph contains triples specifying which users are authorized to see which types. The placeholder `<SESSION>` is dynamically replaced with the `mu-session-id` header. (*functional property*s, and *unique variable*s are defined below). To save space, `PREFIX mu: <http://mu.semte.ch/vocabularies/core/>` declaration is omitted from the constraint and rewritten query.

\small

+-------------------------------------+---------------------------+--------------------------------------+
| Constraint                          | Query                     | Rewritten Query                      |
+=====================================+===========================+======================================+
| ```                                 | ```                       | ```                                  |
| CONSTRUCT {                         | SELECT *                  | SELECT ?s ?color                     |
|   ?a ?b ?c                          | WHERE {                   | WHERE {                              |
| }                                   |   ?s a <Bike>;            |   GRAPH ?graph23694 {                |
| WHERE {                             |      <hasColor> ?color.   |       ?s a <Bike>;                   |  
|   GRAPH ?graph {                    | }                         |          <hasColor> ?color.          |
|    ?a ?b ?c;                        | ```                       |    }                                 |
|       a ?type                       |                           |   GRAPH <auth> {                     |
|   }                                 |                           |    <session123456> mu:account ?user. |
|   GRAPH <auth> {                    |                           |    ?user <authFor> <Bike>            |
|    <SESSION> mu:account ?user.      |                           |   }                                  |
|    ?user <authFor> ?type            |                           |   VALUES (?graph23694) { (<bikes>) } |
|   }                                 |                           | }                                    |
|   VALUES (?graph ?type){            |                           | ```                                  |
|     (<cars> <Car>)                  |                           |                                      |
|     (<bikes> <Bike>)                |                           |                                      |
|   }                                 |                           |                                      |
| }                                   |                           |                                      |
| ```                                 |                           |                                      |
| functional properties: `rdf:type`   |                           |                                      |
| unique variables: `?user`           |                           |                                      |
+-------------------------------------+---------------------------+--------------------------------------+

\normalsize

# Constraint Transformation

The basic task of *constraint rewriting* can be described by a single requirement: the result of running the rewritten query on the full database must be equivalent to running the original query on an intermediate graph constructed by the constraint. However, it is also important that the rewritten query be optimized as much as possible, since otherwise queries will quickly grow too complicated for the database to handle.

Generally speaking, each triple in the incoming query is matched against `?a ?b ?c` in the `CONSTRUCT` statement, and this set of bindings is used to rename the constraint's `WHERE` block to produce the *constraint renaming*, which replaces the original triple.

To avoid redundancy, an important step of the renaming process is calculating the minimal variable dependency for all free variables in the constraint query. For instance, in the example above, both `?type` and `?graph` depend on `?a`, which means that all triples with the same subject can reuse the same renamings of `?type` and `?graph`. 

\small

+--------------------------------+---------------------------+--------------------------------------+
| Constraint                     | Query                     | Rewritten Query                      |
+================================+===========================+======================================+
| ```                            | ```                       | ```                                  |
| CONSTRUCT {                    | SELECT *                  | SELECT ?s ?color                     |
|   ?a ?b ?c                     | WHERE {                   | WHERE {                              |
| }                              |   ?x <hasColor> ?xcolor;  |   GRAPH ?graph23694 {                |
| WHERE {                        |      <hasStyle> ?xstyle;  |       ?x a ?type45123;               |  
|   GRAPH ?graph {               |   ?y <hasColor> ?ycolor.  |          <hasColor> ?xcolor;         |
|    ?a ?b ?c;                   | }                         |          <hasStyle> ?xstyle.         |
|       a ?type                  | ```                       |   GRAPH ?graph23695 {                |
|   }                            |                           |       ?y a ?type45124;               |
|   VALUES (?graph ?type){       |                           |          <hasColor> ?ycolor.         |
|     (<cars> <Car>)             |                           |   }                                  |
|     (<bikes> <Bike>)           |                           |   VALUES (?graph23694 ?type45123){   |
|   }                            |                           |     (<cars> <Car>)                   |
| }                              |                           |     (<bikes> <Bike>)                 |
| ```                            |                           |   }                                  |
| functional properties:         |                           |   VALUES (?graph23695 ?type45124){   |
|    `rdf:type`                  |                           |     (<cars> <Car>)                   |
|                                |                           |     (<bikes> <bike>)                 |
|                                |                           |   }                                  |
|                                |                           |  }                                   |
|                                |                           |  ```                                 |
+--------------------------------+---------------------------+--------------------------------------+

\normalsize

To allow flexibility for ..., few assumptions are made re: deps/values/ filter : ...

-- update queries

-- optimizations

# Annotations and Mu Cache Integration

The Query Rewriter defines annotations, an extension to the SPARQL standard, to ...

# Performance and Caching

Clearly, the Query Rewriter .. latency .. Adding the fact that queries become more complex and thus slower to run, we might expect a serious impact on application performance.

In part, this impact is compensated by the benefits of authorization-aware caching. However, performance can also be vastly improvived by taking into account the fact that most of the queries it receives are generated by a small number of microservices. Therefore we introduce the notion of cache forms, to allow caching on identity modulo URIs and string literals.

# Challenges

There is one important class of challenges which are not addressed in the proof of concept application described below.

## The Incomplete Model

## Model Changes

# Proof of Concept


# Appendix: Basic Algorithms

This appendix gives the basic algorithm for rewriting a query 

```
SELECT V 
WHERE { QW }
```

with the constraint

```
CONSTRUCT { ?a ?b ?c }
WHERE { CW }
```

The function `Depends(a,v)` returns the minimal renaming dependency of the variable `a` in `CW`, namely ... This is defined by .. paths/.../

The set of renaming bindings `B` defines a function `Rename(v, {s}, B)` that maps a free variable in `CW` and the set {s} of variables from `QW` to a renamed variable `v'`. For instance...

The function `New(v)` returns a unique renamed variable based on the variable `v`: `New(?type) = ?type45123`.

```
let B be the empty set of renaming bindings.
RenameQuadsBlock(QW, Bindings)
Apply functional property optimizations and optimize duplicate statements
```

We recurse through Quads blocks:

```
RenameQuadsBlock(QB, Bindings):
  for each quads block QB in QW:
  for each triple T = ?s ?p ?o in QB
  let (T', Bindings) = RenameTriple(?s, ?p, ?o, Bindings)
    replace T with T'
    continue...
```

and for triples do the real work here:

```
RenameTriple(?s, ?p, ?o, Bindings):
  let Match be the matching ?s ?p ?o => ?a ?b ?c, written Match(?s) = ?a
  generate CW', the constraint renaming of CW for B:
    for each free variable v in CW:
      if v is a unique variable replace it with Rename(v, {}, B)
      else
        S := { Match(s) : s in {?s, ?p, ?o}, Depends(a, v) }
        v' := Rename(v, S, B)
        if not v':
          v' := New(v)
          B := Update(B, v, v')
        replace v with v'
  replace ?s ?p ?o by CW' in QW
```


Functional property optimizations... 