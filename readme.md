# family-cron

Frontend i crons automàtics de l'app de compleanys familiars.

**GitHub:** `rxampeny/family-cron`
**Frontend:** https://hola-familia.netlify.app (Netlify, auto-desplegat en push a `main`)

---

## Que conté

```
index.html                          → SPA completa (HTML + CSS + JS en català)
supabase-client.js                  → client per cridar les Edge Functions de Supabase
netlify.toml                        → configuració Netlify (redirecció SPA)
public/                             → fotos i icones de la família
.github/workflows/cron-emails.yml   → GitHub Actions: emails i SMS diaris automàtics
```

## Crons automàtics (GitHub Actions)

| Hora (Madrid) | Acció |
|---------------|-------|
| 04:00 AM | `send-birthday-emails` — email de felicitació al cumpleanyero |
| 04:00 AM | `send-birthday-sms` — SMS de felicitació al cumpleanyero |
| 06:00 AM | `send-reminders` — avisa tota la família del cumpleaños de demà |

## Backend
Les Edge Functions (API) estan desplegades a Supabase (project ref: `bfforytzxicguykosnna`).

## Secrets necessaris a GitHub
- `SUPABASE_URL`
- `SUPABASE_ANON_KEY`
