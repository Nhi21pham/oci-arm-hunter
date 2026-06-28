# Migrating StoreTrack onto the free ARM box

When the hunt grabs a free **VM.Standard.A1.Flex** (ARM64, 2 OCPU / 12 GB), the
instance is **created and booting** — but empty. This is the checklist to move
StoreTrack onto it from the current paid box (`161.118.219.153`).

> Keep the paid box running until the new one is fully verified. Nothing here
> touches the old box's data destructively — you only *read* from it.

Variables used below:
- `NEW_IP` — public IP from the "🎉 acquired" issue
- `OLD_IP=161.118.219.153` — current paid box
- SSH user is **`ubuntu`** (Canonical Ubuntu image). If the image is Oracle
  Linux instead, use **`opc`**.

---

## 1. Open the network (do this first — easy to forget)

ARM Ubuntu OCI images block inbound traffic in **two** places; both must allow
80/443 or Caddy/ACME silently fails:

1. **OCI VCN Security List / NSG** (cloud side): add ingress rules
   `0.0.0.0/0 → TCP 80` and `→ TCP 443` (Networking → VCN → Security Lists).
2. **In-instance iptables** (Ubuntu OCI images ship a default REJECT):
   ```bash
   ssh ubuntu@NEW_IP
   sudo iptables -I INPUT 6 -p tcp --dport 80  -j ACCEPT
   sudo iptables -I INPUT 6 -p tcp --dport 443 -j ACCEPT
   sudo netfilter-persistent save
   ```

## 2. Install Docker on the new box

```bash
ssh ubuntu@NEW_IP
sudo apt-get update && sudo apt-get install -y docker.io docker-compose-v2
sudo usermod -aG docker ubuntu && exit   # re-login so the group applies
```

## 3. Copy the app tree (code + built frontend + .env)

The whole deployment lives in `~/storetrack` on the old box (it is **not** a git
checkout — code was rsync'd). Carry it over via your laptop (which has the OCI
key for both boxes):

```bash
# on your laptop
rsync -az -e ssh --exclude node_modules --exclude '.git' \
  ubuntu@OLD_IP:storetrack/ ./storetrack-snapshot/
rsync -az -e ssh ./storetrack-snapshot/ ubuntu@NEW_IP:storetrack/
```

Keep **`backend/.env`** and the root **`.env`** exactly as-is — the `APP_KEY`
must stay identical or every encrypted DB value (and session) breaks. The root
`.env` also holds `MYSQL_ROOT_PASSWORD` / `MYSQL_DATABASE` used below.
`frontend/dist` is prebuilt and comes along; no front-end rebuild needed.

## 4. Dump the database (genuine MySQL 8 client)

Generated columns are mis-dumped by the MariaDB client — use the `mysql:8.0`
container's own `mysqldump`:

```bash
# on the OLD box
cd ~/storetrack && set -a && . ./.env && set +a
docker compose -f docker-compose.prod.yml exec -T db \
  mysqldump -u root -p"$MYSQL_ROOT_PASSWORD" \
  --single-transaction --routines --triggers \
  "$MYSQL_DATABASE" > ~/storetrack-db.sql
# sanity-check it actually has data, don't trust file size alone:
grep -c 'INSERT INTO' ~/storetrack-db.sql
```

Copy it to the new box (`rsync ... ~/storetrack-db.sql`).

## 5. Restore the DB, then bring up the full stack

```bash
# on the NEW box
cd ~/storetrack && set -a && . ./.env && set +a
docker compose -f docker-compose.prod.yml up -d db        # wait until healthy
docker compose -f docker-compose.prod.yml exec -T db \
  mysql -u root -p"$MYSQL_ROOT_PASSWORD" "$MYSQL_DATABASE" < ~/storetrack-db.sql

docker compose -f docker-compose.prod.yml up -d --build    # builds ARM64 images
```

The app/queue/scheduler images build **natively for arm64** from the local
Dockerfile, and `mysql:8.0` / `caddy:2-alpine` / `redis` are all multi-arch — no
cross-architecture work needed. Watch the build log for any PHP-extension
compile error (rare on arm64).

> The DB is already migrated by the restore — **do not** run `migrate:fresh`.

## 6. Permissions + cache (the classic gotcha)

`www-data` is uid 33 inside the container and must own the env + writable dirs:

```bash
sudo chown 33:33 backend/.env
sudo chown -R 33:33 backend/storage backend/bootstrap/cache
docker compose -f docker-compose.prod.yml exec app php artisan config:cache
```

`APP_URL` is already `https://storetrack.io.vn`, so no change there.

## 7. Repoint the domain (iNET)

At portal.inet.vn (OneShield panel), point both records at the new box:

- `@`   → `NEW_IP`
- `www` → `NEW_IP`
- **No AAAA record.** **OneShield "Protection"/proxy MUST be OFF** (plain
  DNS-only A records) — if it's on, iNET publishes its 103.x proxy IPs + a
  `2001:df7::` AAAA, hides the origin, and Caddy's ACME http-01 challenge fails.

Verify propagation with Google DoH (the local WSL resolver caches stale answers):

```bash
curl -s 'https://dns.google/resolve?name=storetrack.io.vn&type=A'
```

Once DNS points to the new box and ports 80/443 are open, Caddy auto-obtains a
fresh Let's Encrypt cert. Watch it: `docker compose -f docker-compose.prod.yml logs -f caddy`.

## 8. Verify, then decommission the old box

- Open **https://storetrack.io.vn**, log in, and confirm data is all there
  (spot-check a few row counts against the old box).
- Sanctum tokens live in the DB and were restored, so existing logins survive.
- Leave the paid box up for a day as a fallback, then **terminate it** in the
  OCI console to stop burning trial credit.

---

### Quick gotcha index
| Area | Watch out for |
|------|---------------|
| Firewall | OCI Security List **and** Ubuntu iptables must allow 80/443 |
| Architecture | ARM64 — images build natively (compose `--build`); nothing pinned to amd64 |
| DB dump | Use the `mysql:8.0` container's `mysqldump`; verify row counts before trusting it |
| `.env` | Keep `APP_KEY` identical or encrypted values break |
| Permissions | `chown 33:33` on `backend/.env` + `storage` + `bootstrap/cache` |
| DNS | iNET OneShield proxy **OFF**, A-records only, no AAAA |
