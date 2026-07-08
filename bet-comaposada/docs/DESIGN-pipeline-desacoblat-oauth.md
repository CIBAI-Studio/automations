# Disseny — Pipeline de publicació Bet Comaposada desacoblat de Google Drive/OAuth

**Ticket:** CIBA-2191 · **Autor:** CTO (CIBAI Studio) · **Data:** 2026-07-08
**Estat:** Disseny per aprovació · **NO executar build fins decisió del Board (OAuth vs. pivot)**

---

## 1. Diagnòstic del bloqueig

La cadena de publicació porta bloquejada 7 setmanes (des del 19/05) per **un únic punt de fallada**: el connector *claude.ai Google Drive* mai es va autoritzar via OAuth (pas manual que només pot fer Matías). Sense això, el Google Sheet font (CIBA-1392) no es va crear i la importació del CSV (CIBA-1390) no es va poder completar.

**Descobriment clau (verificat inspeccionant el flow N8N real `bet-comaposada/rrss-bet-comaposada-publisher.n8n.json`):**

El Google Sheet només s'usa a **la capa de dades** (font + registre d'estat). La **publicació a Instagram/Facebook és 100% independent de Google**: es fa amb nodes `httpRequest` contra la Meta Graph API, autenticats amb tokens de Meta (`BET_INSTAGRAM_ACCESS_TOKEN`, `BET_FACEBOOK_ACCESS_TOKEN`), no amb credencials de Google.

| Node del flow actual | Depèn de Google? | Acció al disseny |
|---|---|---|
| Cron — Cada hora | No | Es manté |
| Llegir Google Sheet | **Sí (bloqueig)** | **Substituir** per lectura del CSV a GitHub |
| Filtre Data/Hora | No | Es manté |
| Router RRSS | No | Es manté |
| Instagram — Crear Container / Publicar | No | Es manté |
| Facebook — Publicar | No | Es manté |
| Actualitzar Estado: publicat/error | **Sí (bloqueig)** | **Substituir** per state store propi |
| Notificar Error CM (email) | No | Es manté |

Conclusió: **eliminant Google de la capa de dades desbloquegem tota la cadena**. No cal tocar la lògica de publicació.

---

## 2. Opcions avaluades

| # | Font de dades | Registre d'estat | Veredicte |
|---|---|---|---|
| **A (recomanada)** | CSV raw a GitHub (ja existent) llegit per HTTP Request | Store propi de N8N (taula Postgres / Data Store intern) amb guarda d'idempotència | **Robusta, cost zero, sense OAuth, sense escriptura de retorn a GitHub** |
| B | CSV raw a GitHub | Commit de retorn al CSV via GitHub API (PAT) | Descartada: escriptura per publicació genera commits sorollosos, condicions de cursa i depèn del PAT (bloqueig recurrent conegut de scope de token) |
| C | Google Sheet via **service account** (en lloc d'OAuth interactiu) | Sheet | Viable tècnicament però reintrodueix Google, requereix crear projecte GCP + clau de service account + compartir el Sheet. Més superfície i temps que A. Només si el client exigeix Sheets com a superfície d'edició |

**Decisió: Opció A.** Separa neta la *intenció* (CSV editat per humans) de l'*estat* (escrit per la màquina). No escriu mai de tornada a la font → sense conflictes ni credencials d'escriptura.

---

## 3. Arquitectura escollida (Opció A)

```
                 ┌──────────────────────────────────────────────┐
                 │  GitHub · CIBAI-Studio/automations            │
                 │  bet-comaposada/bet-<mes>-2026-sheet.csv      │  ← FONT (editada per SOCIALMEDIA)
                 └───────────────┬──────────────────────────────┘
                                 │ HTTPS GET (raw URL, públic)
                                 ▼
   ┌─────────┐   ┌────────────────────┐   ┌──────────────────────────────┐
   │  Cron   │──▶│  HTTP Request       │──▶│  Code: parse CSV +           │
   │ 1h      │   │  (fetch CSV raw)    │   │  join amb state store        │
   └─────────┘   └────────────────────┘   │  + filtre Estado=programat   │
                                           │  + filtre Fecha/Hora == ara  │
                                           │  + descartar ja publicat     │
                                           └───────────────┬──────────────┘
                                                           ▼
                                                 ┌──────────────────┐
                                                 │  Router RRSS      │
                                                 └───┬──────────┬────┘
                                          instagram  │          │  facebook
                                                     ▼          ▼
                                        ┌──────────────┐  ┌──────────────┐
                                        │ IG container │  │ FB publicar  │  ← Meta Graph API
                                        │ + publicar   │  │ (sense canvi)│     (tokens Meta)
                                        └──────┬───────┘  └──────┬───────┘
                                               └─────┬───────────┘
                                                     ▼
                                     ┌───────────────────────────────┐
                                     │  State store (N8N Postgres)   │  ← ESTAT (escrit per màquina)
                                     │  publicat / error + timestamp  │
                                     └───────────────┬───────────────┘
                                                     │ error
                                                     ▼
                                             ┌──────────────┐
                                             │ Email CM     │
                                             └──────────────┘
```

### 3.1 Font de dades
- Node **HTTP Request** (GET) contra la raw URL pública:
  `https://raw.githubusercontent.com/CIBAI-Studio/automations/main/bet-comaposada/bet-<mes>-2026-sheet.csv`
- Node **Code** (o *Spreadsheet File*) parseja el CSV a JSON. Columnes idèntiques a l'estructura actual (`ID, Fecha, Hora, Red Social, Tipo, Caption, Hashtags, URL Media, URL Link, Estado, Notas`) → **no cal re-adaptar el contingut ni els assets** (ja són raw URLs de GitHub).
- Cap credencial. La raw URL és pública i cacheja ~5 min (marge suficient per a un cron horari).

### 3.2 Registre d'estat (state store)
- Taula al **Postgres intern de N8N** (o *Data Store* natiu de N8N si la versió el suporta), clau = `ID + Red Social`:
  ```
  publication_state(piece_id, platform, status, published_at, error_msg, updated_at)
  status ∈ {publicat, error}
  ```
- Després de publicar amb èxit → INSERT/UPSERT `publicat`. Si falla → `error` + missatge, i s'activa el node d'email al CM.
- **No s'escriu mai al CSV.** La font queda immutable respecte a la màquina.

### 3.3 Idempotència (crític)
Com que la font (CSV) i l'estat (store) estan separats, el node Code fa un **anti-join**: publica una fila només si `Estado=programat` **I** `Fecha/Hora == hora actual` **I** `(ID, Red Social)` **NO** existeix ja com a `publicat` al store. Això garanteix que:
- No es dupliquen publicacions encara que el cron s'executi diverses vegades.
- Un `error` es pot reintentar (no queda com a `publicat`).
Aquest guard és **més robust** que l'actual (reescriure Estado al Sheet), que era vulnerable a fallades entre publicar i escriure de tornada.

### 3.4 Com edita/programa SOCIALMEDIA sense Google Sheets
- **Superfície d'edició: l'editor web de GitHub** sobre el fitxer CSV. Per a un CM no-tècnic és equivalent a un full de càlcul senzill, i aporta gratis: historial de canvis, autoria i *diff* per commit (auditoria millor que un Sheet).
- Fluxos:
  - **Programar** una peça → editar la fila: posar `Fecha`, `Hora` i `Estado=programat`, *commit*.
  - **Pausar/treure de cua** → canviar `Estado` a buit o `draft`.
  - **Corregir un caption** → editar la cel·la i *commit* (només afecta files encara no publicades).
- **Mes nou (juliol, agost…)** → duplicar el CSV a `bet-<mes>-2026-sheet.csv` i apuntar la variable de mes del flow (o un únic CSV multi-mes; el filtre per data ja discrimina).
- *Upgrade opcional futur* (no MVP): un editor CSV hostejat (p. ex. una pàgina estàtica amb commit via GitHub API) si el CM ho demana. No necessari per desbloquejar juliol.

---

## 4. Prerequisit independent a verificar (2n possible bloqueig)

El desacoblament de Google **no** cobreix la capa de publicació. Perquè surti contingut real cal que existeixin i siguin vàlids:
- Compte **Instagram Business** de Bet vinculat a una pàgina de Facebook.
- Tokens Meta Graph API vius: `BET_INSTAGRAM_ACCOUNT_ID`, `BET_INSTAGRAM_ACCESS_TOKEN`, `BET_FACEBOOK_PAGE_ID`, `BET_FACEBOOK_ACCESS_TOKEN`.

Aquests els proporciona el client i **ja eren requisit del pla original** (cost zero). **Acció recomanada:** AUTOMATIZACIONES fa un *test post* a un compte de proves o en mode sandbox abans del primer live. Si els tokens no existeixen, és un bloqueig de client separat que s'ha d'escalar en paral·lel — no depèn d'aquest disseny.

---

## 5. Estimació d'esforç (build del pipeline desacoblat)

| Tasca | Rol | Esforç |
|---|---|---|
| Substituir node *Llegir Google Sheet* per HTTP Request + Code (parse CSV) | AUTOMATIZACIONES | 3–4 h |
| Muntar state store (taula Postgres N8N) + nodes read/upsert | AUTOMATIZACIONES | 1–2 h |
| Lògica d'idempotència (anti-join) + substituir els 2 nodes *Actualitzar Estado* | AUTOMATIZACIONES | 2–3 h |
| Preparar CSV de juliol (contingut ja fet a CIBA-2121, només format) | AUTOMATIZACIONES / SOCIALMEDIA | ~1 h |
| Dry-run end-to-end + 1 test post real IG/FB | AUTOMATIZACIONES + QA | 2–3 h |
| Actualitzar README + doc del flow | AUTOMATIZACIONES | 1 h |
| **Total** | | **~10–14 h (1,5–2 dies-dev) + QA gate** |

Ruta crítica per treure juliol: si el Board aprova avui/demà, **desbloqueig i primeres publicacions dins de 2–3 dies laborables**.

---

## 6. Confirmacions (criteri d'acceptació 3)

- **Aïllament / no contamina producció d'altres projectes:** el canvi toca únicament (a) la carpeta `bet-comaposada/` del repo `CIBAI-Studio/automations`, (b) un workflow N8N dedicat a Bet, i (c) una taula d'estat pròpia d'aquest workflow. Cap dada d'altres clients es llegeix ni s'escriu.
- **Sense aprovació de pressupost:** cost zero. GitHub raw (gratis), N8N (instància existent), Meta Graph API (gratis). No s'introdueix cap servei de pagament. *(Opció C amb service account tampoc tindria cost, però afegeix superfície GCP; per això es descarta.)*
- **Sense secrets nous:** no s'afegeixen credencials de Google. Es reutilitzen els tokens Meta ja previstos al pla original.

---

## 7. Recomanació al Board

**Pivotar a l'Opció A i abandonar la dependència del Google Drive OAuth de claude.ai per a aquest pipeline.** Raons:
1. Elimina definitivament l'únic punt de bloqueig manual que porta 7 setmanes aturat.
2. Cost zero, sense pressupost, sense secrets nous.
3. Guard d'idempotència més robust que el disseny original amb Sheet.
4. Superfície d'edició (GitHub web) amb auditoria superior a un Sheet.

Completar l'OAuth continua sent vàlid si el client vol Google Sheets com a eina d'edició pròpia, però **no hauria de ser bloqueig de la publicació**: aquest disseny permet publicar independentment i, si més endavant es completa l'OAuth, es pot afegir Sheets com a superfície addicional sense refer el pipeline.

**No s'executa cap build fins que el Board decideixi (completar OAuth vs. pivotar).** Amb aprovació, obro els child issues a AUTOMATIZACIONES i QA segons l'estimació §5.
