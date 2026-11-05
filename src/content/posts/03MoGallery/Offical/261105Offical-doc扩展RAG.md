---
title: 261105Offical-doc扩展RAG
published: 2026-11-05
description: 261105Offical-doc扩展RAG
tags:
  - RAG
category: MO Gallery
draft: true
---
RAG一般用于基于文档的知识库问答场景，并非所有情况都适用，文档量很小时通常没有必要；但考虑到后续文档会持续增加，提前引入更稳妥。

数据库目前使用的是Postgres，可以通过扩展的方式支持（ https://github.com/pgvector/pgvector ）。

先说明一个容易误解的点：**「扩展」不是数据库里的一个开关，它本体是文件。**pgvector 的实体是三个文件——`vector.so`（编译好的 C 动态库）、`vector.control`（声明文件）、`vector--0.8.0.sql`（建类型和函数的 SQL）。`CREATE EXTENSION` 只是「登记」，找不到文件就会报 `extension "vector" is not available`，它不会去下载也不会去编译。所以整件事分两层：**系统层装文件** + **库层面启用**。

另外，本文的目标环境是 1Panel 装的 `postgres:17.5-alpine` 容器，同一个实例上有 19 个业务库，以下每一步都按这个前提写。命令里先用变量拿到容器名，避免写死：

```bash
PGC=$(docker ps --format '{{.Names}}' | grep -i postgres | head -1); echo "$PGC"
```

## 一、确认扩展状态

```bash
docker exec "$PGC" psql -U mo-xxx -d mo-xxx \
  -c "SELECT * FROM pg_available_extensions WHERE name = 'vector'" \
  -c "SELECT extversion FROM pg_extension WHERE extname = 'vector'"
```

- 第一条有行 = 文件已装好，可用；空 = 没装（走第二步编译）。
- 第二条有行 = 当前库已启用；空但第一条有行 = 装了没启用，只差一句 `CREATE EXTENSION`。

## 二、编译 pgvector（一次性容器，不碰生产）

做法：用与生产同镜像的临时容器编译，产物输出到宿主机，生产容器不做任何改动。

```bash
mkdir -p /opt/pgvector/pg17 && cd /opt/pgvector/pg17
docker run --rm -v "$PWD":/out postgres:17.5-alpine sh -c '
  set -e
  apk add --no-cache -X https://dl-cdn.alpinelinux.org/alpine/v3.22/main \
      -X https://dl-cdn.alpinelinux.org/alpine/v3.22/community gcc musl-dev make git
  git clone --branch v0.8.0 --depth 1 https://github.com/pgvector/pgvector.git /tmp/pv
  cd /tmp/pv && make with_llvm=
  make install with_llvm= || {
    install -m 755 vector.so /usr/local/lib/postgresql/
    install -m 644 vector.control /usr/local/share/postgresql/extension/
    install -m 644 sql/vector--0.8.0.sql /usr/local/share/postgresql/extension/
  }
  cp /usr/local/lib/postgresql/vector.so /out/
  cp /usr/local/share/postgresql/extension/vector.control /out/
  cp /usr/local/share/postgresql/extension/vector--0.8.0.sql /out/
'
ls -l
```

`with_llvm=` 不能省，也不能写成 `no`：官方 PG 镜像编译时开了 LLVM JIT，`pgxs` 会额外为每个 `.c` 生成 `.jit` 用的 `.bc`，需要 `clang-19`，而 alpine 镜像里没有，会报：

```
make: clang-19: No such file or directory
Error 127
```

而 `Makefile.global` 是用 `ifdef with_llvm` 判断的——`ifdef` 只看「值是否非空」，所以必须传**空值** `with_llvm=`，传 `no` 依然算已定义。若 `make install` 仍失败，`|| {}` 里用已经编好的 `vector.so` 手工 install 兜底。

![](assets/Pasted%20image%2020261105180044.png)

## 三、先在临时容器里验证（不动生产）

文件编出来了不代表能给 PG 用，先用一个一次性的库确认版本匹配、能 `CREATE EXTENSION`：

```bash
docker run --rm -d --name pgv-test -e POSTGRES_PASSWORD=x postgres:17.5-alpine
sleep 8
docker cp vector.so                        pgv-test:/usr/local/lib/postgresql/
docker cp vector.control                   pgv-test:/usr/local/share/postgresql/extension/
docker cp vector--0.8.0.sql                pgv-test:/usr/local/share/postgresql/extension/
docker exec pgv-test psql -U postgres \
  -c "CREATE EXTENSION vector" \
  -c "SELECT extversion FROM pg_extension WHERE extname='vector'" \
  -c "SELECT '[1,2,3]'::vector <=> '[1,2,4]'::vector AS cosine_distance"
docker rm -f pgv-test
```

最后一句返回一个很小的数（约 `0.0085`）就说明 `<=>` 余弦算子可用。

![](assets/Pasted%20image%2020261105180250.png)

## 四、把三个文件挂进生产 PG 容器（这步漏了，前面全白做）

生产容器里没有这三个文件，必须挂进去。编辑 `/opt/1panel/apps/postgresql/postgresql/docker-compose.yml`，在 postgres 服务的 `volumes:` 列表里（和 `- ./data:/var/lib/postgresql/data` 同级、缩进一致）追加三行**只读**挂载：

```yaml
    volumes:
      - ./data:/var/lib/postgresql/data
      # pgvector：只读挂载。注意 1Panel 重建/升级这个 PG 应用会覆盖本文件，见文末
      - /opt/pgvector/pg17/vector.so:/usr/local/lib/postgresql/vector.so:ro
      - /opt/pgvector/pg17/vector.control:/usr/local/share/postgresql/extension/vector.control:ro
      - /opt/pgvector/pg17/vector--0.8.0.sql:/usr/local/share/postgresql/extension/vector--0.8.0.sql:ro
```

修改后先校验语法，避免重建时才发现缩进错误：

```bash
cd /opt/1panel/apps/postgresql/postgresql && docker compose config -q && echo OK
```

## 五、备份 + 重建（唯一会断连的一步）

**该实例上有 19 个业务库，`up -d` 会使它们同时断连 5～15 秒。**因此先备份、再重建，并**严禁使用 `-v`**（会连同数据卷一起删除）。

```bash
mkdir -p /root/backup
docker exec "$PGC" pg_dump -U mo-xxx -d mo-xxx -Fc -f /tmp/mo-xxx.dump
docker cp "$PGC":/tmp/mo-xxx.dump /root/backup/mo-xxx-$(date +%F).dump
cp /opt/1panel/apps/postgresql/postgresql/docker-compose.yml \
   /opt/1panel/apps/postgresql/postgresql/docker-compose.yml.bak-$(date +%F)
ls -lh /root/backup
```

![](assets/Pasted%20image%2020261105180604.png)

重建并确认挂载生效：

```bash
cd /opt/1panel/apps/postgresql/postgresql && docker compose up -d --force-recreate postgresql
docker ps --filter name=postgres --format '{{.Names}}\t{{.Status}}'
docker inspect "$PGC" --format '{{range .Mounts}}{{.Source}} -> {{.Destination}} rw={{.RW}}{{"\n"}}{{end}}'
docker exec "$PGC" ls -l /usr/local/lib/postgresql/vector.so
docker exec "$PGC" psql -U mo-xxx -d mo-xxx \
  -c "SELECT name, default_version FROM pg_available_extensions WHERE name='vector'"
```

三个挂载都应显示 `rw=false`，最后一条有 `vector | 0.8.0` 即表示文件已就位。

（服务名是 `postgresql` 而不是 `postgres`，用 `docker compose config --services` 可以确认。）

![](assets/Pasted%20image%2020261105180850.png)

## 六、在业务库启用扩展

```bash
docker exec "$PGC" psql -U mo-xxx -d mo-xxx \
  -c "CREATE EXTENSION IF NOT EXISTS vector" \
  -c "SELECT extversion FROM pg_extension WHERE extname='vector'" \
  -c "SELECT '[1,2,3]'::vector <=> '[1,2,4]'::vector AS cosine_distance"
```

说明：应用发布时，迁移 `20261105101500_doc_rag_chunks` 的第一句即 `CREATE EXTENSION IF NOT EXISTS vector;`，正常流程无需手工执行；此处手工执行一次，目的是**先确认数据库侧可用**，以便后续排查时能区分扩展问题与代码问题。语句带 `IF NOT EXISTS`，重复执行只返回一条 NOTICE。

## 七、应用侧：配置 → 迁移 → 建索引 → 验证

数据库侧完成后，应用侧还有四步：

```bash
# 1. 配置（.env）
RAG_BASE_URL="https://emrank.tech/v1"
RAG_API_KEY="..."
RAG_EMBEDDING_MODEL="BAAI/bge-m3"
# RAG_EMBEDDING_DIM / RAG_BATCH_SIZE 留空即可，默认 1024 / 16

# 2. 迁移（建 DocChunk / DocIndexMeta）
npx prisma migrate deploy

# 3. 建索引（先 dry-run 看切块数量，再正式跑）
pnpm rag:index --dry-run
pnpm rag:index

# 4. 验证
curl -s localhost:3001/api/docs/search -H 'content-type: application/json' \
  -d '{"query":"怎么配置 S3","locale":"zh"}'
```

- `RAG_BASE_URL` 填到 `/v1` 即可，代码会自动拼接 `/embeddings` 与 `/chat/completions`（写 `.../v1/embeddings` 也会被自动剥掉后缀）。
- `RAG_EMBEDDING_DIM` 不可修改：它必须等于 `DocChunk.embedding` 的 `vector(1024)`，不一致会直接报错。
- 索引是**按块**存储的：319 块对应 48 篇文档（16 篇 × 3 语言），并非一篇一个向量。再次执行时只重算内容有变化的块。
- 问答入口在文档页右下角浮动按钮（≥768px 显示），或直接 `POST /api/docs/ask`。

![](assets/Pasted%20image%2020261105183408.png)

## 八、回滚

```bash
# 数据回滚
docker exec -i "$PGC" pg_restore -U mo-xxx -d mo-xxx --clean --if-exists \
  < /root/backup/mo-xxx-2026-11-05.dump

# 配置回滚（去掉三行挂载）
cp /opt/1panel/apps/postgresql/postgresql/docker-compose.yml.bak-2026-11-05 \
   /opt/1panel/apps/postgresql/postgresql/docker-compose.yml
cd /opt/1panel/apps/postgresql/postgresql && docker compose up -d --force-recreate postgresql
```

## 注意事项：1Panel 会覆盖 compose

那三行挂载是**手工修改**在 1Panel 生成的 compose 中的。其后若在 1Panel 面板里重建或升级该 PG 应用，compose 会被重新渲染，挂载随之丢失——此时若库中已存在 `vector(1024)` 列，所有相关查询会直接报 `type "vector" does not exist`，而非静默变慢。

两种应对：升级后重新添加一次挂载并重建；或另起一个 `pgvector/pgvector:pg17` 容器承载该库，把 `DATABASE_URL` 指过去，彻底摆脱对 1Panel 应用模板的依赖。
