# Port remaps for the 10.73.2.11 deploy host

The base `docker-compose.yml` host ports were remapped only on that server
(not in the shared compose file), because existing services already used
3306, 7860, and 9200/9300. Intra-network names and container ports are unchanged.
Do not change the published ports in the shared `docker/docker-compose.yml`.

| Service | Host port on 10.73.2.11 | Container port | Shared compose default |
| --- | --- | --- | --- |
| MySQL | 13306 | 3306 | 3306 |
| Backend | 17860 | 7860 | 7860 |
| Elasticsearch | 19200 / 19300 | 9200 / 9300 | 9200 / 9300 |
| Frontend | 3001 (unchanged) | 3001 | 3001 |
| Redis | 6379 (unchanged) | 6379 | 6379 |
| MinIO | 9100 (unchanged) | 9000 | 9100 |
| Milvus | 19530 / 9091 (unchanged) | 19530 / 9091 | 19530 / 9091 |

`dsh.platform_public_url` on that host is `http://10.73.2.11:3001`.
The repo default in `docker/bisheng/config/config.yaml` stays
`http://127.0.0.1:3001`. Put a deploy host IP only in that machine's copy
of the config, not in the committed file.

## Override files

Do not copy the Windows workstation override onto the server. The sample in
`docker-compose.override.example.yml` remaps Redis to 16379 and uses
`ports: !override`, which Compose v2.21 on the server does not support.
`ports` concatenates across compose files by default, so `!override` is
required on Compose versions that support it, to replace the base mapping
instead of adding another published port.

On Compose v2.21, patch the published ports in the host's copy of
`docker-compose.yml` instead of using an override file. Never commit that
host copy, and never add a file named `docker-compose.override.yml` to the
repository: Compose auto-loads that name and would change other environments.

`docker/install.sh` invokes Compose with `-f docker-compose.yml` only, so it
does not load an override. `docker compose` run from `docker/` does.

## Operational notes from this deploy

- Redis must be able to read `redis/redis.conf` (file mode 644, directory 755).
  A 640 root:root config causes Redis to fail health checks with permission denied.
  The compose service mounts that file at `/etc/redis.conf`.
- Set `vm.max_map_count=262144` for Elasticsearch and Milvus.
- Set `vm.overcommit_memory=1` for Redis.
- After a hard `docker-compose restart`, Milvus may crash-loop (exit 137) on
  stale etcd sessions. On a fresh deploy only, clear
  `/data/bisheng-xxx/data/milvus` and `/data/bisheng-xxx/data/milvus-etcd`,
  then start etcd and milvus. Do not wipe those directories on a deploy that
  already holds vector data.
