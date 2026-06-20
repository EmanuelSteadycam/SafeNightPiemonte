# Rete Safe Night Piemonte — Documentazione

## Panoramica

Applicazione web per la raccolta dati sul campo composta da quattro pagine:
- `index.html` — form pubblico etilometro
- `scheda-uscite.html` — form pubblico scheda uscite
- `scheda-osservativa.html` — form pubblico scheda osservativa
- `admin.html` — pannello admin protetto da login

I form sono accessibili pubblicamente (senza login). I dati vengono salvati su Supabase in tre tabelle separate. L'admin panel richiede login e mostra solo i dati in base al ruolo dell'utente.

---

## 1. Supabase

**Progetto**
- Project ID: `zotrurzfsaerabpqgjiq`
- Region: Frankfurt (eu-central-1)
- Project URL: `https://zotrurzfsaerabpqgjiq.supabase.co`
- Anon key: Supabase → Settings → API

### Tabelle

#### `etilometro_risposte`

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

#### `scheda_uscite`

Contiene i dati delle uscite sul campo: contatti, evento, materiali distribuiti (inclusi materiali di conforto), volantini, questionari compilati.

Colonne principali: `data_caricamento`, `asl`, `data_evento`, `luogo_tipo/nome`, `contatti`, `denominazione_evento`, `tipologia_evento`, `mat_*` (22 campi), `am_*` (6 campi altri materiali), `vol_*` (27 campi volantini), `q_etilometro`, `q_sostanze`, `note`.

#### `scheda_osservativa`

Contiene le osservazioni sul contesto. I campi a scelta multipla sono salvati come `text[]`.

Colonne principali: `data_caricamento`, `asl`, `data_evento`, `luogo_*`, `denominazione_evento`, `ora_inizio`, `ora_fine`, `variazioni_orario`, `eta[]`, `stile[]`, `interventi[]`, `rischi[]`, `pratiche[]`, `tematiche[]`, `organizzazione[]`, `note`.

---

### Row Level Security (RLS)

**Regola critica per INSERT:**

> NON usare `to anon` nelle policy INSERT — usare sempre `for insert with check (true)` senza specificare il ruolo. Specificare `to anon` causa un errore RLS silenzioso (403) in Supabase/PostgREST.

**Policy complete per ogni tabella:**

```sql
-- INSERT: aperto a tutti (form pubblico, senza to anon)
create policy "allow_insert" on <tabella>
for insert with check (true);

-- SELECT: admin vede tutto, utente ASL vede solo la propria ASL
create policy "allow_select" on <tabella>
for select to authenticated using (
  coalesce(auth.jwt()->'app_metadata'->>'role','admin') = 'admin'
  OR (
    auth.jwt()->'app_metadata'->>'role' = 'asl'
    AND asl = auth.jwt()->'app_metadata'->>'asl'
  )
);

-- DELETE: solo admin
create policy "allow_delete" on <tabella>
for delete to authenticated using (
  coalesce(auth.jwt()->'app_metadata'->>'role','admin') = 'admin'
);

-- Grant necessari
grant insert on <tabella> to anon;
grant select, delete on <tabella> to authenticated;
```

| Operazione | Anon (form pubblico) | Admin | Utente ASL |
|-----------|----------------------|-------|-----------|
| INSERT    | ✓                    | ✓     | ✓         |
| SELECT    | ✗                    | ✓ tutti | ✓ solo sua ASL |
| DELETE    | ✗                    | ✓     | ✗         |
| UPDATE    | ✗                    | ✗     | ✗         |

---

### Ruoli utenti

Gli utenti del pannello admin si dividono in due ruoli gestiti tramite `app_metadata` nel JWT:

| Ruolo | `app_metadata` | Comportamento admin |
|-------|---------------|---------------------|
| Admin | `{"role":"admin"}` | Vede tutti i record, può eliminare, ha filtro ASL |
| ASL   | `{"role":"asl","asl":"ASL CN2"}` | Vede solo i propri record, nessun bottone elimina, filtro ASL nascosto |

**Chi non ha `role` impostato viene trattato come admin (retrocompatibilità).**

#### Aggiungere un utente ASL (procedura completa)

1. **Crea utente** in Supabase → Authentication → Users → Add user
   - Email consigliata: `aslXX@safenightpiemonte.it`
   - Disabilita "Send invite" se disponibile

2. **Conferma manualmente** (le email finte non ricevono link di conferma):
```sql
update auth.users
set email_confirmed_at = now()
where email = 'aslxx@safenightpiemonte.it';
```

3. **Assegna ruolo e ASL** (il valore di `asl` deve corrispondere esattamente a quello nei form):
```sql
update auth.users
set raw_app_meta_data = raw_app_meta_data || '{"role":"asl","asl":"ASL XX"}'::jsonb
where email = 'aslxx@safenightpiemonte.it';
```

4. L'utente fa login → vede automaticamente solo i record della propria ASL.

#### Utenti attivi

| Email | ASL |
|-------|-----|
| `aslal@safenightpiemonte.it` | ASL AL |
| `aslat@safenightpiemonte.it` | ASL AT |
| `aslbi@safenightpiemonte.it` | ASL BI |
| `aslcn1@safenightpiemonte.it` | ASL CN1 |
| `aslcn2@safenightpiemonte.it` | ASL CN2 |
| `aslno@safenightpiemonte.it` | ASL NO |
| `aslcitt@safenightpiemonte.it` | ASL Città di Torino |
| `aslto3@safenightpiemonte.it` | ASL TO3 |
| `aslto4@safenightpiemonte.it` | ASL TO4 |
| `aslto5@safenightpiemonte.it` | ASL TO5 |
| `aslvc@safenightpiemonte.it` | ASL VC |
| `aslvco@safenightpiemonte.it` | ASL VCO |
| `neutravel@safenightpiemonte.it` | NEUTRAVEL |
| `emanuel@progettosteadycam.it` | **Admin** (accesso completo) |

---

## 2. File dell'applicazione

### `index.html` — Etilometro (form pubblico)

Raccolta dati etilometro sul campo. Sfondo blu con pattern doodle (`bg-etilometro.png`). Sezioni: dati base (ASL, luogo, trasporto, valore rilevato) e approfondimento (bevande, sostanze). Validazione in tempo reale sul campo "Valore rilevato preciso" (formato `0,00`).

### `scheda-uscite.html` — Scheda Uscite (form pubblico)

Raccolta dati uscite operative. Sfondo verde con pattern doodle (`bg-uscite.png`). Sezioni:
1. **Contatti** — data, ASL, luogo, numero contatti totali
2. **Evento** — denominazione, tipologia (Club, Street Parade, TAZ, ecc.)
3. **Materiali** — 22 voci con quantità numeriche
4. **Altri materiali** — materiali di conforto (Acqua, Succhi, Caramelle, ecc.)
5. **Volantini** — 27 voci
6. **Questionari** — Etilometro + Sostanze, note

### `scheda-osservativa.html` — Scheda Osservativa (form pubblico)

Raccolta dati osservativi. Sfondo beige con pattern doodle (`bg-osservativa.png`). Sezioni:
1. **Contatti** — data, ASL, luogo, evento, ora inizio/fine (`HH,MM` con auto-completamento), variazioni orario
2. **Soggetti presenti** — fasce d'età, genere prevalente, stili
3. **Interventi e rischi** — checkbox multipli
4. **Gruppi** — caratteristiche, dimensione, genere
5. **Pratiche** — checkbox multipli
6. **Esigenze** — tematiche, organizzazione, note

Auto-completamento orari su blur: `21` → `21,00` / `8,` → `08,00` / `2,30` → `02,30`

### `admin.html` — Pannello admin

Accessibile solo dopo login. Lo sfondo cambia colore in base alla tab attiva (blu/verde/beige).

**Tre tab:** Etilometro / Scheda Uscite / Scheda Osservativa

Funzionalità:
- Tabella con 100 schede per pagina, paginazione
- Filtri per ASL (nascosto per utenti ASL) e intervallo date
- Filtro tipo risposta (solo Etilometro)
- Bottone Elimina (solo admin)
- Export CSV con BOM UTF-8 (compatibile Excel)

### File statici sfondo
- `bg-etilometro.png` — sfondo blu doodle per Etilometro e tab admin
- `bg-uscite.png` — sfondo verde doodle per Scheda Uscite e tab admin
- `bg-osservativa.png` — sfondo beige doodle per Scheda Osservativa e tab admin

---

## 3. Repository GitHub

- **URL:** https://github.com/EmanuelSteadycam/SafeNightPiemonte
- **Branch principale:** `main`
- **CI/CD:** ogni push su `main` → deploy automatico Vercel in ~15 secondi

---

## 4. Deploy Vercel

| Pagina | URL |
|--------|-----|
| Etilometro | https://safenightpiemonte.vercel.app/index.html |
| Scheda Uscite | https://safenightpiemonte.vercel.app/scheda-uscite.html |
| Scheda Osservativa | https://safenightpiemonte.vercel.app/scheda-osservativa.html |
| Admin | https://safenightpiemonte.vercel.app/admin.html |

- **Progetto Vercel:** `safenightpiemonte` (team `centro-steadycams-projects`)
- **Tipo:** sito statico, nessun build step

---

## Architettura

```
Operatori (campo)             Utenti ASL              Admin
       │                          │                     │
       │ INSERT (anon)            │ SELECT (propria ASL)│ SELECT + DELETE (tutto)
       ▼                          ▼                     ▼
┌──────────────────────────────────────────────────────────────┐
│               safenightpiemonte.vercel.app                   │
│   index.html   scheda-uscite.html   scheda-osservativa.html  │
│                        admin.html                            │
└────────────────────────────┬─────────────────────────────────┘
                             │  Supabase JS SDK v2
                             ▼
               Supabase (zotrurzfsaerabpqgjiq)
          ┌──────────────────────────────────────┐
          │  etilometro_risposte                 │
          │  scheda_uscite                       │
          │  scheda_osservativa                  │
          └──────────────────────────────────────┘
           RLS: insert=tutti | select/delete=per ruolo
```

---

## Operazioni comuni

| Operazione | Come farlo |
|-----------|------------|
| Vedere le schede | https://safenightpiemonte.vercel.app/admin.html |
| Esportare CSV | Admin → seleziona tab → "Esporta CSV" |
| Aggiungere utente ASL | Vedi procedura in sezione Ruoli utenti |
| Aggiungere utente admin | Supabase → Auth → Users → Add user, poi SQL: `set raw_app_meta_data = raw_app_meta_data \|\| '{"role":"admin"}'::jsonb` |
| Aggiornare un form | Modifica HTML → `git push` → Vercel rideploya |
| Aggiungere una colonna | SQL Editor: `alter table <tabella> add column ...` |
| Rollback | Vercel Dashboard → Deployments → deploy precedente → Promote |
| Fix errore INSERT 403 | Drop + recreate policy INSERT senza `to anon`: `create policy "allow_insert" on <t> for insert with check (true)` |
