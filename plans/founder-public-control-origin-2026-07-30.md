---
{
  "schema": "subactor.doc/v1",
  "id": "docs.plans.founder-public-control-origin-2026-07-30",
  "version": 1,
  "status": "current",
  "updated": "2026-07-30"
}
---

# Tor 4: publiczny origin Control z auth (formularz bez localhost)

## Problem

Mail z `https://founder.subactor.com/founder/form#token=…` otwierał **statyczną
trampolinę Plesk**, która robiła `location.replace(http://127.0.0.1:8091/…)`.

- Na Lenovo z Control — działało.
- Na telefonie / innym PC — **„formularz nie działa”**.
- Preflight e-maila uznawał trampolinę HTML 200 za „ready” i wysyłał publiczny link.

## Docelowa architektura

```text
Internet
  → founder.subactor.com (DNS / opcjonalnie Cloudflare Tunnel)
    → ingress Caddy
         ├─ /founder/form* + /api/founder/form/*     token only (bez Basic Auth)
         ├─ /founder/action* + urgent respond        token only
         └─ reszta panelu                            Basic Auth (founder:hash)
              → hr-control:8181
```

Static `static_loopback_bridge` pozostaje fallbackiem, dopóki DNS nie wskazuje
ingressu. Po cutover ustaw `projekty/founder-subactor-com/site/founder/control-origin.js`
→ `mode: "public"`.

## Co wdrożono w kodzie (2026-07-30)

| Element | Zmiana |
| --- | --- |
| `platform/config/ingress/Caddyfile` | Form + API form bez Basic Auth (jak action) |
| `Caddyfile.public` | To samo dla ACME |
| `founder-endpoint-readiness.mjs` | Odrzuca trampolinę (`static_loopback_trampoline`) |
| `site/founder/form/index.html` | Brak auto-redirect na 127.0.0.1 z publicznego hosta; wybór originu |
| `control-origin.js` | `mode: bridge \| public` |
| `scripts/founder-public-control-up.sh` | Start ingress (+ opcjonalny tunnel) + probe |

## Jak włączyć live

### A. Lokalny probe ingress (już możliwy)

```bash
cd ~/github/subactor/platform
# SUBACTOR_INGRESS_HASH w .env (caddy hash-password)
bash scripts/founder-public-control-up.sh
# Oczekiwane:
#   GET /founder/form → 200 + "Subactor · Odpowiedź Foundera"
#   GET /             → 401
```

### B. Publiczny Internet (wymaga tokenu tunelu lub publicznego IP)

```bash
# Wypełnij platform/.secrets/cloudflare-tunnel-token (niepusty)
# Zero Trust: public hostname founder.subactor.com → http://ingress:80
bash scripts/founder-public-control-up.sh --with-tunnel
```

Albo hostuj Control/ingress na VPS/Plesk Docker z publicznym HTTPS i Basic Auth
(reverse-proxy-ensure wymaga publicznego upstreamu z challenge 401 — Caddy to daje).

### C. Cutover DNS / site

1. DNS `founder.subactor.com` → host z ingress (nie sam static httpdocs).
2. `control-origin.js`: `mode: "public"`.
3. Deploy site (opcjonalnie; przy pełnym reverse-proxy Control serwuje `/founder/form`).
4. Nowy mail z form linkiem — preflight musi widzieć **Control HTML**, nie trampolinę.

## EQL / DoD

- [ ] Unauth `GET https://founder.subactor.com/` → **401** (panel)
- [ ] Unauth `GET https://founder.subactor.com/founder/form` → **200** Control HTML
- [ ] Submit form tokenem (pending) → zapis decyzji, token consumed
- [ ] Preflight `founderHtmlEndpointReady` na trampolinie → `ready:false`
- [ ] Mail zawiera działający publiczny link **albo** tylko local + jasny powód

## Stan na 2026-07-30

- Tunnel token file **pusty** → publiczny Internet jeszcze nie podpięty.
- Ingress lokalny nasłuchuje `127.0.0.1:18081` (HTTP).
- Po `founder-public-control-up.sh` form na ingressie powinien iść do Control
  bez hasła Basic Auth (token w fragmencie).

## Powiązane

- [founder-plesk-native-public-control-2026-07-30.md](./founder-plesk-native-public-control-2026-07-30.md)
- PLF-592 / PLF-794 (public links EQL)
- ADR-012 twin observe vs mutate
