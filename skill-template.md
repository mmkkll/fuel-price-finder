# Fuel Prices & EV Charging — Skill

Quando Mirko chiede il prezzo del carburante, il distributore più economico in zona, **oppure colonnine di ricarica elettrica**, usa questa skill.

## Trigger
- "qual è il distributore meno caro in zona?"
- "prezzo gasolio/benzina"
- "distributore vicino"
- "carburante"
- qualsiasi richiesta su prezzi benzina/gasolio/GPL/metano → modalità **carburante**
- "colonnina", "colonnine", "ricarica elettrica", "EV charging", "punto di ricarica", "stazione di ricarica" → modalità **elettrico**
- **Invio location Telegram**: se Mirko manda una posizione (location pin) senza testo, default = 3 distributori gasolio self più economici. Se il testo dice "elettrica/colonnina", usa la modalità EV. Se dice "benzina/GPL/metano", usa quel carburante.

## Fonte dati
Il Ministero delle Imprese e del Made in Italy pubblica i prezzi dei carburanti in formato CSV:
- **Anagrafica impianti**: https://www.mimit.gov.it/images/exportCSV/anagrafica_impianti_attivi.csv
- **Prezzi in vigore**: https://www.mimit.gov.it/images/exportCSV/prezzo_alle_8.csv

## Geolocalizzazione
- Base di Mirko: Pergine Valdarno (AR), Toscana
- Coordinate: 43.4726° N, 11.7772° E
- Raggio di ricerca default: 10 km

## Logica
1. Scarica i CSV dal MIMIT (anagrafica + prezzi)
2. Filtra per tipo carburante richiesto (default: **Gasolio**)
3. Filtra solo prezzi **self-service** (isSelf = 1)
4. Calcola la distanza dal punto di riferimento (Pergine Valdarno) per ogni impianto
5. Ordina per prezzo crescente
6. Restituisci i **primi 3 impianti più economici** entro il raggio

## Output format (Telegram)
```
⛽ DISTRIBUTORI PIÙ ECONOMICI — Gasolio (self)

1. [Nome Impianto]
   💰 €X.XXX/L
   📍 [Indirizzo], [Comune]
   📏 X.X km

2. [Nome Impianto]
   💰 €X.XXX/L
   📍 [Indirizzo], [Comune]
   📏 X.X km

3. [Nome Impianto]
   💰 €X.XXX/L
   📍 [Indirizzo], [Comune]
   📏 X.X km

Prezzi aggiornati al [data] — Fonte: MIMIT
```

## Parametri
- **Carburante**: Gasolio (default), Benzina, GPL, Metano
- **Raggio**: 10 km default, personalizzabile
- **Modalità**: Self-service (default), Servito
- **Posizione**: da location Telegram (se inviata), altrimenti Pergine Valdarno (43.4726°N, 11.7772°E)

## Location Telegram
⚠️ Il plugin Telegram NON supporta la ricezione di location pin. Mirko deve inviare la posizione come testo:
- "gasolio vicino a 43.47, 11.77" (coordinate)
- "gasolio vicino Levane" (nome del posto)
- "distributore zona Arezzo" (nome città)

Per i nomi di località, usa geocoding (WebSearch o coordinate note) per trovare lat/lon.

## Note tecniche
- I CSV del MIMIT vengono aggiornati quotidianamente (prezzi alle 8:00)
- **Separatore CSV: `|`** (pipe, NON punto e virgola)
- **Prima riga**: header data ("Estrazione del YYYY-MM-DD") — va saltata
- **Seconda riga**: nomi colonne reali
- Il campo `idImpianto` collega anagrafica e prezzi
- Anagrafica colonne: idImpianto|Gestore|Bandiera|Tipo Impianto|Nome Impianto|Indirizzo|Comune|Provincia|Latitudine|Longitudine
- Prezzi colonne: idImpianto|descCarburante|prezzo|isSelf|dtComu
- Coordinate: numeri con punto decimale (es. 43.4726)
- Prezzo: numeri con punto decimale (es. 1.994)
- isSelf: 1 = self-service, 0 = servito
- Formula distanza: Haversine
- Encoding CSV: UTF-8

---

# Colonnine elettriche

Quando Mirko chiede colonnine di ricarica o prezzi ricarica elettrica.

## Strategia: due fonti in parallelo, poi merge

⚠️ **Open Charge Map richiede API key** (anonima → 403 dal 2026-04). Vedi sotto "Come ottenere la tua API key" — registrazione gratuita in 60 secondi. Salva la chiave in un file fuori dal repo, es. `~/mission-control/.secrets/openchargemap.key` con `chmod 600`. **Non committarla mai.**

Interroga **entrambe** le fonti in parallelo, poi unisci e dedup per coordinate vicine (< 80 m):
1. **Open Charge Map** (primaria — dati ricchi, struttura affidabile)
2. **OpenStreetMap Overpass** via mirror Kumi (fallback / integrazione, no key)

Se OCM fallisce o ha pochi risultati, mostra quelli OSM. Se entrambe rispondono, OCM ha la precedenza nei conflitti perché ha dati più completi.

## API 1 — Open Charge Map (con key)
```
https://api.openchargemap.io/v3/poi/?output=json&latitude=LAT&longitude=LON&distance=10&distanceunit=KM&maxresults=10&key=$(cat ~/mission-control/.secrets/openchargemap.key)
```

⚠️ **NON usare** `compact=true&verbose=false`: stripperebbe `OperatorInfo.Title`, `ConnectionType.Title` e `StatusType` (li riduce a soli ID). Usa la versione default per avere i nomi descrittivi.

Parametri utili:
- `latitude`/`longitude`: posizione di riferimento
- `distance`: raggio in km (default skill: 10)
- `maxresults`: 10 (poi mostriamo top 5)
- `connectiontypeid`: tipo connettore (25 = Type 2, 33 = CCS, 2 = CHAdeMO, 27 = Tesla Supercharger)
- `levelid`: 2 = AC lenta, 3 = DC veloce
- `operatorid`: filtra per operatore

Campi POI: `OperatorInfo.Title`, `AddressInfo` (Title, AddressLine1, Town, Distance), `Connections[].ConnectionType.Title + PowerKW + Quantity`, `StatusType.IsOperational`, `NumberOfPoints`. Filtra `IsOperational==true`.

## API 2 — OpenStreetMap Overpass (no key, fallback)
**Usa il mirror Kumi**, non `overpass-api.de` che spesso dà 504:
```
POST https://overpass.kumi.systems/api/interpreter
data=[out:json][timeout:25];nwr["amenity"="charging_station"](around:RADIUS_M,LAT,LON);out center tags;
```
RADIUS_M è in metri (10 km → 10000).

Tag rilevanti: `operator` / `network` / `brand`, `name`, `addr:street`+`addr:housenumber`, `addr:city`/`addr:hamlet`/`addr:village`, `capacity`, `socket:type2`, `socket:type2_combo`, `socket:chademo`, `socket:tesla_supercharger`, `socket:*:output` (potenza kW). Spesso incompleti, soprattutto per operatori minori.

## Logica di merge
1. Determina la posizione (location Telegram, coordinate, o geocoding)
2. Esegui in parallelo: chiamata OCM + chiamata Overpass
3. Per ogni POI calcola la distanza Haversine dal punto di riferimento
4. Dedup: due stazioni con distanza geografica reciproca < 80 m sono la stessa (preferisci la versione OCM)
5. Filtra OCM su `IsOperational==true`
6. Ordina per distanza crescente
7. Restituisci le prime 5

## Nota sui prezzi
I prezzi di ricarica **NON** sono in nessuna delle due fonti — variano per operatore e contratto. Mostra operatore e potenza kW, e suggerisci di verificare il prezzo sull'app dell'operatore (Enel X, BeCharge, Ionity, Tesla, ecc.).

## Output format (Telegram)
```
🔌 COLONNINE DI RICARICA — entro 10 km

1. [Operatore] — [Nome stazione]
   📍 [Indirizzo, Comune]
   📏 X.X km
   ⚡ [Connettori e potenza, es. "CCS 150 kW, Type 2 22 kW"]
   🔢 N punti di ricarica

2. ...

Fonte: Open Charge Map + OpenStreetMap · Verifica i prezzi sull'app dell'operatore
```

## Parametri
- **Raggio**: 10 km default, personalizzabile
- **Numero risultati**: 5 default
- **Posizione**: da location Telegram o nome località (geocoding)
- **Tipo connettore**: tutti per default; filtra solo se Mirko lo chiede esplicitamente

## Come ottenere la tua API key Open Charge Map (gratuita)

L'accesso anonimo a OCM **non funziona più** (da aprile 2026 restituisce `403 — You must specify an API key`). La registrazione è gratuita, richiede ~60 secondi e basta un'email.

1. Vai su https://openchargemap.org/site/loginprovider/beginlogin e accedi con Google, Microsoft, GitHub, Apple o un altro provider OpenID.
2. Apri il tuo profilo → **My Apps** (o direttamente https://openchargemap.org/site/profile/applications).
3. Clicca **Register an Application**:
   - **App name**: qualcosa di descrittivo (es. "chief-of-staff")
   - **Description**: scopo (es. "Personal Telegram bot for EV charging stations")
   - **Website**: il tuo repo o `https://github.com/<your-user>`
4. Submit. La pagina mostra la tua **API key** (formato UUID, es. `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`). Copiala.
5. Salva la chiave **fuori dal repo**, e con permessi restrittivi:
   ```bash
   mkdir -p ~/mission-control/.secrets
   echo 'TUA_CHIAVE_QUI' > ~/mission-control/.secrets/openchargemap.key
   chmod 600 ~/mission-control/.secrets/openchargemap.key
   ```
   Aggiungi `.secrets/` al tuo `.gitignore`. **Non committare mai la chiave.**
6. La skill la legge al volo con `$(cat ~/mission-control/.secrets/openchargemap.key)`.

**Limiti del free tier**: qualche centinaio di richieste/ora per chiave — più che sufficiente per un assistente personale. Se serve di più, dalla stessa pagina My Apps puoi chiedere un tier superiore.

**Senza chiave**: la skill usa il fallback OpenStreetMap Overpass (no key), ma i dati sono meno completi (indirizzi, capacità e connettori spesso mancanti per operatori minori).
