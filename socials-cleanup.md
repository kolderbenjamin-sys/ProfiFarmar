---
name: socials-cleanup
description: "Úklidová routine pro Cloudinary složku SOCIALS (cloud dxrpsbvx2) — maže staré vizuály po úspěšném odeslání přes Buffer, s hard-floor pojistkou 6 nejnovějších assetů. Nezávislá na agro-socials-local/cloud skillech. Trigger keywords: socials cleanup, uklid cloudinary, smaž staré vizuály, cleanup socials."
---

# Socials-Cleanup Skill

Jednorázový/opakovaný úklidový běh nad Cloudinary složkou `SOCIALS/` (cloud `dxrpsbvx2`), nezávislý na
`agro-socials-local`/`agro-socials-cloud`. Cíl: smazat vizuály, které už byly úspěšně odeslané přes
Buffer, a udržet složku malou — s tvrdou pojistkou, že se nikdy nesmaže posledních 6 nejnovějších assetů.

> **Ostrý běh — bez dry-run.** Matching přes Buffer API byl ověřen (3 běhy dry-run, 2026-07-13) a
> potvrzen uživatelem jako validní. Od teď skill **mazání provádí skutečně**, žádný dry-run flag.
> Historie běhů a jejich výsledky jsou v `socials-cleanup-log.json` (append po každém běhu).

---

## Předpoklady prostředí

- **Environment variables:** `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`,
  `BUFFER_API_KEY` (stejné jako `agro-socials-cloud`).
- **Repo obsahuje** `socials-cleanup-log.json` — append-only historie běhů (run_number, run_at, mode,
  counts, poznámky).
- **Nedotýkat se `ČLÁNKY/`** — všechny Cloudinary Admin API dotazy jdou vždy s `prefix=SOCIALS/`, nikdy
  bez prefixu ani s obecným listováním celého cloudu.

---

## Krok 1 — Vypiš resources v SOCIALS/ (Cloudinary Admin API)

```bash
set -euo pipefail
: "${CLOUDINARY_CLOUD_NAME:?chybí}"; : "${CLOUDINARY_API_KEY:?chybí}"; : "${CLOUDINARY_API_SECRET:?chybí}"

curl -sS -u "${CLOUDINARY_API_KEY}:${CLOUDINARY_API_SECRET}" \
  "https://api.cloudinary.com/v1_1/${CLOUDINARY_CLOUD_NAME}/resources/image?prefix=SOCIALS/&type=upload&max_results=500" \
  > /tmp/socials_raw.json

# pokud .next_cursor není "null"/chybí, je potřeba stránkovat (&next_cursor=...) — v praxi
# < 500 assetů, ale kontrola se nevyplácí přeskočit
jq '.next_cursor' /tmp/socials_raw.json

jq '[.resources[] | {public_id, created_at}] | sort_by(.created_at) | reverse' /tmp/socials_raw.json \
  > /tmp/socials_sorted.json
```

**Hard floor:** prvních **6** položek z `/tmp/socials_sorted.json` (nejnovější `created_at`) se **nikdy**
nemažou, bez ohledu na Buffer matching.

---

## Krok 2 — Buffer: najdi "sent" posty za posledních 7 dní

```bash
: "${BUFFER_API_KEY:?chybí}"

call_buffer() {
  local method="$1" params="$2" raw
  raw=$(curl -sS -X POST "https://mcp.buffer.com/mcp" \
    -H "Authorization: Bearer $BUFFER_API_KEY" \
    -H "Content-Type: application/json" \
    -H "Accept: text/event-stream, application/json" \
    -d "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"$method\",\"params\":$params}")
  # Odpověď může být plain JSON nebo SSE (řádky "data: ") — obojí ošetřit
  if echo "$raw" | grep -q '^data: '; then
    echo "$raw" | grep '^data: ' | tail -1 | sed 's/^data: //'
  else
    echo "$raw"
  fi
}

org_id=$(call_buffer "tools/call" '{"name":"get_account","arguments":{}}' \
  | jq -r '.result.content[0].text | fromjson | .organizations[0].id')

SEVEN_D_AGO=$(date -u -d "-7 days" +%Y-%m-%dT%H:%M:%SZ)
args=$(jq -n --arg org "$org_id" --arg start "$SEVEN_D_AGO" \
  '{organizationId:$org, status:["sent"], dueAt:{start:$start}, sort:[{field:"dueAt",direction:"desc"}], first:100}')

posts=$(call_buffer "tools/call" "{\"name\":\"list_posts\",\"arguments\":$args}")
echo "$posts" | jq -r '.result.content[0].text' > /tmp/posts_sent.json

# hasNextPage kontrola — pokud true a víc než 100 postů za 7 dní existuje, stránkuj přes `after` cursor
jq '.pageInfo.hasNextPage' /tmp/posts_sent.json

# public_id z asset.source URL (obsahuje "SOCIALS/<public_id>.png")
jq -r '.edges[].node.assets[]?.source // empty' /tmp/posts_sent.json \
  | grep -o 'SOCIALS/[^."]*' | sort -u > /tmp/matched_pids.txt
```

**Matching:** public_id assetu (bez `SOCIALS/` prefixu a bez verze/extension v URL) se hledá jako
substring v `asset.source` URL sent postu. Shoda = post byl úspěšně odeslán s tímto vizuálem během
posledních 7 dní.

---

## Krok 3 — Rozhodnutí per asset (mimo hard floor)

Pro každý asset **kromě top 6** z Kroku 1:

- **public_id je v `/tmp/matched_pids.txt`** → smazat (odpovídající post byl `sent` za posledních 7 dní).
- **public_id NENÍ v `/tmp/matched_pids.txt`**:
  - `created_at` starší nebo rovno **48 hodin** → smazat (fallback pojistka — nechceme, aby složka
    rostla donekonečna kvůli chybějícímu/chybnému Buffer matchi).
  - `created_at` mladší než 48 hodin → **nechat na příští běh** (post ještě může být ve frontě).

```bash
python3 - "$@" << 'PYEOF'
import json, datetime, sys

now = datetime.datetime.utcnow().replace(tzinfo=datetime.timezone.utc)

with open("/tmp/socials_sorted.json") as f:
    resources = json.load(f)
with open("/tmp/matched_pids.txt") as f:
    matched = set(l.strip() for l in f if l.strip())

top6 = set(r["public_id"] for r in resources[:6])

delete_list, keep_list = [], []
for r in resources:
    pid = r["public_id"]
    created = datetime.datetime.fromisoformat(r["created_at"].replace("Z", "+00:00"))
    age_h = (now - created).total_seconds() / 3600
    if pid in top6:
        keep_list.append((pid, "top6_floor"))
    elif pid in matched:
        delete_list.append((pid, "buffer_sent_match"))
    elif age_h >= 48:
        delete_list.append((pid, "age_fallback_no_match"))
    else:
        keep_list.append((pid, "age<48h_wait_next_run"))

with open("/tmp/delete_list.json", "w") as f:
    json.dump([d[0] for d in delete_list], f)

print("DELETE:", len(delete_list))
for d in delete_list: print(" ", d)
print("KEEP:", len(keep_list))
for k in keep_list: print(" ", k)
PYEOF
```

> Assety mazané jen přes `age_fallback_no_match` (žádný Buffer match vůbec) stojí za extra pozornost
> v run summary (Krok 5) — může jít o post, který v Bufferu selhal/nebyl nikdy odeslán. Skill je i tak
> maže (to je záměrné chování potvrzené uživatelem), ale ohlásí je zvlášť.

---

## Krok 4 — Smazání přes Cloudinary Admin API (autentizované, ne veřejné)

**Nikdy** nepoužívat nepodepsané/veřejné delivery API. Mazání jde vždy přes autentizovaný Admin API
endpoint (`-u api_key:api_secret`), bulk `DELETE` na `/resources/image/upload`:

```bash
public_ids=$(jq -r '.[]' /tmp/delete_list.json)

# --data-urlencode "public_ids[]=<pid>" pro každý pid; sestav curl argumenty dynamicky
curl_args=()
for pid in $public_ids; do
  curl_args+=(--data-urlencode "public_ids[]=${pid}")
done

curl -sS -u "${CLOUDINARY_API_KEY}:${CLOUDINARY_API_SECRET}" -G \
  "https://api.cloudinary.com/v1_1/${CLOUDINARY_CLOUD_NAME}/resources/image/upload" \
  "${curl_args[@]}" \
  -X DELETE > /tmp/delete_result.json

jq '.deleted' /tmp/delete_result.json
```

Ověř `"deleted"` status pro každý `public_id` ve výstupu. Cloudinary bulk delete zvládá až 100 ID na
jedno volání — pokud `delete_list.json` má víc, rozděl na dávky po 100.

---

## Krok 5 — Zapiš run do `socials-cleanup-log.json` a commitni

```bash
jq -n \
  --argjson run_number "$(jq 'length + 1' socials-cleanup-log.json)" \
  --arg run_at "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --argjson total "$(jq 'length' /tmp/socials_sorted.json)" \
  --argjson deleted "$(jq 'length' /tmp/delete_list.json)" \
  '{run_number: $run_number, run_at: $run_at, mode: "live", cloud_name: env.CLOUDINARY_CLOUD_NAME,
    prefix: "SOCIALS/", total_resources: $total, hard_floor_kept: 6,
    actually_deleted: $deleted}' > /tmp/new_run.json

jq -s '.[0] + [.[1]]' socials-cleanup-log.json /tmp/new_run.json > /tmp/merged_log.json
mv /tmp/merged_log.json socials-cleanup-log.json

git add socials-cleanup-log.json
git commit -m "socials-cleanup: run $(date -u +%Y-%m-%d) — $(jq 'length' /tmp/delete_list.json) deleted"
git push
```

---

## Krok 6 — Run summary

```
✅ Socials-cleanup — [datum běhu]

Celkem assetů v SOCIALS/: [N]
Hard floor (nikdy nemazáno): 6
Smazáno: [M]
  - přes Buffer sent-match (7 dní): [X]
  - přes 48h age-fallback (bez Buffer matche): [Y]  ⚠️ zkontroluj, jestli tyto posty opravdu vyšly
Ponecháno na příští běh (bez matche, < 48h staré): [Z]

Smazané public_id: [seznam]
```

---

## Troubleshooting

| Problém | Řešení |
|---|---|
| Cloudinary `next_cursor` není null | Stránkuj přes `&next_cursor=<hodnota>` dokud není null, jinak se staré assety mimo první stránku nikdy nevyhodnotí |
| Buffer `list_posts` vrátí `hasNextPage: true` a chybí matchi | Stránkuj přes `after: <endCursor>`, hlavně pokud je za 7 dní publikováno hodně postů |
| Buffer odpověď není SSE ani čistý JSON | Zkontroluj `Accept` header (`text/event-stream, application/json`) a token platnost |
| public_id matching false positive | Match je jen substring v URL — pokud se `public_id` jednoho assetu vyskytuje jako podřetězec jiného (např. shodný prefix), zpřesni na `SOCIALS/<public_id>.` (s tečkou před extension) |
| Asset smazán, ale nikdy nebyl v Bufferu (`age_fallback_no_match`) | Očekávané chování dle zadání (bezpečnostní fallback proti nekonečnému růstu složky) — zaznamenej do run summary pro zpětnou kontrolu, nekonzultuj před smazáním (uživatel to potvrdil 2026-07-13) |
| Omylem smazán asset ze `ČLÁNKY/` | Nemělo by nastat — všechny dotazy mají pevný `prefix=SOCIALS/`; pokud se to stane, zkontroluj, že žádný krok neposílá dotaz bez prefixu |
