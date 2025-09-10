# A Git-Centric Approach to GraphQL Federation: Registry-Less Validation with Hive Gateway

### Executive Summary

This document outlines a sophisticated, Git-centric strategy for managing and validating a federated GraphQL architecture. This approach, termed "dehydrated deployment," treats the entire GraphQL Hive Gateway configuration and schema composition process as ephemeral, stateless operations perfectly suited for a CI/CD pipeline. The core principle is to maintain all configuration and subgraph schemas as declarative artifacts within a Git repository, which serves as the single source of truth.

In this model, a running GraphQL Hive instance is not a prerequisite for validating changes. Instead, the supergraph is composed on-the-fly within the CI pipeline by fetching subgraph schemas directly from their raw file URLs in the Git repository. The Hive Gateway is then spun up ephemerally in a Docker container to load this newly composed supergraph, providing a definitive validation of its integrity. This workflow enables teams to iterate on subgraphs, supergraphs, and gateway configurations with confidence, ensuring that only composable schemas are merged.

While this approach deliberately bypasses the Schema Registry for the initial CI validation step to achieve speed and infrastructure independence, it is best viewed as a powerful complement to the registry's capabilities. The registry remains indispensable for production environments, offering schema history, usage-based breaking change detection, and a high-availability CDN. This guide presents a hybrid framework where the dehydrated, registry-less approach is used for pull request validation, while the traditional registry-based workflow is used for deployment to persistent environments like staging and production.

---

## Table of Contents

*   [I. The Philosophy: Dehydrating the Stack for CI](#i-the-philosophy-dehydrating-the-stack-for-ci)
*   [II. The Toolkit for On-Demand Supergraph Composition](#ii-the-toolkit-for-on-demand-supergraph-composition)
    *   [2.1 GraphQL Mesh Compose CLI: The Composition Engine](#21-graphql-mesh-compose-cli-the-composition-engine)
    *   [2.2 Hive Gateway: The Validation Runtime](#22-hive-gateway-the-validation-runtime)
*   [III. A Practical CI Workflow with Docker and GitHub Actions](#iii-a-practical-ci-workflow-with-docker-and-github-actions)
    *   [3.1 Repository Structure](#31-repository-structure)
    *   [3.2 The GitHub Actions Workflow](#32-the-github-actions-workflow)
*   [IV. The Role of the Schema Registry: From CI Insights to Production Governance](#iv-the-role-of-the-schema-registry-from-ci-insights-to-production-governance)
    *   [4.1 Advanced CI Scenarios: Coverage, Performance, and Load Testing](#41-advanced-ci-scenarios-coverage-performance-and-load-testing)
    *   [4.2 Production Hydration: Governance and High Availability](#42-production-hydration-governance-and-high-availability)
*   [Works cited](#works-cited)

---

## I. The Philosophy: Dehydrating the Stack for CI

The traditional workflow for GraphQL Federation involves a persistent, stateful Schema Registry that acts as the central hub for all schema-related operations. While this is the gold standard for production, it introduces an external dependency into the CI/CD process. The "dehydrated" approach inverts this model for validation purposes, with the following core tenets:

*   **Git as the Single Source of Truth:** Every aspect of the federated graph—subgraph schemas, gateway configuration, and even the composition logic—is defined in version-controlled files. There is no "hidden" state residing in an external service during the validation phase.
*   **Ephemeral Infrastructure:** The CI pipeline dynamically provisions the necessary tools, specifically the composition engine and the Hive Gateway, within Docker containers. These components exist only for the duration of the CI job and are then discarded.
*   **Registry-Less Composition:** Instead of publishing schemas to a registry and waiting for it to compose them, the CI job performs the composition itself. It fetches the required subgraph schemas directly from their source in the Git repository, typically via raw content URLs.
*   **Validation Through Execution:** The ultimate test of a valid supergraph is whether a gateway can successfully load and serve it. By spinning up the Hive Gateway with the composed schema, the CI pipeline provides a high-fidelity confirmation that the configuration is sound.

This methodology provides fast, self-contained, and reproducible validation for every pull request, ensuring that the main branch always contains a valid and composable supergraph.

---

## II. The Toolkit for On-Demand Supergraph Composition

This workflow relies on two key components from the GraphQL Hive ecosystem, used here in a CI-specific context: the GraphQL Mesh Compose CLI for on-the-fly composition, and the Hive Gateway for final validation.

### 2.1 GraphQL Mesh Compose CLI: The Composition Engine

GraphQL Mesh provides a powerful standalone composition tool, the `mesh-compose` CLI, that is perfectly suited for this registry-less workflow. Its function is to take a set of subgraph definitions from various sources and attempt to compose them into a single, valid supergraph SDL file.

Crucially, `mesh-compose` can be configured to load subgraph schemas directly from remote URLs. This allows the CI job to point directly to the raw schema files (`schema.graphql`) within the pull request branch on GitHub or any other Git provider.

A complete configuration file (`mesh.config.ts`) for this process would look like the following full example:

```typescript
// mesh.config.ts
import { defineConfig } from '@graphql-mesh/compose-cli';
import { loadGraphQLHTTPSubgraph } from '@graphql-mesh/graphql';

// In a real CI job, these URLs would be dynamically generated to point to the
// specific commit/branch of the pull request. For example:
// const commitSha = process.env.GITHUB_SHA;
// const repo = process.env.GITHUB_REPOSITORY;
// const usersSubgraphUrl = `https://raw.githubusercontent.com/${repo}/${commitSha}/subgraphs/users/schema.graphql`;
// const productsSubgraphUrl = `https://raw.githubusercontent.com/${repo}/${commitSha}/subgraphs/products/schema.graphql`;

// For this example, we use static URLs.
const usersSubgraphUrl = 'https://raw.githubusercontent.com/my-org/my-repo/my-feature-branch/subgraphs/users/schema.graphql';
const productsSubgraphUrl = 'https://raw.githubusercontent.com/my-org/my-repo/my-feature-branch/subgraphs/products/schema.graphql';

export const composeConfig = defineConfig({
  subgraphs: [
    loadGraphQLHTTPSubgraph('users', {
      endpoint: usersSubgraphUrl,
    }),
    loadGraphQLHTTPSubgraph('products', {
      endpoint: productsSubgraphUrl,
    }),
  ],
});
```

Within the CI pipeline, executing `npx mesh-compose -o supergraph.graphql` will use this configuration to fetch the schemas and generate the `supergraph.graphql` file. If composition fails due to federation errors, the command will exit with a non-zero status code, automatically failing the CI check.

### 2.2 Hive Gateway: The Validation Runtime

Once the `supergraph.graphql` file has been successfully generated, the Hive Gateway is used to confirm that it is a valid and loadable schema. For this CI workflow, the gateway is not configured to connect to a Hive CDN. Instead, it is configured to load its schema directly from the local filesystem.

The `gateway.config.ts` file for this ephemeral validation step is simple, declarative, and complete:

```typescript
// gateway.config.ts
import { defineConfig } from '@graphql-hive/gateway';

export const gatewayConfig = defineConfig({
  // Instruct the gateway to load its supergraph from a local file.
  // This file will be the one generated by mesh-compose in the previous CI step.
  supergraph: './supergraph.graphql',
    
  // Disable the landing page for a cleaner startup in CI.
  landingPage: false
});
```

This configuration, along with a simple Dockerfile, allows the CI job to build and run a Hive Gateway instance that serves the exact supergraph composed from the pull request's code.

---

## III. A Practical CI Workflow with Docker and GitHub Actions

This section outlines a concrete implementation of the dehydrated validation workflow using Docker and GitHub Actions.

### 3.1 Repository Structure

A recommended repository structure for this approach would be:

```
.
├── .github/
│   └── workflows/
│       └── validate-composition.yml  # The CI workflow definition
├── gateway/
│   ├── Dockerfile
│   └── gateway.config.ts             # Gateway config to load local supergraph
├── subgraphs/
│   ├── products/
│   │   └── schema.graphql
│   └── users/
│       └── schema.graphql
├── mesh.config.ts                    # Mesh Compose config for CI
└── package.json
```

### 3.2 The GitHub Actions Workflow

The following workflow demonstrates the end-to-end validation process. It is triggered on every pull request targeting the `main` branch.

```yaml
# .github/workflows/validate-composition.yml
name: 'Validate Supergraph Composition'

on:
  pull_request:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install Dependencies
        run: npm install

      - name: Compose Supergraph from Git Sources
        id: compose
        run: |
          # This command attempts to compose the supergraph.
          # If it fails, the workflow will stop here.
          # Note: The mesh.config.ts would need logic to dynamically
          # build the raw GitHub URLs based on the current branch/commit SHA.
          npx mesh-compose -o ./gateway/supergraph.graphql

      - name: Build Gateway Docker Image
        run: docker build -t temp-gateway ./gateway

      - name: Run Ephemeral Gateway for Validation
        run: |
          # Run the gateway in the background
          docker run -d --rm --name validation-gateway -p 4000:4000 temp-gateway

          # Give the server a moment to start
          sleep 5

          # Perform a readiness check. If the gateway fails to start with the
          # composed schema, this curl command will fail, causing the step to fail.
          # The /readiness endpoint confirms the gateway is ready to serve traffic.
          curl --fail http://localhost:4000/readiness

          echo "✅ Gateway started successfully with the composed supergraph."

      - name: Cleanup Ephemeral Gateway
        if: always()
        run: docker stop validation-gateway
```

This workflow provides two distinct validation gates:

1.  **Composition Check:** The `mesh-compose` command ensures that the subgraph schemas are compatible and can be federated.
2.  **Runtime Check:** The ephemeral Hive Gateway confirms that the composed supergraph is not just syntactically correct, but also operationally sound and can be served successfully.

**TODO: add diagram**

---

## IV. The Role of the Schema Registry: From CI Insights to Production Governance

This dehydrated CI workflow is a powerful tool for pull request validation, but it intentionally omits the features that make a Schema Registry essential for managing a production graph. The registry is not just a composition engine; it is a system of record that provides critical governance and operational capabilities, including detailed assessments of what elements of the schema are used over time ("code coverage for the composed/supergraph schema). This informs detemining the potential impact of a change to a subgraph's schema on the composed supergraph, including identifying which clients (who's queries are served by the Gateway) would be impacted and/or fail. [1]

### 4.1 Advanced CI Scenarios: Coverage, Performance, and Load Testing

While the primary CI check should be fast and registry-less, the Schema Registry can be invaluable for more advanced, optional CI workflows. After the initial composition check passes, a subsequent job can be triggered to perform deeper analysis:

1.  **Spin up an Ephemeral Registry:** The CI job uses Docker Compose to launch a self-hosted GraphQL Hive instance, including the registry and its database dependencies.
2.  **Publish the Schema:** The job uses the Hive CLI (`hive schema:publish`) to publish the subgraph schemas from the pull request branch to a temporary, ephemeral target within the local registry.
3.  **Run Test Suites:** With the ephemeral gateway now pointing to the local registry, automated test suites can be executed. These tests can include:
    *   **Integration and E2E Tests:** As these tests run, the Hive client reports schema usage to the ephemeral registry.
    *   **Performance Baselines:** A standard set of benchmark queries can be run to establish performance baselines and detect potential regressions introduced by the schema change.
    *   **Load Testing:** A controlled load test can be executed against the ephemeral stack to understand the performance impact of the changes under stress.
4.  **Analyze Results:** After the tests complete, scripts can query the ephemeral registry's API to gather metrics on schema coverage (which parts of the graph were used by the tests), performance data, and error rates, providing quantitative feedback directly in the pull request.

This approach allows you to quantify the impact of a change before it ever reaches a persistent environment, catching performance regressions and ensuring adequate test coverage as part of the automated pipeline.

### 4.2 Production Hydration: Governance and High Availability

After a pull request is merged, a separate Continuous Deployment (CD) workflow is triggered. This workflow "rehydrates" the system by using the **Hive CLI** to execute `hive schema:publish` for any modified subgraphs. [2] This action:

*   Pushes the new, validated subgraph schema to the persistent GraphQL Hive Schema Registry. [2]
*   Creates an immutable, versioned record of the schema change.
*   Allows the registry to perform its own composition and run checks against historical **production usage data** to detect potential breaking changes that composition validation alone cannot catch. [1]
*   Updates the supergraph artifact on the high-availability CDN.

Production and staging instances of the Hive Gateway should be configured to point to the Hive CDN, not a local file. This decouples deployment of the gateway from deployment of the schema and ensures that gateways are always serving a stable, registry-approved supergraph version.

By combining these two approaches, you get the best of both worlds: the speed and isolation of a dehydrated, Git-centric validation process during development, and the safety, governance, and resilience of a fully-managed Schema Registry for your production environments.

#### Referenced Materials

1.  Schema Registry | Hive - GraphQL (The Guild), accessed September 2, 2025, [https://the-guild.dev/graphql/hive/docs/schema-registry](https://the-guild.dev/graphql/hive/docs/schema-registry)
2.  CI/CD and Hive CLI - GraphQL (The Guild), accessed September 2, 2025, [https://the-guild.dev/graphql/hive/docs/other-integrations/ci-cd](https://the-guild.dev/graphql/hive/docs/other-integrations/ci-cd)
