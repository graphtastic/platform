# Implementation Plan: 01 - Foundation and Stateless Spoke

**Goal:** Establish the core orchestration tooling and create the first, standalone, stateless Spoke (`subgraph-blogs`).

This plan is designed to be executed sequentially. It assumes you have cloned all necessary repositories and are executing the commands from within the correct repository's root directory.

## Task Checklist

- [x] **Phase 0: The Developer Control Plane**
    - [x] [Task 0.1: Create `tools-docker-compose` Repository Content](#task-01-create-tools-docker-compose-repository-content)
    - [x] [Task 0.2: Create `tools-subgraph-core` Repository Content](#task-02-create-tools-subgraph-core-repository-content)
- [ ] **Phase 1: Minimal Viable Federation with Stateless Spokes**
    - [ ] [Task 1.1: Create `subgraph-blogs`](#task-11-create-subgraph-blogs)

---

### Phase 0: The Developer Control Plane

#### Task 0.1: Create `tools-docker-compose` Repository Content

1.  From within the `tools-docker-compose` repository, create the scripts subdirectory: `mkdir -p scripts`
2.  Create the master `Makefile` at `Makefile.master`:

        ```makefile
        # This is the master Makefile, located in `tools-docker-compose`.
        # It is called by a Hub's lightweight, root Makefile.

        # Default variables
        DEPS_DIR := ./.deps
        MANIFEST := ./graphtastic.deps.yml
        SUPERGRAPH_SDL := ./supergraph.graphql
        SHARED_NETWORK_NAME ?= $(shell grep SHARED_NETWORK_NAME .env | cut -d '=' -f2)

        # This allows passing a stack name, e.g., `make logs stack=subgraph-blogs`
        stack ?= all

        # Allow passing persistence mode, e.g., `make up mode=volume`
        mode ?= bind

        .PHONY: help setup up down clean restart deps supergraph validate ps logs seed

        help:
            @echo "Graphtastic Platform - Master Orchestrator"
            @echo "  logs               - Tail the logs of services. Use 'stack=' to target one."

        setup:
            @docker network inspect $(SHARED_NETWORK_NAME) >/dev/null 2>&1 || \
                docker network create $(SHARED_NETWORK_NAME)

        up: setup deps seed
            @echo "🚀 Bringing up all services with persistence mode: $(mode)..."
            ./$(DEPS_DIR)/tools-docker-compose/scripts/manage-stacks.sh up $(MANIFEST) $(mode)

        down:
            @echo "🔥 Bringing down all services..."
            ./$(DEPS_DIR)/tools-docker-compose/scripts/manage-stacks.sh down $(MANIFEST)

        clean:
            @read -p "This will stop all services and permanently remove all local data for this supergraph. Are you sure? (y/N) " confirm && [ $${confirm:-N} = y ] || exit 1
            @docker network rm $(SHARED_NETWORK_NAME) 2>/dev/null || true
        ```

3.  Create the dependency sync script at `scripts/sync-deps.sh`:

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
        ```

4.  Create the stack management script at `scripts/manage-stacks.sh`:

        ```bash
        #!/bin/bash
        set -e
        ACTION=$1
        MANIFEST=$2
        PERSISTENCE_MODE=${3:-bind}
        STACK=${4:-all}
        SUPERGRAPH_NAME=$(yq e '.name' graphtastic.deps.yml)

        # This script now iterates for all actions to ensure project isolation.
        SPOKES=$(yq e '.components[] | select(.type == "spoke") | .name' $MANIFEST)

        for name in $SPOKES; do
            if [ "$STACK" = "all" ] || [ "$STACK" = "$name" ]; then
                # Each spoke is launched as a separate Docker Compose project for isolation
                COMPOSE_FILE="./.deps/$name/compose.yaml"
                if [ -f "$COMPOSE_FILE" ]; then
                    PROJECT_NAME="${SUPERGRAPH_NAME}_$name"
                    case "$ACTION" in
                        up)
                            echo "Bringing up $name as project $PROJECT_NAME..."
                            PERSISTENCE_MODE=$PERSISTENCE_MODE docker compose -p $PROJECT_NAME -f $COMPOSE_FILE up -d
                            ;;
                        down)
                            echo "Bringing down $name as project $PROJECT_NAME..."
                            PERSISTENCE_MODE=$PERSISTENCE_MODE docker compose -p $PROJECT_NAME -f $COMPOSE_FILE down
                            ;;
                        ps|logs)
                            echo "Showing $ACTION for $name (project $PROJECT_NAME)..."
                            PERSISTENCE_MODE=$PERSISTENCE_MODE docker compose -p $PROJECT_NAME -f $COMPOSE_FILE $ACTION
                            ;;
                        seed)
                            SPOKE_MAKEFILE="./.deps/$name/Makefile"
                            if [ -f "$SPOKE_MAKEFILE" ] && grep -q "^seed:" "$SPOKE_MAKEFILE"; then
                                echo "--- Seeding data for $name ---"
                                (cd "./.deps/$name" && make seed)
                            fi
                            ;;
                        *)
                            echo "Unknown action: $ACTION"
                            exit 1
                            ;;
                    esac
                fi
            fi
        done
        ```

5.  Make scripts executable: `chmod +x scripts/*.sh`

#### Task 0.2: Create `tools-subgraph-core` Repository Content

1.  From within the `tools-subgraph-core` repository, create the workflows subdirectory: `mkdir -p .github/workflows`
2.  Create a master Makefile for subgraphs at `Makefile.subgraph.master`:

        ```makefile
        # This is the master Makefile for subgraphs.
        # It is included by a Spoke's lightweight, root Makefile.
        .PHONY: validate-federation

        validate-federation:
                @echo "🔎 Validating schema for federation readiness..."
                # In a real implementation, this would use GraphQL Inspector
                @echo "✅ Schema is valid (placeholder)."
        ```

3.  Create the reusable CI workflow at `.github/workflows/subgraph-ci.yml`:

        ```yaml
        name: Subgraph CI
        on:
            workflow_call:
        jobs:
            validate:
                runs-on: ubuntu-latest
                steps:
                    - uses: actions/checkout@v4
                    - name: Validate Federation Readiness
                        run: make validate-federation # This command is provided by the included Makefile
        ```

### Phase 1: Minimal Viable Federation with Stateless Spokes

#### Task 1.1: Create `subgraph-blogs`

1.  From within the `subgraph-blogs` repository, create the Yoga server at `index.js`:

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
                author: Author
            }

            # We are extending the Author type, which is owned by the authors subgraph.
            # The @key directive tells the gateway how to fetch an Author.
            # The @extends directive tells the gateway this type is defined elsewhere.
            type Author @key(fields: "id") @extends {
                id: ID! @external
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
            Blog: {
                // This resolver provides the "link" to the Author type.
                // It returns a representation containing the key field ('id').
                // The gateway will use this to fetch the full Author from the other subgraph.
                author: (blog) => {
                    return { __typename: "Author", id: blog.authorId };
                }
            }
        };

        const yoga = createYoga({ schema: createSchema({ typeDefs, resolvers }) });
        const server = createServer(yoga);
        const PORT = process.env.PORT || 4001;
        server.listen(PORT, () => { console.info(`Blogs subgraph running at http://localhost:${PORT}/graphql`) });
        ```

2.  Create `schema.graphql` with the schema from the `typeDefs` above.

        ```graphql
            type Query {
                blogs: [Blog!]
                blog(id: ID!): Blog
            }

            type Blog @key(fields: "id") {
                id: ID!
                title: String!
                author: Author
            }

            type Author @key(fields: "id") @extends {
                id: ID! @external
            }
        ```

3.  Create a `package.json`:

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

4.  Create a `compose.yaml`:

        ```yaml
        version: "3.8"
        services:
            blogs-service:
                build: .
                ports: ["${SUBGRAPH_BLOGS_PORT:-4001}:4001"]
                networks: [graphtastic_net]
        networks:
            graphtastic_net:
                name: ${SHARED_NETWORK_NAME:-graphtastic_net}
                external: true
        ```

5.  Create a `Dockerfile`:

        ```dockerfile
        FROM node:18-alpine
        WORKDIR /app
        COPY package.json .
        RUN npm install
        COPY . .
        CMD ["node", "index.js"]
        ```
