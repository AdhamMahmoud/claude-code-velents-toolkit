# llms.txt for Redis Documentation
# Description: List of Redis documentation markdown files for LLMs and AI assistants to ingest and cite.

## Core Docs

- [Docs Homepage](https://redis.io/docs/latest/) Access Redis documentation for developers and operators, covering setup, usage, commands, integrations, and advanced features.
- [APIs](https://redis.io/docs/latest/apis/index.html.md) An overview of Redis APIs for developers and operators
- [Quick starts](https://redis.io/docs/latest/develop/get-started/index.html.md) Get started with Redis development using this guide covering setup, basic commands, and working with key data types.

## Commands Reference

- [Commands](https://redis.io/docs/latest/commands/) Browse the full list of Redis commands by category, with syntax, examples, and detailed usage for each command.
- [AUTH](https://redis.io/docs/latest/commands/auth/index.html.md) Details on the AUTH command for authenticating Redis clients with passwords or ACLs to secure access.
- [DEL](https://redis.io/docs/latest/commands/del/index.html.md) Learn how the DEL command removes one or more keys from Redis, freeing up memory immediately.
- [EXPIRE](https://redis.io/docs/latest/commands/expire/index.html.md) The EXPIRE command sets a timeout on a key, allowing automatic deletion after a specified number of seconds.
- [GET](https://redis.io/docs/latest/commands/get/index.html.md) The GET command retrieves the value of a key stored as a string in Redis, returning nil if the key doesn't exist.
- [HSET](https://redis.io/docs/latest/commands/hset/index.html.md) The HSET command sets one or more field-value pairs in a Redis hash, creating the hash if it doesn’t exist.
- [SET](https://redis.io/docs/latest/commands/set/index.html.md) The SET command assigns a string value to a key, with optional expiration and conditional set options.
- [TTL](https://redis.io/docs/latest/commands/ttl/index.html.md) The TTL command returns the remaining time to live of a key with an expiration, in seconds.

## Development

- [Develop with Redis](https://redis.io/docs/latest/develop/index.html.md) Learn how to develop with Redis.
- [Connect with Redis client API libraries](https://redis.io/docs/latest/develop/clients/index.html.md) Find official and community Redis clients for various programming languages, with setup guides and usage examples.
- [Understand Redis data types](https://redis.io/docs/latest/develop/data-types/index.html.md) Explore Redis data types like strings, hashes, lists, sets, and more, with usage patterns and best practices.
- [Redis Query Engine](https://redis.io/docs/latest/develop/interact/search-and-query/index.html.md) Learn how to use Redis for search and query operations with full-text, secondary indexes, and filtering capabilities.
- [Vectors](https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/vectors/index.html.md) Understand how to store, index, and query vector data in Redis for AI, similarity search, and machine learning use cases.
- [Indexing](https://redis.io/docs/latest/develop/interact/search-and-query/indexing/index.html.md) Learn how to create and manage indexes in Redis to enable efficient querying, filtering, and full-text search.
- [Key eviction](https://redis.io/docs/latest/develop/reference/eviction/index.html.md) Understand Redis eviction policies and how keys are removed when memory limits are reached.
- [Redis CLI](https://redis.io/docs/latest/develop/tools/cli/index.html.md) Use the Redis CLI to interact with your database, run commands, and troubleshoot with built-in tools.
- [Redis Insight release notes](https://redis.io/docs/latest/develop/tools/insight/release-notes/index.html.md) Stay up to date with the latest Redis Insight releases, including new features, improvements, and bug fixes.
- [Redis for VS Code](https://redis.io/docs/latest/develop/tools/redis-for-vscode/index.html.md) Learn how to use the Redis extension for VS Code to visualize data, run commands, and manage your Redis databases.
- [Distributed Locks with Redis](https://redis.io/docs/latest/develop/use/patterns/distributed-locks/index.html.md) Implement distributed locks with Redis to manage concurrent access and ensure consistency across systems.

## Integrations

- [Libraries and tools that integrate with Redis](https://redis.io/docs/latest/integrate/index.html.md) Explore ways to integrate Redis with popular languages, frameworks, and services for building high-performance applications.
- [Prometheus metrics v2 preview](https://redis.io/docs/latest/integrate/prometheus-with-redis-enterprise/prometheus-metrics-definitions/index.html.md) View Prometheus metrics definitions for Redis Enterprise, including metric names, types, and usage guidance.
- [Redis Data Integration](https://redis.io/docs/latest/integrate/redis-data-integration/index.html.md) Connect Redis with external data sources using Redis Data Integration (RDI) to stream, transform, and sync data in real time.
- [Python client for Redis](https://redis.io/docs/latest/integrate/redis-py/index.html.md) Learn how to use redis-py, the official Python client for Redis, with setup instructions and usage examples.
- [RedisVL](https://redis.io/docs/latest/integrate/redisvl/index.html.md) Explore RedisVL, a Python library for storing and querying vector embeddings in Redis with model integration support.
- [RIOT](https://redis.io/docs/latest/integrate/riot/index.html.md) Use RIOT to move and transform data between Redis and other systems with a customizable, event-driven framework.
- [Create a Redis Cloud database with the Vercel integration](https://redis.io/docs/latest/operate/rc/cloud-integrations/vercel/index.html.md) Integrate Redis Cloud with Vercel to enable fast, scalable data access in serverless applications.

## Operations

- [Redis products](https://redis.io/docs/latest/operate/index.html.md) Find operational guides for deploying, configuring, securing, and maintaining Redis across cloud and software deployments.
- [Redis Enterprise for Kubernetes](https://redis.io/docs/latest/operate/kubernetes/index.html.md) Redis Enterprise Operator allows you to deploy and manage Redis Enterprise clusters on Kubernetes with automation and scalability.
- [Install Redis](https://redis.io/docs/latest/operate/oss_and_stack/install/install-stack/index.html.md) Install Redis on Linux, macOS, and Windows
- [Redis configuration](https://redis.io/docs/latest/operate/oss_and_stack/management/config/index.html.md) Overview of redis.conf, the Redis configuration file
- [Redis persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/index.html.md) Learn about Redis persistence options, including RDB snapshots and AOF logs, to ensure data durability and recovery.
- [Redis Cloud](https://redis.io/docs/latest/operate/rc/index.html.md) Operate Redis Cloud with guides on provisioning, managing databases, backups, metrics, and user access.
- [Back up and export a database](https://redis.io/docs/latest/operate/rc/databases/back-up-data/index.html.md) Learn how to back up your Redis Cloud database, including manual backups, schedules, and data retention options.
- [Redis Cloud quick start](https://redis.io/docs/latest/operate/rc/rc-quickstart/index.html.md) Get started with Redis Cloud in minutes using this quickstart guide for setup, provisioning, and basic CLI interaction.
- [Install Redis Insight](https://redis.io/docs/latest/operate/redisinsight/install/index.html.md) Install RedisInsight on your preferred platform to visualize, monitor, and optimize your Redis databases.
- [Redis Enterprise Software](https://redis.io/docs/latest/operate/rs/index.html.md) Operate Redis Software with documentation on installation, upgrades, clustering, CLI tools, and monitoring.
- [Connect to a database](https://redis.io/docs/latest/operate/rs/databases/connect/index.html.md) Learn how to connect your application to a Redis database hosted by Redis Enterprise Software and test your connection.
- [Test client connection](https://redis.io/docs/latest/operate/rs/databases/connect/test-client-connectivity/index.html.md) Learn how to test Redis database connectivity using built-in tools and verify client access across different deployment types.
- [Hardware requirements](https://redis.io/docs/latest/operate/rs/installing-upgrading/install/plan-deployment/hardware-requirements/index.html.md) Review hardware requirements for deploying Redis Software, including CPU, memory, storage, and networking guidelines.
- [Redis Enterprise Software product lifecycle](https://redis.io/docs/latest/operate/rs/installing-upgrading/product-lifecycle/index.html.md) Understand the Redis Software product lifecycle, including support phases, version timelines, and upgrade recommendations.
- [Network port configurations](https://redis.io/docs/latest/operate/rs/networking/port-configurations/index.html.md) Understand Redis Software port configurations for client communication, clustering, metrics, and administrative access.
- [redis-cli](https://redis.io/docs/latest/operate/rs/references/cli-utilities/redis-cli/index.html.md) Learn how to use redis-cli, the Redis command-line tool for interacting with Redis databases and running commands directly.
- [REST API](https://redis.io/docs/latest/operate/rs/references/rest-api/index.html.md) Access and manage Redis Software programmatically using the REST API for automation, monitoring, and integration.
- [Release notes](https://redis.io/docs/latest/operate/rs/release-notes/index.html.md) Review Redis Software release notes for updates on new features, enhancements, bug fixes, and version compatibility.