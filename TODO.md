# TODO

- [ ] Correct DNS/TLS for `identity.subactor.com` and `chat.subactor.com`.
- [ ] Confirm Plesk add-on domain quota through an authoritative panel/API path.
- [ ] Configure a trusted HTTPS Plesk endpoint; live `auth/query/status` is fail-closed with `plesk_https_required`.
- [ ] Document credential bootstrap and phone-provider legal requirements.
- [ ] Keep the live status report synchronized with tested evidence.
- [ ] Add an umbrella manifest/bootstrap for the independent `platform`, `docs`, `logo` and `www` repositories.
- [x] Wire the read-only Access Resolver path to live urirun discovery, auth status, scope proof and acquisition-method routes.
- [ ] Persist Access Resolver evidence and consent resumptions in an append-only outbox.
- [ ] Gate bootstrap/refresh/delegate adapters with signed, one-time child grants bound to `plan_hash`.
- [ ] Roll out secret-free auth conformance to DNS, GitHub, e-mail, voice and e-sign connectors.
- [ ] Add a typed notification outbox only for AQL/consent/MFA/root-of-trust blockers after automatic strategies are exhausted.
