# Reference deployment topology

These files document how AsiFin was deployed. They are reference material, not a
runnable build: the `api` and `frontend` services build from an application source
tree that is not published in this repository.

What the Compose file records:

| Service | Image / role |
|---|---|
| `db` | `postgres:16-alpine`, with healthcheck and named volume |
| `redis` | `redis:7-alpine`, append-only persistence |
| `qdrant` | `qdrant/qdrant:latest`, vector store for document chunks |
| `ollama` | `ollama/ollama:latest`, local inference for text and vision models |
| `api` | FastAPI backend, builds from the unpublished source tree |
| `frontend` | React + Vite, builds from the unpublished source tree |

The infrastructure services are unmodified upstream images and their configuration
here is complete. Only the two application services are unbuildable.
