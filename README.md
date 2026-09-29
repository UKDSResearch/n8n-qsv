# N8N

# Run
Ensure Dataverse is up and running before n8n-compose
Runs on http://localhost:5678
```
docker compose up -d
```

# Dataverse
update DV api keys if changed (final post back to DV)

# qsv (readstat) support

The `n8n` container is musl-based (Alpine), but qsv's `readstat` feature (SAS/Stata/SPSS
conversion) is only bundled in the glibc build - the official musl prebuilt omits it. The
`n8n` image is therefore built locally (see `build:` in [compose.yml](compose.yml)) from a
context that stages a glibc runtime alongside a glibc qsv binary. `DARTFX_QSV_BIN_PATH` in
`.env` must point at `/opt/custom/bin/qsv` (the wrapper script), matching the path baked into
the image.

## Fresh install steps

1. As per https://docs.n8n.io/deploy/host-n8n/install-options/one-line-setup
   ```bash
   curl -fsSL https://get.n8n.io | sh
   cd n8n

   export USER_DIR=<YOUR HOME DIR>
   ```

2. **Ensure `.env` (in this repo) has:**
   ```
   DARTFX_QSV_BIN_PATH=/opt/custom/bin/qsv

   USER_DIR=<YOUR HOME DIR>
   N8N_DIR=<PATH TO THIS N8N DIR>
   ```

3. **Get a glibc build of qsv with `readstat`.** The prebuilt Linux x86_64 GNU release already
   includes it (only the musl prebuilt excludes features). Release assets are named with the
   version embedded (e.g. `qsv-23.0.1-x86_64-unknown-linux-gnu.zip`), so resolve the current
   tag first rather than assuming a fixed filename:
   ```bash
   mkdir -p $USER_DIR/n8n-glibc-build && cd $USER_DIR/n8n-glibc-build
   QSV_VERSION=$(curl -s https://api.github.com/repos/dathere/qsv/releases/latest | grep -oP '"tag_name": *"\K[^"]+')
   curl -L -o qsv-gnu.zip "https://github.com/dathere/qsv/releases/download/${QSV_VERSION}/qsv-${QSV_VERSION}-x86_64-unknown-linux-gnu.zip"
   unzip -o qsv-gnu.zip qsv -d .
   mv qsv qsv-glibc
   chmod +x qsv-glibc
   ./qsv-glibc --version | tr ' ' '\n' | grep -i readstat   # confirm the feature is present
   ```

4. **Add the wrapper script** that runs the glibc binary through a staged glibc dynamic linker
   (`$USER_DIR/n8n-glibc-build/qsv-wrapper.sh`):
   ```sh
   #!/bin/sh
   exec /opt/custom/glibc-compat/lib/ld-linux-x86-64.so.2 --library-path /opt/custom/glibc-compat/lib /opt/custom/bin/qsv-glibc "$@"
   ```

5. **Add the Dockerfile** (`$USER_DIR/n8n-glibc-build/Dockerfile`) that layers a modern glibc runtime
   (>= 2.38, required for `__isoc23_*`/`__res_init`) plus the binary/wrapper onto the official
   n8n image:
   ```dockerfile
   ARG N8N_VERSION

   # source: a glibc runtime new enough for __isoc23_*/__res_init (glibc >= 2.38)
   # libwayland-client0 (and its libffi dependency) are not in the slim image by default,
   # but the all-features qsv binary links against them (pulled in transitively, unused headless)
   FROM debian:trixie-slim AS glibc-runtime
   RUN apt-get update && apt-get install -y --no-install-recommends libwayland-client0 \
       && rm -rf /var/lib/apt/lists/*

   FROM docker.io/n8nio/n8n:${N8N_VERSION}

   USER root

   COPY --from=glibc-runtime /lib/x86_64-linux-gnu/libc.so.6 /lib/x86_64-linux-gnu/libm.so.6 /lib/x86_64-linux-gnu/libresolv.so.2 /lib/x86_64-linux-gnu/libgcc_s.so.1 /lib/x86_64-linux-gnu/libstdc++.so.6 /lib/x86_64-linux-gnu/libssl.so.3 /lib/x86_64-linux-gnu/libcrypto.so.3 /lib/x86_64-linux-gnu/libffi.so.8 /lib/x86_64-linux-gnu/libwayland-client.so.0 /opt/custom/glibc-compat/lib/
   COPY --from=glibc-runtime /lib64/ld-linux-x86-64.so.2 /opt/custom/glibc-compat/lib/ld-linux-x86-64.so.2

   # the all_features glibc build of qsv - musl-native prebuilts do not ship readstat
   COPY qsv-glibc /opt/custom/bin/qsv-glibc
   COPY qsv-wrapper.sh /opt/custom/bin/qsv

   RUN chmod 755 /opt/custom/glibc-compat/lib/* /opt/custom/bin/qsv-glibc /opt/custom/bin/qsv

   USER node
   ```

6. **Ensure `compose.yml`'s `n8n` service builds from that context** instead of pulling a
   plain image:
   ```yaml
     n8n:
       image: n8n-with-qsv:${N8N_VERSION}
       build:
         context: ${USER_DIR}/n8n-glibc-build
         args:
           N8N_VERSION: ${N8N_VERSION}
   ```

7. **Build and start:**
   ```bash
   cd $N8N_DIR
   docker compose build n8n
   docker compose up -d
   ```

8. **Verify:**
   ```bash
   docker exec n8n-n8n-1 /opt/custom/bin/qsv readstat --help
   ```
