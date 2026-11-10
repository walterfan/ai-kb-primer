---
title: "Ch 11 — Code Knowledge Graph"
status: draft
authors:
  - walterfan
last_verified_commit: a1b2c3d
---

# Code Knowledge Graph

The code knowledge graph stores entities (functions, structs, interfaces)
and their relationships (CALLS, IMPLEMENTS, IMPORTS, CONTAINS).

## Bounded relation taxonomy

The graph uses exactly 8 edge types:

1. CONTAINS
2. IMPORTS
3. CALLS
4. IMPLEMENTS
5. EMBEDS
6. DEPENDS_ON
7. RETURNS
8. ACCEPTS

Adding a 9th requires a design review — this constraint prevents the
schema from becoming an unmaintainable "museum" within two sprints.

## Graph store interface

The `GraphStore` interface is defined in the service layer. The current
reference implementation uses Memgraph, but any graph database that
supports Cypher can be plugged in.

## [TODO] Graph visualisation

Add a section on how to render subgraphs for debugging.

<!-- PKB-metadata -->
<!-- layer: L3 -->
<!-- updated_by: ai -->
<!-- review_status: pending -->
<!-- review_score: 0 -->
<!-- reviewed_by: null -->
<!-- commit: a1b2c3d -->
