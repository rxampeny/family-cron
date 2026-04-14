# Hola Família — App de compleanys familiars

App web en català per gestionar els compleanys de la família. Envia felicitacions automàtiques per email i SMS cada matí.

**URL:** https://hola-familia.netlify.app
**GitHub:** `rxampeny/family-cron` → Netlify auto-desplega en push a `main`

---

## Què fa

- Llista tots els membres de la família amb les seves dates de cumpleaños
- Permet afegir, editar i eliminar persones
- Envia automàticament cada matí:
  - **Email + SMS de felicitació** directament al cumpleanyero (4:00 AM)
  - **Avís a tota la família** del cumpleaños de l'endemà (6:00 AM)

---

## Arquitectura

```
GitHub Actions (cron) → Supabase Edge Functions → PostgreSQL + Gmail + Twilio
Usuari → Netlify (index.html) → Supabase Edge Functions → PostgreSQL
```

| Peça | Servei | Detall |
|------|--------|--------|
| Frontend | Netlify | `index.html` — SPA tot en un fitxer |
| Backend | Supabase Edge Functions | Deno/TypeScript, project ref `bfforytzxicguykosnna` |
| Base de dades | Supabase PostgreSQL | Taula principal `personas` |
| Emails | Gmail OAuth2 | Felicitacions i recordatoris |
| SMS | Twilio | Felicitacions i recordatoris |
| Crons | GitHub Actions | `.github/workflows/cron-emails.yml` |

---

## Fitxers del repo

```
readme.md                           → aquest fitxer
index.html                          → SPA completa (HTML + CSS + JS en català)
supabase-client.js                  → client per cridar les Edge Functions
netlify.toml                        → configuració Netlify (redirecció SPA)
package.json                        → deps Netlify CLI
public/                             → fotos i icones de la família
.github/workflows/cron-emails.yml   → GitHub Actions: emails i SMS diaris
```

---

## Crons automàtics (GitHub Actions)

| Hora (Madrid) | UTC | Edge Function | Acció |
|---------------|-----|---------------|-------|
| 04:00 AM | `0 3 * * *` | `send-birthday-emails` | Email de felicitació al cumpleanyero |
| 04:00 AM | `0 3 * * *` | `send-birthday-sms` | SMS de felicitació al cumpleanyero |
| 06:00 AM | `0 5 * * *` | `send-reminders` | Avisa tota la família del cumpleaños de demà |

Els crons també es poden llançar manualment des de GitHub Actions → `workflow_dispatch`.

---

## Edge Functions (Supabase)

| Funció | Descripció |
|--------|------------|
| `get-personas` | Llista totes les persones |
| `create-persona` | Afegeix una persona (detecta duplicats) |
| `update-persona` | Modifica una persona |
| `delete-persona` | Arxiva una persona (soft delete) |
| `send-birthday-emails` | Envia emails de felicitació (cron) |
| `send-birthday-sms` | Envia SMS de felicitació (cron) |
| `send-reminders` | Envia recordatoris del dia anterior (cron) |

Per desplegar una funció:
```bash
npx supabase functions deploy <nom-funcio> --project-ref bfforytzxicguykosnna
```

---

## Base de dades (PostgreSQL)

Taules principals:
- `personas` — membres de la família (nom, dia, mes, any, email, telèfon, foto...)
- `personas_eliminadas` — arxiu de persones esborrades (soft delete)
- `log_emails` — historial d'emails enviats
- `log_sms` — historial de SMS enviats
- `configuracion` — configuració (manteniment, email_actiu, sms_actiu)

Camp especial: `enviar_recordatoris` (boolean, default `true`) — si és `false`, la família no rep avís quan aquesta persona tingui el cumpleaños. Només editable des de Supabase.

---

## Secrets necessaris

**GitHub Actions** (Settings → Secrets):
- `SUPABASE_URL`
- `SUPABASE_ANON_KEY`

**Supabase Edge Functions** (Dashboard → Settings → Secrets):
- `GMAIL_CLIENT_ID`, `GMAIL_CLIENT_SECRET`, `GMAIL_REFRESH_TOKEN`, `GMAIL_FROM`
- `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_FROM_NUMBER`
