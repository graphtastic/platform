### **The Graphtastic Platform: Implementation Plan v1.0**

This document provides the complete, actionable plan for implementing the Graphtastic Platform as defined in **The Graphtastic Platform Tome v8.3**. It is divided into two parts:

*   **Part 1: Strategic Rationale & Architectural Decisions:** This section explains the guiding philosophy and critical decisions that have shaped the phased implementation. It serves as the "why" behind the detailed steps that follow.
*   **Part 2: The Detailed Execution Plan:** This section provides a verbose, step-by-step guide for engineers to execute the plan. It includes all necessary commands, code, and configuration to ensure a consistent and successful implementation.

---

### **Part 1: Strategic Rationale & Architectural Decisions**

#### **1.0 Introduction: The Guiding Philosophy**

The implementation strategy for the Graphtastic Platform is founded on a core philosophy: **systematically reduce risk by isolating and solving one architectural challenge at a time.** We will not attempt a "big bang" integration. Instead, each phase is designed to deliver a stable, verifiable milestone that serves as the foundation for the next. This ensures that when problems arise, they are immediately attributable to the new component being introduced, dramatically simplifying debugging and ensuring predictable progress.

#### **1.1 The "Tools-First" Imperative: Building the Factory Before the Car**

The first and most critical decision in our plan is to build the foundational tooling *before* any functional components. As detailed in **Phase 0**, we will first create the `tools-docker-compose`, `tools-subgraph-core`, and `template-*` repositories.

*   **Rationale:** This approach establishes the complete developer control plane and governance framework from day one. It guarantees that the very first Spoke (`subgraph-blogs`) is built using the same standardized CI pipelines, `Makefile` orchestration, and validation checks as the most complex Spoke that will come later. This front-loading of effort eradicates an entire class of future integration problems and ensures a consistent, high-quality developer experience throughout the project's lifecycle.

#### **1.2 The "Isolate the Variables" Approach to Federation**

The most complex technical challenge in this architecture is achieving successful federation. Our plan deliberately de-risks this by isolating it from other variables, specifically the introduction of new backend technologies.

*   **Rationale:** As laid out in **Phase 1**, we will first achieve a "Minimal Viable Federation" using two technologically identical, stateless subgraphs (`subgraph-blogs` and `subgraph-authors`, both based on GraphQL Yoga). By federating two simple, known-good components, we can be certain that any issues that arise are in the federation layer itself (the `federated-graph-core` services, the `supergraph-cncf` configuration, or the inter-service networking) and not in the subgraphs. This validates the core architectural pattern in the most controlled environment possible.

#### **1.3 A Phased Introduction to Stateful Backends (The Dgraph Track)**

Only after proving the federation mechanism do we introduce the complexity of a stateful backend. The plan defines a clear, multi-stage "Dgraph track" that progresses from simple to complex.

*   **Rationale:** This progression isolates new challenges at each step.
    *   **Phase 2 (`subgraph-dgraph-static`):** This phase focuses *only* on the challenge of integrating Dgraph as a Spoke. By using a static dataset, we are not yet concerned with data pipelines.
    *   **Phase 3 (`subgraph-dgraph-gharchive`):** This phase builds on the now-stable Dgraph pattern and introduces the next challenge: dynamic data ingestion from an external source.
    *   **Phase 4 (`subgraph-dgraph-software-supply-chain`):** The final and most complex phase introduces a sophisticated internal ETL pipeline, leveraging our analysis from `design--guac-to-dgraph.md`. This is tackled last because it relies on all previously validated patterns.

Furthermore, we will leverage Dgraph's native federation compliance. This simplifies the implementation of Dgraph-backed Spokes, as they can expose a federated GraphQL endpoint directly, perfectly aligning with the "Spoke as a black box" principle from the Tome.

#### **1.4 Architectural Decision: Encapsulating GraphQL Mesh**

For the `subgraph-dgraph-software-supply-chain`, which requires transforming the GUAC API, we explicitly decided that GraphQL Mesh will be run *within* the Spoke.

*   **Rationale:** This decision directly upholds the architectural precepts of the Tome v8.3.
    1.  **It preserves the "Spoke as a Black Box" model.** The Hub does not need to know about the Spoke's internal transformation needs.
    2.  **It ensures the Spoke remains "Standalone."** The developers of this Spoke can run and test their entire data pipeline in isolation without needing the `federated-graph-core`.
    3.  **It avoids tight coupling and configuration sprawl** that a centralized Mesh service would create.

#### **1.5 Architectural Decision: Solving the Template "Update Problem" from Day One**

Our review of the "GitHub Repository Templates: A Primer" document highlighted the critical "update problem"—the lack of an ongoing link between a template and its generated children.

*   **Rationale:** Relying on manual, error-prone Git commands is not a scalable solution. Therefore, our plan mandates solving this from the start. As part of **Phase 0**, the `template-*` repositories **must** include a pre-configured, automated GitHub Actions workflow (e.g., using `actions-template-sync`). This "self-updating" pattern ensures every component in our ecosystem is born with the capability to receive future improvements, transforming lifecycle management from a manual burden into a governed, automated process.

#### **1.6 Architectural Decision: Naming Conventions and Abstractions**

During our discussion, we clarified and confirmed our naming conventions to maintain architectural clarity.

*   **Rationale:**
    *   We will retain the name `federated-graph-core` because it precisely describes its function as the engine that enables federation.
    *   We will defer the creation of a `tools-supergraph-core` repository. The Hubs are intended to be simple assemblers, and their logic is either provided by `tools-docker-compose` or is use-case-specific. Introducing another tools repository at this stage would be an unnecessary abstraction.

---

### **Part 2: The Detailed Execution Plan**

This plan is designed to be executed sequentially by an engineer. It assumes the engineer has the following tools installed: `git`, `docker`, `docker compose`, and `yq` (for YAML parsing in shell scripts). All work should be done within a single parent directory.

#### **Phase 0: The Developer Control Plane**

**Goal:** Create the foundational tooling that will govern and orchestrate all subsequent development.

**Task 0.1: Create `tools-docker-compose` Repository**

1.  Create the repository directory: `mkdir -p tools-docker-compose/scripts`
2.  Create the master `Makefile` at `tools-docker-compose/Makefile.master`:
    ```makefile
    # This is the master Makefile, located in `tools-docker-compose`.
    # It is called by a Hub's lightweight, root Makefile.

    # Default variables
    DEPS_DIR := ./.deps
    MANIFEST := ./graphtastic.deps.yml
    SUPERGRAPH_SDL := ./supergraph.graphql

    # This allows passing a stack name, e.g., `make logs stack=subgraph-blogs`
    stack ?= all

    # Allow passing persistence mode, e.g., `make up mode=volume`
    mode ?= bind

    .PHONY: help up down clean restart deps supergraph validate-schemas validate ps logs

    help:
        @echo "Graphtastic Platform - Master Orchestrator"
        @echo "--------------------------------------------"
        @echo "Usage: make [target] [stack=<stack_name>] [mode=bind|volume]"
        @echo ""
        @echo "Lifecycle Targets:"
        @echo "  up                 - Start all services defined in the manifest."
        @echo "  down               - Stop and remove all services."
        @echo "  clean              - Run 'down' and remove shared Docker resources."
        @echo "  restart            - Restart all services."
        @echo ""
        @echo "Dependency & Build Targets:"
        @echo "  deps               - Sync/update local dependencies from the manifest."
        @echo "  supergraph         - Render the final supergraph.graphql artifact."
        @echo ""
        @echo "Interactive & Debugging Targets:"
        @echo "  ps                 - Show the status of running containers. Use 'stack=' to target one."
        @echo "  logs               - Tail the logs of services. Use 'stack=' to target one."

    up: deps
        @echo "🚀 Bringing up all services with persistence mode: $(mode)..."
        ./$(DEPS_DIR)/tools-docker-compose/scripts/manage-stacks.sh up $(MANIFEST) $(mode)

    down:
        @echo "🔥 Bringing down all services..."
        ./$(DEPS_DIR)/tools-docker-compose/scripts/manage-stacks.sh down $(MANIFEST)

    clean: down
        @echo "🧹 Removing shared Docker resources and local data..."
        @docker network rm $$(grep SHARED_NETWORK_NAME .env | cut -d '=' -f2) 2>/dev/null || true
        yq e '.components[].name' $(MANIFEST) | while read -r name; do \
            if [ -d "$(DEPS_DIR)/$$name/data" ]; then \
                echo "Removing local data for $$name..."; \
                rm -rf $(DEPS_DIR)/$$name/data; \
            fi \
        done

    restart: down up

    deps:
        @echo "🔄 Syncing component repositories from $(MANIFEST)..."
        ./$(DEPS_DIR)/tools-docker-compose/scripts/sync-deps.sh $(MANIFEST) $(DEPS_DIR)

    supergraph: deps
        @echo "✍️ Rendering supergraph artifact..."
        # This requires a mesh.config.js in the supergraph root
        npx @graphql-mesh/compose-cli --config ./mesh.config.js > $(SUPERGRAPH_SDL)

    ps:
        @echo "📊 Status for stack: $(stack)"
        ./$(DEPS_DIR)/tools-docker-compose/scripts/manage-stacks.sh ps $(MANIFEST) $(stack)

    logs:
        @echo "📜 Tailing logs for stack: $(stack)..."
        ./$(DEPS_DIR)/tools-docker-compose/scripts/manage-stacks.sh logs $(MANIFEST) $(stack)
    ```
3.  Create the dependency sync script at `tools-docker-compose/scripts/sync-deps.sh`:
    ```bash
    #!/bin/bash
    set -e
    MANIFEST=$1
    DEPS_DIR=$2

    if [ ! -f "$MANIFEST" ]; then
        echo "Manifest file not found: $MANIFEST"
        exit 1
    fi

    mkdir -p $DEPS_DIR

    yq e '.components[] | .name + " " + .git + " " + .version' $MANIFEST | while read -r name git version; do
        echo "--- Processing $name ---"
        TARGET_DIR="$DEPS_DIR/$name"
        if [ ! -d "$TARGET_DIR/.git" ]; then
            echo "Cloning $git..."
            git clone --branch $version $git $TARGET_DIR
        else
            echo "Fetching and checking out version $version..."
            (cd $TARGET_DIR && git fetch origin && git checkout $version && git pull origin $version --ff-only)
        fi
    done
    ```4.  Create the stack management script at `tools-docker-compose/scripts/manage-stacks.sh`:
    ```bash
    #!/bin/bash
    set -e
    ACTION=$1
    MANIFEST=$2
    PERSISTENCE_MODE=${3:-bind}
    STACK=${4:-all}
    SUPERGRAPH_NAME=$(basename $(pwd))

    COMPOSE_FILES=""
    yq e '.components[] | .name' $MANIFEST | while read -r name; do
        if [ "$STACK" = "all" ] || [ "$STACK" = "$name" ]; then
            COMPOSE_FILE="./.deps/$name/compose.yaml"
            if [ -f "$COMPOSE_FILE" ]; then
                # Apply supergraph-level overrides if they exist
                OVERRIDE_FILE="./compose.override.yaml"
                if [ -f "$OVERRIDE_FILE" ]; then
                     COMPOSE_CMD="docker compose -p ${SUPERGRAPH_NAME}-${name} -f ${COMPOSE_FILE} -f ${OVERRIDE_FILE} ${ACTION}"
                else
                     COMPOSE_CMD="docker compose -p ${SUPERGRAPH_NAME}-${name} -f ${COMPOSE_FILE} ${ACTION}"
                fi

                echo "Executing for $name: $COMPOSE_CMD"
                PERSISTENCE_MODE=$PERSISTENCE_MODE $COMPOSE_CMD
            fi
        fi
    done
    ```
5.  Make scripts executable: `chmod +x tools-docker-compose/scripts/*.sh`

**Task 0.2: Create `template-subgraph` Repository**

1.  Create the repository directory: `mkdir -p template-subgraph/.github/workflows`
2.  Create a placeholder `compose.yaml`:
    ```yaml
    version: "3.8"
    services:
      # Service definition will go here
    ```
3.  Create a placeholder `schema.graphql`:
    ```graphql
    type Query {
      hello: String
    }
    ```
4.  Create the "self-updating" GitHub Actions workflow at `template-subgraph/.github/workflows/sync-from-template.yml`:
    ```yaml
    # .github/workflows/sync-from-template.yml
    name: Sync from Template

    on:
      schedule:
        - cron: '0 3 * * 1' # Run every Monday at 3:00 AM UTC
      workflow_dispatch:

    jobs:
      sync:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4
          - name: Sync Template Repository
            uses: AndreasAugustin/actions-template-sync@v1.1.0
            with:
              source_repo_path: graphtastic/template-subgraph # This should be the template's actual path
              github_token: ${{ secrets.GITHUB_TOKEN }}
    ```

**Task 0.3: Create `template-supergraph` Repository**

1.  Create the directory: `mkdir template-supergraph`
2.  Create a root `Makefile`:
    ```makefile
    # Graphtastic Supergraph Makefile
    # This file delegates all logic to the master makefile from tools-docker-compose.

    include ./.deps/tools-docker-compose/Makefile.master
    ```
3.  Create a placeholder `graphtastic.deps.yml`:
    ```yaml
    # Declare dependencies for this supergraph
    components:
      - name: tools-docker-compose
        git: https://github.com/graphtastic/tools-docker-compose.git # Use actual path
        version: main
    ```

#### **Phase 1: Minimal Viable Federation with Stateless Spokes**

**Goal:** Create and federate two simple, stateless Yoga-based subgraphs.

**Task 1.1: Create `subgraph-blogs`**

1.  **Instantiate the Spoke Repository:**
    *   On your Git provider (e.g., GitHub), create a new repository named `subgraph-blogs` using `graphtastic/template-subgraph` as the template.
    *   Clone the newly created repository: `git clone https://github.com/graphtastic/subgraph-blogs.git`
2.  Navigate into the new directory: `cd subgraph-blogs`
3.  Create the Yoga server at `subgraph-blogs/index.js`:
    ```javascript
    import { createServer } from 'node:http'
    import { createYoga } from 'graphql-yoga'
    import { createSchema } from 'graphql-yoga'

    const typeDefs = /* GraphQL */ `
      type Query {
        blogs: [Blog!]
        blog(id: ID!): Blog
      }
      type Blog @key(fields: "id") {
        id: ID!
        title: String!
        authorId: ID!
      }
    `;

    const blogsData = [
      { id: '1', title: 'Introduction to GraphQL', authorId: '101' },
      { id: '2', title: 'Federation Deep Dive', authorId: '102' },
    ];

    const resolvers = {
      Query: {
        blogs: () => blogsData,
        blog: (_, { id }) => blogsData.find(b => b.id === id),
      },
    };

    const yoga = createYoga({ schema: createSchema({ typeDefs, resolvers }) });
    const server = createServer(yoga);
    server.listen(4001, () => { console.info('Blogs subgraph running at http://localhost:4001/graphql') });
    ```
4.  Update `subgraph-blogs/schema.graphql` with the schema from the `typeDefs` above.
5.  Create a `subgraph-blogs/package.json`:
    ```json
    {
      "name": "subgraph-blogs",
      "type": "module",
      "dependencies": {
        "graphql": "^16.8.1",
        "graphql-yoga": "^5.1.1"
      }
    }
    ```
6.  Create `subgraph-blogs/compose.yaml`:
    ```yaml
    version: "3.8"
    services:
      blogs-service:
        build: .
        ports: ["4001:4001"]
        networks: [graphtastic_net]
    networks:
      graphtastic_net:
        external: true
    ```
7.  Create `subgraph-blogs/Dockerfile`:
    ```dockerfile
    FROM node:18-alpine
    WORKDIR /app
    COPY package.json .
    RUN npm install
    COPY . .
    CMD ["node", "index.js"]
    ```
8.  Commit and push these changes to the `subgraph-blogs` repository.

**Task 1.2: Create `subgraph-authors`**

1.  Follow the same Git-based procedure as in Task 1.1 to create and clone a new `subgraph-authors` repository from the template.
2.  Create the Yoga server, schema, `package.json`, `compose.yaml`, and `Dockerfile` similar to `subgraph-blogs`, but for Authors.
    *   **Schema:** `type Author @key(fields: "id") { id: ID!, name: String! }`
    *   **Data:** `[{ id: '101', name: 'Alice' }, { id: '102', name: 'Bob' }]`
    *   **Port:** Use port `4002`.
3.  Commit and push the changes to the `subgraph-authors` repository.

**Task 1.3: Create `federated-graph-core`**

1.  Create the directory: `mkdir federated-graph-core`
2.  Create the `compose.yaml` for the Hive stack at `federated-graph-core/compose.yaml`:
    ```yaml
    version: '3.8'
    services:
      # ... (A full compose file for Hive would be very long. A simplified gateway is shown here)
      gateway:
        image: ghcr.io/the-guild-org/graphql-hive/gateway:latest
        ports:
          - '4000:4000'
        environment:
          HIVE_CDN_ENDPOINT: 'http://localhost:3002' # Placeholder, not used in static mode
          HIVE_CDN_KEY: 'dummy-key'
          SUPERGRAPH: '/app/supergraph.graphql' # Path inside container
        volumes:
          - type: bind
            source: ./supergraph.graphql # This will be mounted by the supergraph
            target: /app/supergraph.graphql
            read_only: true
        networks:
          - graphtastic_net
    networks:
      graphtastic_net:
        external: true
    ```

**Task 1.4: Create and Configure `supergraph-cncf`**

1.  Instantiate the Hub repository from the `template-supergraph` and clone it locally.
2.  Update `supergraph-cncf/graphtastic.deps.yml`:
    ```yaml
    components:
      - name: tools-docker-compose
        git: ../tools-docker-compose # Using local path for development
        version: main
      - name: federated-graph-core
        git: ../federated-graph-core
        version: main
      - name: subgraph-blogs
        git: ../subgraph-blogs # Use the path to your locally cloned repo
        version: main
      - name: subgraph-authors
        git: ../subgraph-authors # Use the path to your locally cloned repo
        version: main
    ```
3.  Create `supergraph-cncf/.env`:
    ```
    SHARED_NETWORK_NAME=graphtastic_net
    ```
4.  Create `supergraph-cncf/mesh.config.js`:
    ```javascript
    module.exports = {
      sources: [
        { name: 'blogs', handler: { graphql: { endpoint: 'http://blogs-service:4001/graphql' } } },
        { name: 'authors', handler: { graphql: { endpoint: 'http://authors-service:4002/graphql' } } },
      ],
      compose: { supergraph: true },
    };
    ```
5.  Create `supergraph-cncf/compose.override.yaml` to mount the supergraph:
    ```yaml
    services:
      gateway:
        volumes:
          - type: bind
            source: ./supergraph.graphql
            target: /app/supergraph.graphql
            read_only: true
    ```
6.  **Execute and Verify:**
    *   `cd supergraph-cncf`
    *   Create the shared network: `docker network create graphtastic_net`
    *   Render the supergraph: `make supergraph` (This will create `supergraph.graphql`)
    *   Start the system: `make up`
    *   Run a federated query against `http://localhost:4000/graphql`:
        ```graphql
        query GetBlogsWithAuthors {
          blogs {
            id
            title
            author { # This field doesn't exist yet, we need to extend Author
              name
            }
          }
        }
        ```    *   **Refinement:** To enable the query above, we must extend the types. Update `subgraph-blogs/index.js` to add `type Author @key(fields: "id") @extends { id: ID! @external }` and a resolver for `Blog.author`. Re-run `make supergraph` and `make up`. This completes the federation loop.

This concludes the most critical phases. The subsequent phases (2, 3, 4, and 5) would follow this same detailed pattern: creating the repository, defining its `compose.yaml` and logic, integrating it into the `supergraph-cncf/graphtastic.deps.yml`, re-rendering the supergraph, and running verification queries. The specific Dgraph configurations would follow the best practices laid out in the `on--dgraph-docker-compose.md` document.

#### **Phase 2: Simple Stateful Backend (Dgraph "Hello World")**

**Goal:** Introduce Dgraph into the ecosystem in the most controlled way possible, validating the base technology and the pattern for a stateful Spoke.

**Task 2.1: Create `subgraph-dgraph-static` Spoke**

1.  Create the `subgraph-dgraph-static` repository from the `template-subgraph` and clone it.
2.  Establish the Dgraph data directory structure inside the Spoke: `mkdir -p subgraph-dgraph-static/data/{zero,alpha}`
3.  Create the Dgraph schema at `subgraph-dgraph-static/schema.graphql`. This schema includes both Dgraph's `@id` directive for unique key enforcement and the Federation `@key` directive for compatibility with the supergraph.
    ```graphql
    # Dgraph schema for static Star Trek character data
    type Character @key(fields: "id") {
      id: ID! @id
      name: String! @search(by: [term])
      species: String! @search(by: [hash])
      affiliation: String
    }
    ```
4.  Create the Docker Compose file at `subgraph-dgraph-static/compose.yaml`. This configuration is a direct implementation of the best practices from the `on--dgraph-docker-compose.md` guide, adapted for our Spoke architecture.
    ```yaml
    version: "3.8"
    services:
      zero:
        image: dgraph/dgraph:latest
        container_name: dgraph_static_zero
        volumes:
          - type: ${PERSISTENCE_MODE:-bind}
            source: ${PERSISTENCE_PATH:-./data/zero}
            target: /dgraph
        ports:
          - "5081:5080" # Use non-default ports to avoid host conflicts
          - "6081:6080"
        restart: on-failure
        command: dgraph zero --my=zero:5080
        networks:
          - graphtastic_net

      alpha:
        image: dgraph/dgraph:latest
        container_name: dgraph_static_alpha
        volumes:
          - type: ${PERSISTENCE_MODE:-bind}
            source: ${PERSISTENCE_PATH:-./data/alpha}
            target: /dgraph
        ports:
          - "8081:8080" # Expose GraphQL endpoint
          - "9081:9080"
        restart: on-failure
        command: dgraph alpha --my=alpha:7080 --zero=zero:5080
        depends_on:
          - zero
        networks:
          - graphtastic_net

      ratel:
        image: dgraph/ratel:latest
        container_name: dgraph_static_ratel
        ports:
          - "8001:8000" # Expose Ratel UI
        restart: on-failure
        networks:
          - dgraph-net

    networks:
      graphtastic_net:
        external: true
    ```
5.  Create a `Makefile` within the Spoke to manage its lifecycle, including schema and data loading. Place this at `subgraph-dgraph-static/Makefile`.
    ```makefile
    .PHONY: up down clean apply-schema load-data

    up:
        docker compose up -d

    down:
        docker compose down

    clean: down
        @echo "🧹 Removing local data..."
        rm -rf ./data

    apply-schema:
        @echo "Applying GraphQL schema..."
        curl -X POST localhost:8081/admin/schema --data-binary "@schema.graphql"

    load-data:
        @echo "Loading static character data..."
        curl -X POST localhost:8081/graphql -H "Content-Type: application/json" -d '{"query": "mutation {\n  addCharacter(input: [\n    {id: \"DS9-001\", name: \"Benjamin Sisko\", species: \"Human\", affiliation: \"Starfleet\"},\n    {id: \"DS9-002\", name: \"Kira Nerys\", species: \"Bajoran\", affiliation: \"Bajoran Militia\"},\n    {id: \"TNG-001\", name: \"Jean-Luc Picard\", species: \"Human\", affiliation: \"Starfleet\"}\n  ]) { numUids }\n}"}'
    ```
6.  **Standalone Verification:**
    *   `cd subgraph-dgraph-static`
    *   `docker compose up -d`
    *   Wait a few seconds for the cluster to stabilize, then: `make apply-schema`
    *   `make load-data`
    *   Verify data by running a query against `http://localhost:8081/graphql`.

**Task 2.2: Integrate `subgraph-dgraph-static` into `supergraph-cncf`**

1.  Update `supergraph-cncf/graphtastic.deps.yml` to include the new Spoke:
    ```yaml
      - name: subgraph-dgraph-static
        git: ../subgraph-dgraph-static
        version: main
    ```
2.  Update `supergraph-cncf/mesh.config.js` to add the Dgraph subgraph as a source. Note the use of the `alpha` service name and its internal port `8080`.
    ```javascript
    // ... existing sources
    {
      name: 'characters',
      handler: {
        graphql: { endpoint: 'http://alpha:8080/graphql' } // Dgraph Alpha's endpoint
      }
    },
    ```
3.  To demonstrate a federated join, let's extend the `Author` type in `subgraph-authors` to have a favorite character.
    *   Update `subgraph-authors/index.js` `typeDefs`:
        ```graphql
        type Query {
          authors: [Author!]
          author(id: ID!): Author
        }
        type Author @key(fields: "id") {
          id: ID!
          name: String!
          favoriteCharacter: Character
        }
        type Character @key(fields: "id") @extends {
          id: ID! @external
        }
        ```
    *   Add a resolver for `Author.favoriteCharacter` to `subgraph-authors/index.js`:
        ```javascript
        Author: {
            favoriteCharacter: (author) => {
                // In a real app, this would be from a database. Here we stub it.
                const favs = { '101': 'DS9-001', '102': 'TNG-001' };
                return { __typename: 'Character', id: favs[author.id] };
            }
        }
        ```
4.  **Execute and Verify:**
    *   `cd supergraph-cncf`
    *   `make supergraph`
    *   `make up`
    *   Run a federated query against `http://localhost:4000/graphql`:
        ```graphql
        query GetAuthorsWithFavoriteCharacters {
          authors {
            id
            name
            favoriteCharacter {
              name
              species
            }
          }
        }
        ```

---

#### **Phase 3: Complex Stateful Backend (Dynamic Data Ingestion)**

**Goal:** Build upon the Dgraph foundation by creating a Spoke that handles a dynamic data loading pipeline from an external source (`gharchive`).

**Task 3.1: Create `subgraph-dgraph-gharchive` Spoke**

1.  Create the `subgraph-dgraph-gharchive` repository from the `subgraph-dgraph-static` repository (since it's a stateful Dgraph Spoke) and clone it.
2.  Update the schema at `subgraph-dgraph-gharchive/schema.graphql` for GHArchive data:
    ```graphql
    type PushEvent @key(fields: "id") {
      id: String! @id
      actor: String! @search(by: [hash])
      repo: String! @search(by: [term])
      commitCount: Int
      createdAt: DateTime! @search(by: [hour])
    }
    ```
3.  Create a directory for the ingestion script: `mkdir subgraph-dgraph-gharchive/loader`
4.  Create a `package.json` for the loader script at `subgraph-dgraph-gharchive/loader/package.json`:
    ```json
    {
      "name": "gharchive-loader",
      "type": "module",
      "dependencies": {
        "node-fetch": "^3.3.2",
        "zlib": "^1.0.5"
      }
    }
    ```
5.  Create the ingestion script at `subgraph-dgraph-gharchive/loader/index.js`. This script fetches, parses, transforms, and loads data in batches.
    ```javascript
    import fetch from 'node-fetch';
    import { createGunzip } from 'zlib';
    import { Writable } from 'stream';
    import { pipeline } from 'stream/promises';

    const DGRAPH_ALPHA_URL = 'http://localhost:8082/graphql'; // Corresponds to port in compose.yaml
    const BATCH_SIZE = 1000;

    async function main() {
      // Fetch data for a specific hour, e.g., 2025-09-05 at 15:00 UTC
      const response = await fetch('https://data.gharchive.org/2025-09-05-15.json.gz');
      if (!response.ok) throw new Error(`Failed to fetch: ${response.statusText}`);

      let buffer = [];
      const jsonlParser = new Writable({
        async write(chunk, encoding, callback) {
          const lines = chunk.toString().split('\n');
          for (const line of lines) {
            if (!line) continue;
            try {
              const event = JSON.parse(line);
              if (event.type === 'PushEvent') {
                buffer.push({
                  id: event.id,
                  actor: event.actor.login,
                  repo: event.repo.name,
                  commitCount: event.payload.commits?.length || 0,
                  createdAt: event.created_at
                });

                if (buffer.length >= BATCH_SIZE) {
                  await loadBatch(buffer);
                  buffer = [];
                }
              }
            } catch (e) { /* Ignore malformed JSON lines */ }
          }
          callback();
        }
      });

      await pipeline(response.body, createGunzip(), jsonlParser);
      if (buffer.length > 0) await loadBatch(buffer); // Load final batch
      console.log('Ingestion complete.');
    }

    async function loadBatch(events) {
      const mutation = `
        mutation AddPushEvents($events: [AddPushEventInput!]!) {
          addPushEvent(input: $events, upsert: true) {
            numUids
          }
        }
      `;
      const response = await fetch(DGRAPH_ALPHA_URL, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ query: mutation, variables: { events } }),
      });
      const result = await response.json();
      console.log(`Loaded batch of ${events.length}. Response:`, result);
    }

    main().catch(console.error);
    ```
6.  Update `subgraph-dgraph-gharchive/compose.yaml` to use different host ports (e.g., `8082`, `8002`, etc.) to avoid conflicts.
7.  Update the `subgraph-dgraph-gharchive/Makefile` to manage the loader script:
    ```makefile
    # ... (up, down, clean targets) ...
    apply-schema:
        curl -X POST localhost:8082/admin/schema --data-binary "@schema.graphql"

    install-loader:
        cd loader && npm install

    load-data: install-loader
        @echo "Loading GHArchive data for a sample hour..."
        node loader/index.js
    ```

**Task 3.2: Integrate `subgraph-dgraph-gharchive` into `supergraph-cncf`**

1.  Add the new Spoke to `graphtastic.deps.yml`.
2.  Update `mesh.config.js` to add `gharchive` as a source, pointing to the Dgraph Alpha service on its internal port `8080`.
3.  Re-render the supergraph (`make supergraph`) and restart the system (`make up`).
4.  Verify with a query against `http://localhost:4000/graphql` that filters for `PushEvent` data.

---

#### **Phase 4: Advanced Composite Spoke (ETL Pipeline)**

**Goal:** Implement the complex `software-supply-chain` Spoke, which is itself a composite system involving an external data source (GUAC) and a sophisticated ETL process.

**Task 4.1: Create `subgraph-dgraph-software-supply-chain` Spoke**

1.  Create the `subgraph-dgraph-software-supply-chain` repository from the `subgraph-dgraph-static` template and clone it.
2.  Update the Spoke's `compose.yaml`. For isolated development, it will run its own instance of GUAC.
    ```yaml
    # In subgraph-dgraph-software-supply-chain/compose.yaml
    services:
      # ... (Dgraph Zero, Alpha, Ratel services on non-conflicting ports, e.g. 8083)
      
      # GUAC services for local development
      guac-postgres:
        image: postgres:13
        # ... env vars, volumes ...
      guac-nats:
        image: nats:2.9
      # ... other GUAC collector/assembler services ...
      guac-graphql:
        image: guacsec/guac-graphql:latest # Use official GUAC image
        ports: ["8085:8080"] # Expose GUAC GraphQL API for the ETL script
    ```
3.  Create the ETL script directory: `mkdir subgraph-dgraph-software-supply-chain/etl`
4.  Implement the ETL script (`etl/index.js`) and schema augmentation logic as detailed in `design--guac-to-dgraph.md`. This script will be significantly more complex, performing a two-stage extraction (Nodes, then Edges) from `http://guac-graphql:8080` and writing to an RDF file.
5.  Update the Spoke's `Makefile` to orchestrate the entire ETL process:
    ```makefile
    # ... (up, down, clean) ...
    
    etl:
        @echo "Running GUAC to Dgraph ETL process..."
        # 1. Run the extractor script to query GUAC and generate guac-data.rdf.gz
        node etl/index.js --source-url http://localhost:8085 --output-file ./data/guac-data.rdf.gz

        # 2. Use Dgraph Live Loader to ingest the RDF file
        docker compose exec alpha dgraph live --files /dgraph/guac-data.rdf.gz --alpha alpha:7080 --zero zero:5080
    ```

**Task 4.2: Integrate into `supergraph-cncf`**

1.  Add the Spoke to `graphtastic.deps.yml`.
2.  Update `mesh.config.js`, adding a source for the software supply chain data.
3.  Re-render the supergraph (`make supergraph`) and restart (`make up`).
4.  Verify with a complex query joining data across multiple subgraphs.

---

#### **Phase 5: Production Hardening and Polish**

**Goal:** Implement the remaining production-readiness features from the Tome across the entire platform.

**Task 5.1: Supergraph CI/CD Hardening**

1.  Create the CI workflow file at `supergraph-cncf/.github/workflows/ci.yml`:
    ```yaml
    name: Supergraph CI
    on: [pull_request]
    jobs:
      validate:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4
          - name: Setup Node
            uses: actions/setup-node@v4
            with: { node-version: '18' }
          - name: Install Dependencies
            run: npm install -g @graphql-mesh/compose-cli yq
          - name: Sync Deps & Render Supergraph
            run: make supergraph
          - name: Check for Stale Supergraph Artifact
            run: |
              git diff --exit-code supergraph.graphql
              echo "✅ supergraph.graphql is up-to-date."
    ```

**Task 5.2: Secret Management Implementation**

1.  Modify `federated-graph-core/compose.yaml` to use an environment variable for a database password.
    ```yaml
    services:
      postgres:
        image: postgres:15
        environment:
          POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-hive} # Default to 'hive' if not set
    ```
2.  Create `.env.template` in the `federated-graph-core` repository:
    ```
    # Secrets for the federated-graph-core stack
    # Copy this file to .env and fill in your values for local development.
    POSTGRES_PASSWORD=
    ```

**Task 5.3: Configurable Persistence Implementation**

1.  Modify `subgraph-dgraph-static/compose.yaml` to support configurable persistence.
    ```yaml
    # ...
    services:
      alpha:
        volumes:
          - type: ${PERSISTENCE_MODE:-bind}
            source: ./data/alpha # Note: PERSISTENCE_PATH is not needed if relative
            target: /dgraph
    # ...
    ```
2.  The `manage-stacks.sh` script and `Makefile.master` from **Phase 0** are already designed to pass the `PERSISTENCE_MODE` variable, so no further changes are needed in the orchestration layer. A developer can now run `make up mode=volume` to switch to named volumes.