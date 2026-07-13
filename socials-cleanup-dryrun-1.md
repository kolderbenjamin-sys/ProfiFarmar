# SOCIALS cleanup — DRY RUN 1/3 (2026-07-13T04:11:17Z)

Cloud: `dxrpsbvx2`, prefix `SOCIALS/`, ČLÁNKY folder not queried/touched.
Total resources found: 21. Hard floor (6 newest, never delete) applied.
**No deletions performed** — dry run only, per instructions (first 3 runs must be dry run).

## Hard floor — KEEP, never delete (6 newest by created_at)

| public_id | created_at |
|---|---|
| SOCIALS/social_2026-06_halter_starlink | 2026-07-12T04:10:14Z |
| SOCIALS/social_2026-07_weidemann_t6025 | 2026-07-12T04:10:11Z |
| SOCIALS/social_2026-07_vlny_veder_mleko | 2026-07-12T04:09:56Z |
| SOCIALS/social_2026-06_eu_dotace_strop | 2026-07-11T04:11:20Z |
| SOCIALS/social_2026-06_kataralni_horecka | 2026-07-11T04:11:10Z |
| SOCIALS/social_2026-06_lely_hub_vector | 2026-07-11T04:10:56Z |

## Would DELETE — matched a Buffer "sent" post within last 7 days (10)

Matched by public_id found in `asset.source` URL of a `list_posts(status=["sent"])` result with `dueAt` in the last 7 days.

| public_id | created_at | age (h) |
|---|---|---|
| SOCIALS/social_2026-06_cesky_med_varroaza | 2026-07-10T15:05:13Z | 61.1 |
| SOCIALS/social_2026-06_eu_hnojiva_540m | 2026-07-10T15:05:10Z | 61.1 |
| SOCIALS/social_2026-06_yanmar_yt4s5s | 2026-07-10T15:05:07Z | 61.1 |
| SOCIALS/social_2026-07_cap_rozpocet | 2026-07-08T04:10:16Z | 120.0 |
| SOCIALS/social_2026-07_ptaci_chripka | 2026-07-08T04:10:14Z | 120.0 |
| SOCIALS/social_2026-07_traktor_roku | 2026-07-08T04:10:10Z | 120.0 |
| SOCIALS/social_2026-06_biostimulanty_trh | 2026-07-07T04:11:43Z | 144.0 |
| SOCIALS/social_2026-06_coceral_sklizen | 2026-07-07T04:11:41Z | 144.0 |
| SOCIALS/social_2026-07_fendt_900vario | 2026-07-07T04:11:38Z | 144.0 |
| SOCIALS/social_2026-07_cr11 | 2026-07-06T04:10:38Z | 168.0 |

## Would DELETE — no Buffer match found, but age >= 48h fallback (5)

| public_id | created_at | age (h) | Buffer match detail |
|---|---|---|---|
| SOCIALS/social_2026-07_nl_kultivovane_maso | 2026-07-05T04:10:55Z | 192.0 | Matched a **sent** post, but `dueAt` = 2026-07-05T18:00Z — 8 days ago, just outside the 7-day window. Legitimately posted. |
| SOCIALS/social_2026-07_case_ih_puma_reddot | 2026-07-05T04:10:53Z | 192.0 | Same — sent 2026-07-05T15:00Z, 8 days ago, outside window. Legitimately posted. |
| SOCIALS/social_2026-07_kubota_m5_m7004 | 2026-07-05T04:10:36Z | 192.0 | Same — sent 2026-07-05T05:00Z, 8 days ago, outside window. Legitimately posted. |
| **SOCIALS/social_2026-06_valtra_cvt** | 2026-07-06T04:10:42Z | 168.0 | ⚠️ Only Buffer match found (any status, any date) is a **sent** post from **2026-06-30** — 6 days *before* this Cloudinary asset's `created_at`. Implies the public_id was overwritten (re-uploaded) on 07-06 after already being posted once; the current image version at this public_id may never have been posted itself. |
| **SOCIALS/social_2026-06_farmdroid_spray** | 2026-07-06T04:10:44Z | 168.0 | ⚠️ **Zero Buffer posts found in any status** (sent/scheduled/error/draft), across full post history back to 2026-05-14. This visual may never have been sent via Buffer at all. |

## Flag for manual review before enabling real deletion

`social_2026-06_valtra_cvt` and `social_2026-06_farmdroid_spray` are being marked for deletion only
because of the 48h age fallback, not because a matching sent post was found. This is consistent with
the literal task rule ("no match + age>=48h → delete anyway"), but means their current image content
may never have gone out on social media. Worth confirming manually whether these correspond to posts
that failed silently in the Buffer scheduling step, or whether the public_id was legitimately reused.

## Summary

- 21 total assets in SOCIALS/
- 6 kept (hard floor)
- 15 would be deleted (10 confirmed via Buffer sent-match, 5 via 48h-age fallback — 3 of those 5 are
  legitimately-sent posts just outside the 7-day window, 2 are the flagged anomalies above)
- 0 actually deleted (dry run)
- Real deletion via Cloudinary `destroy` API stays disabled until dry run 3/3 completes and the Buffer
  API matching approach above is explicitly confirmed valid.
