# Setup – Questionario Etilometro Safe Night Piemonte

## Panoramica

Applicazione web composta da due pagine:
- `index.html` — form pubblico per la raccolta dati etilometro sul campo
- `admin.html` — pannello admin protetto da login per visualizzare, filtrare ed esportare le risposte

I dati vengono salvati su Supabase.

---

## 1. Supabase

**Progetto esistente**
- Project ID: `zotrurzfsaerabpqgjiq`
- Region: Frankfurt (eu-central-1)
- Project URL: `https://zotrurzfsaerabpqgjiq.supabase.co`

### Tabella creata

```sql
create table if not exists etilometro_risposte (
  id                      uuid        default gen_random_uuid() primary key,
  creato_il               timestamptz default now(),
  data_caricamento        date,
  asl                     text,
  data_evento             date,
  luogo_tipo              text,
  luogo_nome              text,
  tempo_da_bere           text,
  genere                  text,
  sesso_assegnato         text,
  eta                     text,
  patente                 text,
  patente_ritirata        text,
  trasporto_prima         text,
  trasporto_prima_altro   text,
  valore_atteso_range     text,
  valore_atteso_preciso   text,
  valore_rilevato_range   text,
  valore_rilevato_preciso text,
  trasporto_dopo          text,
  trasporto_dopo_altro    text,
  tipo_risposta           text,
  stima_consumo           integer,
  bevute_birra            integer,
  bevute_vino             integer,
  bevute_shot             integer,
  bevute_aperitivo        integer,
  ha_mangiato             text,
  ha_mangiato_da_quanto   text,
  sostanze                text[],
  sostanze_altro          text
);
```

### Row Level Security (RLS)

RLS abilitato sulla tabella. Tre ruoli distinti:

```sql
-- Chiunque può inserire dal form pubblico
create policy "Consenti insert anonimi"
on etilometro_risposte
for insert
to anon
with check (true);

-- Solo utenti autenticati (admin) possono leggere i dati
create policy "Solo utenti autenticati possono leggere"
on etilometro_risposte
for select
to authenticated
using (true);
```

| Operazione | anon (form pubblico) | authenticated (admin) |
|-----------|---------------------|----------------------|
| INSERT    | ✓                   | ✓                    |
| SELECT    | ✗                   | ✓                    |
| UPDATE    | ✗                   | ✗                    |
| DELETE    | ✗                   | ✗                    |

### Autenticazione admin

Gli utenti admin si gestiscono da **Supabase → Authentication → Users**.  
Per aggiungere un nuovo operatore admin: **"Add user" → "Create new user"** con email e password.

> La anon key è pubblica per design (Supabase la espone lato client). La protezione dei dati è garantita da RLS, non dalla segretezza della chiave.

---

## 2. File dell'applicazione

### `index.html` — Form pubblico

Accessibile a tutti, nessun login richiesto. Usato dagli operatori sul campo.

Configurazione Supabase nel file:
```js
const SUPABASE_URL = 'https://zotrurzfsaerabpqgjiq.supabase.co';
const SUPABASE_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...';
const TABLE_NAME   = 'etilometro_risposte';
```

### `admin.html` — Pannello admin

Accessibile solo dopo login con email/password (Supabase Auth).

Funzionalità:
- **Tabella** con tutte le schede (50 per pagina, paginazione)
- **Filtri** per ASL, intervallo date evento, tipo risposta (base/completa)
- **Export CSV** — scarica tutte le schede filtrate, con BOM UTF-8 per compatibilità Excel
- **Logout**

---

## 3. Repository GitHub

- **URL:** https://github.com/EmanuelSteadycam/SafeNightPiemonte
- **Branch principale:** `main`
- **Contenuto:**
  - `index.html` — il questionario completo
  - `admin.html` — pannello admin
  - `create_table.sql` — lo schema SQL della tabella
  - `.gitignore` — esclude `.vercel` e `.DS_Store`
  - `SETUP.md` — questo file

---

## 4. Deploy Vercel

- **Form pubblico:** https://safenightpiemonte.vercel.app
- **Pannello admin:** https://safenightpiemonte.vercel.app/admin.html
- **Progetto Vercel:** `safenightpiemonte`
- **Team:** `centro-steadycams-projects`
- **Tipo:** sito statico (nessun build step)
- **CI/CD:** collegato al repo GitHub — ogni push su `main` fa un deploy automatico in produzione

---

## Architettura

```
Operatore (campo)          Admin
      │                      │
      │  INSERT (anon key)   │  SELECT (email + password)
      ▼                      ▼
https://safenightpiemonte.vercel.app
index.html                admin.html
      │                      │
      └──────────┬───────────┘
                 │  Supabase JS SDK
                 ▼
    Supabase – tabella etilometro_risposte
         (RLS: insert=anon, select=auth)
```

---

## Operazioni future

| Operazione | Come farlo |
|-----------|------------|
| Vedere le risposte | https://safenightpiemonte.vercel.app/admin.html |
| Esportare i dati in CSV | Admin panel → "Esporta CSV" (rispetta i filtri attivi) |
| Aggiungere un utente admin | Supabase → Authentication → Users → Add user |
| Aggiornare il questionario | Modifica `index.html`, fai `git push` — Vercel rideploya automaticamente |
| Aggiungere una colonna | SQL Editor Supabase: `alter table etilometro_risposte add column ...` |
| Rollback a versione precedente | Vercel Dashboard → Deployments → scegli un deploy precedente → Promote |
