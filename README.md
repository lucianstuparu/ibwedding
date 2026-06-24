# ibwedding — Transylvania Wedding Run PWA

A static, self-contained PWA itinerary (Geneva → Transylvania, 25–28 Jun 2026)
served publicly at **https://ibwedding.stuparu.eu**.

Unlike the infra services (proxmox/sma/ha), this is **not** behind the Cloudflare
tunnel or any Proxmox CT. It is hosted on **GitHub Pages** — chosen for temp/throwaway
PWAs where no LAN backend is needed.

> **To change the app's content** (it's a gzip+base64 Claude-artifact bundle, not plain
> source), follow **[`EDITING.md`](EDITING.md)** — decode → edit one resource → re-pack →
> deploy → verify, with ready-to-run scripts. Don't hand-edit `index.html`.

## Hosting

- **GitHub repo:** https://github.com/lucianstuparu/ibwedding (public)
- **GitHub Pages:** source = `main` branch, `/` root (legacy build, plain static)
- **Custom domain:** `ibwedding.stuparu.eu` (via `CNAME` file in repo root)
- **TLS:** terminated at Cloudflare's edge (zone Universal SSL); the DNS record is
  proxied/orange-cloud. GitHub's own cert was never issued and is not used (see DNS).

## DNS (Cloudflare, zone stuparu.eu)

- `CNAME ibwedding → lucianstuparu.github.io`, **proxied = ON (orange cloud)**.
  - TLS is terminated at **Cloudflare's edge** using the zone's Universal SSL
    wildcard cert (`stuparu.eu` + `*.stuparu.eu`, Google Trust Services) — same
    cert family as sma/proxmox/ha. This gives instant valid HTTPS.
  - **Why not grey-cloud / GitHub's own cert:** we started grey-cloud, but on
    2026-05-31 GitHub's Let's Encrypt cert for the custom domain never issued
    (stuck >1h in their queue with no fault on our side — DNS correct, no CAA,
    domain verified). Flipping to orange-cloud made Cloudflare serve a valid cert
    immediately. Cloudflare → origin pull to GitHub works under the zone's SSL
    mode (no 526). Do NOT enable GitHub "Enforce HTTPS" while proxied (GitHub's
    own cert may still be unissued; let Cloudflare handle the redirect).
  - Note: with the orange cloud on, GitHub's ACME HTTP-01 challenge now passes
    through Cloudflare, so GitHub may never issue its own cert — that's fine, we
    don't use it. Cloudflare caches the HTML (~10 min); purge the CF cache after
    a redeploy if you need it live immediately.

## Access / privacy

- **Public** to anyone with the link. GitHub Pages has no auth on non-Enterprise
  accounts.
- Kept out of search engines via `<meta name="robots" content="noindex,nofollow,noarchive">`
  in `index.html` and a root `robots.txt` (`Disallow: /`). This is obscurity, not auth.
- If real gating is ever needed, move hosting to CT 101 behind the tunnel +
  Cloudflare Access (see `runbooks/expose-service.md` in the `our.infra` repo).

## Source of the PWA

Built externally (a bundler-generated single-file app: `index.html` ≈ 1.6 MB plus
icons, `manifest.webmanifest`, `sw.js`). The repo is the source of truth for what's
deployed — push to `main` to update; GitHub Pages rebuilds automatically.

## Redeploy / update

```bash
# clone or pull https://github.com/lucianstuparu/ibwedding
# replace files, keep CNAME / robots.txt / .nojekyll / the noindex meta
git add -A && git commit -m "update" && git push origin main
# Pages rebuilds in ~30-60s
```

## Verify

```bash
curl -sI https://ibwedding.stuparu.eu/            # 200, Server: cloudflare
# cert should be CN=stuparu.eu / SAN *.stuparu.eu (Cloudflare edge), not *.github.io:
echo | openssl s_client -servername ibwedding.stuparu.eu -connect ibwedding.stuparu.eu:443 2>/dev/null | openssl x509 -noout -issuer
```

## Notes

- `.nojekyll` is present so GitHub Pages serves files verbatim (no Jekyll processing).
- First HTTPS cert issuance after pointing DNS can take a few minutes to ~15 min;
  HTTP works immediately. Enable "Enforce HTTPS" in repo Settings → Pages once the
  cert shows `approved` (or via the API).
