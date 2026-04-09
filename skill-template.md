# Fuel Prices — Skill

Quando Mirko chiede il prezzo del carburante o il distributore più economico in zona, usa questa skill.

## Trigger
- "qual è il distributore meno caro in zona?"
- "prezzo gasolio/benzina"
- "distributore vicino"
- "carburante"
- qualsiasi richiesta su prezzi benzina/gasolio/GPL/metano
- **Invio location Telegram**: se Mirko manda una posizione (location pin) senza testo, rispondi automaticamente con i 3 distributori gasolio self più economici vicini a quella posizione. Se aggiunge testo (es. "benzina"), usa quel carburante.

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
