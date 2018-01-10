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

Conceptually speaking, the access rules are expressed as a dynamic *constraint* expressed in standard SPARQL, namely as a `CONSTRUCT` query. The constraint is thought of as constructing an intermediate *constraint graph* on which incoming SPARQL queries are run. Since the constraint can depend on `mu-session-id` header, the constraint graph will be different for each user, and represents a hypothetical personal graph which that user is authorized to read and/or update. (See Figure 1.)

In practice, an incoming query is optimally rewritten to a form which, when run against the full database, is equivalent to the original query being run against the constraint graph. The result is effectively run against the subset of data which the current user has permission to query or update. 

![Basic mu.semte.ch architecture with the Query Rewriter and conceptual constraint graph](../rewriter.png)

As a simple example, here is a constraint describing a database where triples whose subject have type `<Car>` are stored in the `<cars>` graph, and similarly `<Bike>`s in the `<bikes>` graph. The `<auth>` graph contains triples specifying which users are authorized to see which types. The placeholder `<SESSION>` is dynamically replaced with the `mu-session-id` header. (`rdf:type` is declared as a "functional property", defined see below). To save space, prefixe declarations are omitted.

\small

+--------------------------------+---------------------------+--------------------------------------+
| Constraint                     | Query                     | Rewritten Query                      |
+================================+===========================+======================================+
| ```                            | ```                       | ```                                  |
| CONSTRUCT {                    | SELECT *                  | SELECT ?s ?color                     |
|   ?a ?b ?c                     | WHERE {                   | WHERE {                              |
| }                              |   ?s a <Bike>;            |   GRAPH ?graph23694 {                |
| WHERE {                        |      <hasColor> ?color.   |       ?s a <Bike>;                   |  
|   GRAPH ?graph {               | }                         |          <hasColor> ?color.          |
|    ?a ?b ?c;                   | ```                       |    }                                 |
|       a ?type                  |                           |   GRAPH <auth> {                     |
|   }                            |                           |    <session123456> mu:account ?user. |
|   GRAPH <auth> {               |                           |    ?user <authFor> <Bike>            |
|    <SESSION> mu:account ?user. |                           |   }                                  |
|    ?user <authFor> ?type       |                           |   VALUES (?graph23694) { (<bikes>) } |
|   }                            |                           | }                                    |
|   VALUES (?graph ?type){       |                           | ```                                  |
|     (<cars> <Car>)             |                           |                                      |
|     (<bikes> <Bike>)           |                           |                                      |
|   }                            |                           |                                      |
| }                              |                           |                                      |
| ```                            |                           |                                      |
+--------------------------------+---------------------------+--------------------------------------+

\normalsize

# Constraint Transformation

The query transformation process ...

-- minimal variable dependency
-- values vs. filter 
-- nested ...
-- ..
-- optimization
-- optimizations

# Annotations and Mu Cache Integration

The Mu Query Rewriter 

# Performance and Caching

Clearly, the Query Rewriter .. latency .. Adding the fact that queries become more complex and thus slower to run, we might expect a serious impact on application performance.

In part, this impact is compensated by the benefits of authorization-aware caching. However, performance can also be vastly improvived by taking into account the fact that most of the queries it receives are generated by a small number of microservices. Therefore we introduce the notion of cache forms, to allow caching on identity modulo URIs and string literals.

# Challenges

There is one important class of challenges which are not addressed in the proof of concept application described below.

## The Incomplete Model

## Model Changes

# Proof of Concept