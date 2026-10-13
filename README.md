# MeMAD Knowledge Graph

Deployment of the MeMAD Knowledge Graph on QLever and RDF Workbench with Docker.

**Table of contents**

- [Requirements](#requirements)
- [Setup](#setup)
  - [Dumps](#dumps)
  - [Ontologies](#ontologies)
  - [Configuration](#configuration)
  - [First start](#first-start)
  - [Data refresh](#data-refresh)
- [Example Queries](#example-queries)

## Requirements

- Docker with compose v2
- About 3 GB of disk for the dumps and the built index

## Setup

Note: the setup requires that the data dumps are available in the `dumps` directory. They are not included in this repository due to their size.

### Dumps

The index is built from the INA-LD broadcast archive drop of April 2020. The expected layout is:

```
dumps/
  ina-ld/
    ld.ttl
    ld_sujets.ttl
    ld_flow_filenames.ttl
```

These three files reproduce the `http://data.memad.eu/graph/ina-ld` graph as served by the previous Virtuoso instance (3,202,303 triples after deduplication). The Qleverfile carries commented entries for the remaining 2020 material (`ina-pa`, `yle`) and the 2021 update drop; treat the 2021 files as updates to reconcile, not as additive data.

### Ontologies

The ontology graph `http://data.memad.eu/ontology` is built from two files versioned in `ontology/`:

- `ebucore-20182011.nt` - the [EbuCore release of 2018-11-20](http://www.ebu.ch/metadata/ontologies/ebucore/20182011/ebucore.rdf) (md5 `6238e5252396187d01dc1eabeab484f6`), the version the data was generated against, converted to N-Triples with `rapper`
- `memad.ttl` - the MeMAD ontology extension ([source](https://github.com/MeMAD-project/rdf-converter/blob/master/ontology/memad.ttl), md5 `ad258eb85bc21d742548e09d79904d7d`)

### Configuration

Copy `.env.example` into `.env` and update its content:

- `QLEVER_ACCESS_TOKEN` - Token required for QLever administrative operations (generate with `openssl rand -hex 24`)
- `WORKBENCH_ADMIN_PASSWORD` - RDF Workbench admin password (seeds the admin user on first start only)
- `DEREFERENCE_PATHS` - IRI path prefixes the workbench serves as dereferenceable resources

### First start

The first start builds the SPARQL index from the dumps. The build takes seconds for this dataset; run it detached and follow the logs to check its status:

```bash
docker compose up -d
docker compose logs -f qlever-index
```

The `qlever-index` service builds the index into a staging directory, then swaps it into place, so the running engine is never served a partial index. `docker compose ps` shows the stack ready once `qlever` and `workbench` report healthy and `qlever-index` has exited with code 0.

The SPARQL endpoint answers on `http://localhost:8851` and the workbench console on `http://localhost:3024` (admin on `http://localhost:3023`).

### Data refresh

Rebuild the index after the dumps change:

```bash
QLEVER_FORCE_REBUILD=1 docker compose run --rm qlever-index
docker compose up -d --force-recreate qlever
```

The rebuild stages a fresh index, promotes it, and keeps the previous generation on disk until the next rebuild.

## Example Queries

Graph-scoped triple counts:

```sparql
SELECT (COUNT(*) AS ?c) WHERE { GRAPH <http://data.memad.eu/graph/ina-ld> { ?s ?p ?o } }
```

Text search over programme titles. The keyword index is lowercased, so search terms must be lowercase:

```sparql
PREFIX ebucore: <http://www.ebu.ch/metadata/ontologies/ebucore/ebucore#>
PREFIX ql: <http://qlever.cs.uni-freiburg.de/builtin-functions/>
SELECT ?id ?title WHERE {
  GRAPH <http://data.memad.eu/graph/ina-ld> { ?id a ebucore:TVProgramme }
  ?id ebucore:title ?title .
  ?title ql:has-word "foot"
} LIMIT 10
```
