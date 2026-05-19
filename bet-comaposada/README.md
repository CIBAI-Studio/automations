# RRSS Publisher — Bet Comaposada

## Descripció

Automatització N8N per a la publicació programada de continguts a Instagram i Facebook per al client **Bet Comaposada** (fisioterapeuta neurològica, Berguedà).

## Trigger

- **Tipus:** Schedule (Cron)
- **Freqüència:** Cada hora
- **Lògica:** Llegeix el Google Sheet, filtra files amb `Estado = programat` i `Fecha/Hora` coincident amb l'hora actual, i publica.

## Nodos principals

| Nodo | Tipus | Descripció |
|------|-------|------------|
| Cron — Cada hora | ScheduleTrigger | Disparador cada hora |
| Llegir Google Sheet | googleSheets | Llegeix files amb `Estado = programat` |
| Filtre Data/Hora | filter | Filtra per data i hora actuals |
| Router RRSS | switch | Ruta per plataforma (instagram/facebook) |
| Instagram — Crear Container | httpRequest | Crea container via API Meta |
| Instagram — Publicar | httpRequest | Publica el container creat |
| Facebook — Publicar | httpRequest | Publica directament al feed |
| Actualitzar Estado: publicat | googleSheets | Marca la fila com publicat |
| Actualitzar Estado: error | googleSheets | Marca la fila com error |
| Notificar Error CM | emailSend | Envia email al CM si hi ha errors |

## Variables d'entorn requerides

```env
BET_COMAPOSADA_SHEET_ID=<ID del Google Sheet de Bet Comaposada>
BET_INSTAGRAM_ACCOUNT_ID=<ID compte Instagram de Bet>
BET_INSTAGRAM_ACCESS_TOKEN=<Token accés Instagram Graph API>
BET_FACEBOOK_PAGE_ID=<ID pàgina Facebook de Bet>
BET_FACEBOOK_ACCESS_TOKEN=<Token accés Facebook Graph API>
GOOGLE_SHEETS_CREDENTIAL_ID=<ID credencial N8N per Google Sheets>
SMTP_CREDENTIAL_ID=<ID credencial SMTP per notificacions>
CM_EMAIL=<Email del Community Manager>
```

## Estructura del Google Sheet

El Google Sheet ha de tenir les columnes en aquest ordre (fila 1 = capçaleres):

| ID | Fecha | Hora | Red Social | Tipo | Caption | Hashtags | URL Media | URL Link | Estado | Notas |
|----|-------|------|-----------|------|---------|----------|-----------|----------|--------|-------|
| BET-JUN26-001 | 2026-06-03 | 19:00 | instagram | reel | ... | ... | | | programat | ... |

**Valors vàlids per `Estado`:** `programat` → `publicat` / `error`

**Valors vàlids per `Red Social`:** `instagram`, `facebook`

**Valors vàlids per `Tipo`:** `reel`, `imagen`, `carrusel`, `story`

## Calendari juny 2026 (importat)

20 files preparades per al mes de juny 2026. Veure `bet-juny-2026-sheet.csv`.

- Primera publicació: **3 juny 2026 a les 19:00**
- 12 peces × plataformes IG/FB = 20 files totals

## Relació amb altres issues

- [CIBA-1384](/CIBA/issues/CIBA-1384) — Programació publicació (SOCIALMEDIA)
- [CIBA-1390](/CIBA/issues/CIBA-1390) — Configuració Google Sheet + importació CSV (AUTOMATIZACIONES)
- [CIBA-1391](/CIBA/issues/CIBA-1391) — Assets visuals (DESIGNER)
