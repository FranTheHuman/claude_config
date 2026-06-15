# /quine:deploy — Deployment, Recipe, and Persistor Scaffolding

Generate deployment configuration for Quine: Docker Compose service definition,
recipe YAML for development, persistor selection, resource sizing, and health
checks consistent with this stack's existing VPS conventions.
Apply knowledge from `.claude/skills/quine-architecture/SKILL.md`.

## Instructions

1. Read `.claude/skills/quine-architecture/SKILL.md` sections 11, 12, and 13
   (Deployment, Monitoring, Recipes) before generating.
2. Default to RocksDB persistor on the existing VPS unless the user states an
   HA/scale requirement — if they ask for Cassandra without stating such a
   requirement, ask what's driving that choice before generating a multi-service
   Cassandra setup (hard rule: don't add Cassandra by default).
3. For Docker Compose: follow the conventions of this stack's existing
   `docker-compose.yml` (loopback-bound ports unless public exposure is explicitly
   needed, memory limits, json-file logging with rotation, healthcheck blocks).
4. Health checks use `/api/v2/system/readiness` and `/api/v2/system/liveness`.
5. For recipes: full YAML following the standard structure (skill section 13),
   with a clear note that recipes are for development — production should use
   named, persistent API-driven ingest/standing query configuration.
6. Include resource-sizing guidance (`-Xmx`, `in-memory-soft-node-limit`) scaled
   to what the user describes about their data volume — don't give a single
   generic number without context.

## Output Structure

### Persistor Decision
RocksDB / Cassandra / MapDB — with the one-sentence reason.

### Docker Compose Service

```yaml
# quine service block — consistent with existing VPS compose conventions:
# loopback ports unless told otherwise, memory limit, logging, healthcheck
```

### Resource Sizing

```bash
# JVM flags and in-memory node limits, with reasoning for the chosen values
```

### Development Recipe (if requested)

```yaml
# Full recipe YAML — version, title, ingestStreams, standingQueries, etc.
```

### Production Notes
- How this differs for production (named API-driven config vs recipe)
- Health check integration
- What to monitor (dashboard panels, REST metrics endpoint, backpressure metric)

## Arguments

Usage: `/quine:deploy [dev|prod] [details about data volume / HA needs]`

Examples:
- `/quine:deploy dev` → recipe-based local setup, RocksDB, single instance
- `/quine:deploy prod low volume, same VPS as the rest of the stack`
- `/quine:deploy prod high availability required, multi-region`
- `/quine:deploy resource sizing for ~10k events/sec ingest`
