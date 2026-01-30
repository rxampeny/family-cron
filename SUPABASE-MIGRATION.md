# Guía de Migración a Supabase

## Resumen

Esta guía explica cómo migrar la aplicación "Aniversaris Familiars" de Google Sheets + Google Apps Script a Supabase.

## Estructura de Archivos Creados

```
supabase/
├── config.toml                    # Configuración del CLI de Supabase
├── .env.example                   # Variables de entorno de ejemplo
├── migrations/
│   ├── 001_initial_schema.sql     # Esquema de base de datos
│   └── 002_cron_jobs.sql          # Documentación de cron jobs
└── functions/
    ├── _shared/
    │   ├── cors.ts                # Utilidades CORS
    │   ├── supabase.ts            # Cliente Supabase compartido
    │   ├── email.ts               # Utilidades de email (Resend)
    │   └── sms.ts                 # Utilidades de SMS (Twilio)
    ├── get-personas/index.ts      # GET: Obtener todas las personas
    ├── create-persona/index.ts    # POST: Crear persona
    ├── update-persona/index.ts    # POST: Actualizar persona
    ├── delete-persona/index.ts    # POST: Eliminar persona
    ├── send-birthday-emails/      # POST: Enviar emails de cumpleaños
    ├── send-birthday-sms/         # POST: Enviar SMS de cumpleaños
    ├── send-reminders/            # POST: Enviar recordatorios
    └── chat/index.ts              # POST: Chat con IA (Claude)

scripts/
└── migrate-to-supabase.js         # Script de migración de datos

supabase-client.js                 # Cliente JS para el frontend
```

---

## Paso 1: Crear Proyecto en Supabase

1. Ve a [supabase.com](https://supabase.com) y crea una cuenta
2. Crea un nuevo proyecto
3. Guarda estos datos (los necesitarás):
   - **Project URL**: `https://xxxxx.supabase.co`
   - **Anon Key**: Para el frontend
   - **Service Role Key**: Para las Edge Functions

---

## Paso 2: Crear la Base de Datos

1. Ve al **SQL Editor** en el dashboard de Supabase
2. Copia y pega el contenido de `supabase/migrations/001_initial_schema.sql`
3. Ejecuta el SQL

Esto creará:
- Tabla `personas` (datos principales)
- Tabla `personas_eliminadas` (log de eliminados)
- Tabla `log_emails` (historial de emails)
- Tabla `log_sms` (historial de SMS)
- Tabla `configuracion` (configuración del sistema)
- Funciones útiles (cumpleaños de hoy/mañana, detección de duplicados, etc.)

---

## Paso 3: Configurar Variables de Entorno

En el dashboard de Supabase, ve a **Settings > Edge Functions** y añade:

```
RESEND_API_KEY=re_xxxxxxxxx
FROM_EMAIL=Família <noreply@tudominio.com>
TWILIO_ACCOUNT_SID=ACxxxxxxxx
TWILIO_AUTH_TOKEN=xxxxxxxx
TWILIO_PHONE_NUMBER=+34600000000
ANTHROPIC_API_KEY=sk-ant-xxxxxxxx
```

Las variables `SUPABASE_URL`, `SUPABASE_ANON_KEY` y `SUPABASE_SERVICE_ROLE_KEY` se configuran automáticamente.

---

## Paso 4: Desplegar Edge Functions

### Opción A: Usando Supabase CLI (Recomendado)

1. Instala el CLI:
   ```bash
   npm install -g supabase
   ```

2. Login:
   ```bash
   supabase login
   ```

3. Vincula el proyecto:
   ```bash
   cd family-agents-sdk
   supabase link --project-ref tu-project-ref
   ```

4. Despliega todas las funciones:
   ```bash
   supabase functions deploy get-personas
   supabase functions deploy create-persona
   supabase functions deploy update-persona
   supabase functions deploy delete-persona
   supabase functions deploy send-birthday-emails
   supabase functions deploy send-birthday-sms
   supabase functions deploy send-reminders
   supabase functions deploy chat
   ```

### Opción B: Dashboard de Supabase

1. Ve a **Edge Functions** en el dashboard
2. Crea cada función manualmente y copia el código

---

## Paso 5: Migrar Datos desde Google Sheets

### 5.1 Exportar datos de Google Sheets

En Google Apps Script, ejecuta este código:

```javascript
function exportDataAsJSON() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const data = sheet.getDataRange().getValues();
  const headers = data[0];

  const result = [];
  for (let i = 1; i < data.length; i++) {
    if (!data[i][0]) continue;

    const row = {};
    for (let j = 0; j < headers.length; j++) {
      if (headers[j]) {
        // Convertir nombres de columnas
        const key = headers[j]
          .replace('Nom', 'nom')
          .replace('Dia', 'dia')
          .replace('Mes', 'mes')
          .replace('AnyNaixement', 'anyNaixement')
          .replace('Telefon', 'telefon')
          .replace('Email', 'email')
          .replace('Gènere', 'genere')
          .replace('Viu', 'viu');
        row[key] = data[i][j];
      }
    }
    result.push(row);
  }

  console.log(JSON.stringify(result, null, 2));
  return result;
}
```

Guarda el resultado como `scripts/data-export.json`.

### 5.2 Importar a Supabase

```bash
# Configura las variables de entorno
export SUPABASE_URL=https://tu-proyecto.supabase.co
export SUPABASE_SERVICE_ROLE_KEY=tu-service-role-key

# Ejecuta el script
node scripts/migrate-to-supabase.js
```

---

## Paso 6: Actualizar el Frontend

### 6.1 Configurar el cliente

Edita `supabase-client.js`:

```javascript
const SUPABASE_URL = 'https://tu-proyecto.supabase.co';
const SUPABASE_ANON_KEY = 'tu-anon-key-aqui';
```

### 6.2 Incluir en index.html

Añade antes del script principal:

```html
<!-- Antes de </head> o al inicio del <script> principal -->
<script src="supabase-client.js"></script>
```

### 6.3 Eliminar código de Google Apps Script

Elimina o comenta estas líneas en `index.html`:

```javascript
// ELIMINAR ESTO:
const GOOGLE_APPS_SCRIPT_URL = 'https://script.google.com/...';

// Las funciones fetchBirthdays, addBirthdayToSheet, etc.
// ahora vienen de supabase-client.js
```

---

## Paso 7: Configurar Cron Jobs

### Para Plan Gratuito: Usar cron-job.org

1. Crea cuenta en [cron-job.org](https://cron-job.org)

2. Crea 3 trabajos:

**Emails de cumpleaños (9:00 AM):**
- URL: `https://tu-proyecto.supabase.co/functions/v1/send-birthday-emails`
- Método: POST
- Headers: `Authorization: Bearer TU_ANON_KEY`
- Horario: 9:00 (Europe/Madrid)

**SMS de cumpleaños (9:30 AM):**
- URL: `https://tu-proyecto.supabase.co/functions/v1/send-birthday-sms`
- Método: POST
- Headers: `Authorization: Bearer TU_ANON_KEY`
- Horario: 9:30 (Europe/Madrid)

**Recordatorios (20:00):**
- URL: `https://tu-proyecto.supabase.co/functions/v1/send-reminders`
- Método: POST
- Headers: `Authorization: Bearer TU_ANON_KEY`
- Horario: 20:00 (Europe/Madrid)

### Para Plan Pro: Usar pg_cron

Descomenta y configura el código en `002_cron_jobs.sql`.

---

## Paso 8: Actualizar el Chat de IA

El chat ahora usa Claude en lugar de OpenAI. Actualiza las llamadas en el frontend:

```javascript
// Antes (Railway):
const response = await fetch('https://tu-app.railway.app/chat', ...);

// Después (Supabase):
const result = await sendChatMessage(message, files);
```

---

## Verificación Final

- [ ] Base de datos creada con todas las tablas
- [ ] Edge Functions desplegadas y funcionando
- [ ] Datos migrados desde Google Sheets
- [ ] Frontend actualizado con cliente Supabase
- [ ] CRUD de personas funciona
- [ ] Detección de duplicados funciona
- [ ] Emails de cumpleaños se envían
- [ ] SMS de cumpleaños se envían
- [ ] Recordatorios funcionan
- [ ] Chat IA funciona
- [ ] Cron jobs configurados

---

## Troubleshooting

### Error CORS
Verifica que las Edge Functions incluyan los headers CORS correctos.

### Error de autenticación
Verifica que estás usando la clave correcta (anon key para frontend, service role para scripts).

### Emails/SMS no se envían
1. Verifica las credenciales de Resend/Twilio
2. Revisa los logs en el dashboard de Supabase
3. Comprueba la tabla `log_emails` / `log_sms`

### Función no encontrada
Asegúrate de que la función está desplegada:
```bash
supabase functions list
```

---

## Costes Estimados

| Servicio | Plan Gratuito | Notas |
|----------|--------------|-------|
| Supabase | Gratis | 500MB DB, 500K invocaciones/mes |
| Resend | Gratis | 100 emails/día |
| Twilio | ~0.07€/SMS | Sin cambios |
| Anthropic | ~$3/1M tokens | Según uso del chat |

**Total estimado:** Similar al coste actual (~$5/mes + SMS)
