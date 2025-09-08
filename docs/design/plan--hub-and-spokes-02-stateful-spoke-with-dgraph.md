# Implementation Plan: 02 - Stateful Spoke with Dgraph

**Goal:** Introduce the first stateful component into the ecosystem by creating a Dgraph-backed Spoke with a static dataset.

This plan is designed to be executed sequentially. It assumes you have cloned all necessary repositories and are executing the commands from within the correct repository's root directory.

## Task Checklist

- [ ] **Phase 2: Simple Stateful Backend (Dgraph "Hello World")**
    - [ ] [Task 2.1: Create `subgraph-dgraph-static` Spoke](#task-21-create-subgraph-dgraph-static-spoke)

---

### Phase 2: Simple Stateful Backend (Dgraph "Hello World")

#### Task 2.1: Create `subgraph-dgraph-static` Spoke

1.  From within the `subgraph-dgraph-static` repository, establish the Dgraph data directory structure:

```sh
mkdir -p ./data/{zero,alpha}
```

2.  Create the Dgraph schema at `schema.graphql`. This schema includes both Dgraph's `@id` directive for unique key enforcement and the Federation `@key` directive for compatibility with the supergraph.

    ```graphql
    type Character @key(fields: "id") {
        id: ID! @id
        name: String! @search(by: [hash])
        species: String
    }
    ```

3.  Create the `compose.yaml` file. This configuration is a direct implementation of best practices, adapted for our Spoke architecture.

    ```yaml
    version: '3.8'
    services:
        zero:
            image: dgraph/dgraph:v21.03.2
            command: dgraph zero --my=zero:5080
            volumes:
                - ./data/zero:/dgraph
            networks:
                - default
        alpha:
            image: dgraph/dgraph:v21.03.2
            command: dgraph alpha --my=alpha:7080 --zero=zero:5080 --graphql_schema=/dgraph/schema.graphql
            volumes:
                - ./data/alpha:/dgraph
                - ./schema.graphql:/dgraph/schema.graphql:ro
                - ./data.json:/dgraph/data.json:ro
            ports:
                - "8080:8080" # GraphQL endpoint
            networks:
                - default
        ratel:
            image: dgraph/ratel:v21.03.2
            ports:
                - "8000:8000"
            networks:
                - default
    networks:
        default:
            name: graphtastic_net
            external: true
    ```

4.  Create the static dataset at `data.json`. This file contains the data to be loaded.

    ```json
    [
        {
            "uid": "_:char1",
            "dgraph.type": "Character",
            "id": "char1",
            "name": "Luke Skywalker",
            "species": "Human"
        },
        {
            "uid": "_:char2",
            "dgraph.type": "Character",
            "id": "char2",
            "name": "R2-D2",
            "species": "Droid"
        }
    ]
    ```

5.  Create a `Makefile` within the Spoke to manage its lifecycle, including schema/data loading.

    ```makefile
    .PHONY: up down seed clean

    up:
        docker-compose up -d

    down:
        docker-compose down

    seed:
        curl -X POST localhost:8080/admin/schema --data-binary '@schema.graphql' -H 'Content-Type: application/graphql'
        curl -X POST localhost:8080/admin/import --data-binary '@data.json' -H 'Content-Type: application/json'

    clean:
        rm -rf ./data/zero/* ./data/alpha/*
    ```

6.  **Standalone Verification (Optional but Recommended):**
        *   From within the `subgraph-dgraph-static` repository:
        *   Remove the shared network if needed:
``sh
docker network rm graphtastic_net
```
