# Database interattivo – Sicurezza Stradale e Mobilità Attiva

> Documento di recap della strategia concordata. Versione iniziale.
> I dati di dettaglio sui singoli progetti verranno aggiunti in seguito.

## 1. Obiettivo

Creare un **database interattivo su mappa** degli interventi del gruppo
Sicurezza Stradale e Mobilità Attiva (zone 30, velox, radar, strade
scolastiche, ecc.), che permetta di:

- **visualizzare su mappa** dove sono stati realizzati gli interventi;
- **interrogare** gli elementi (cliccare un punto/area e vedere le sue informazioni);
- **collegarsi alla cartella di progetto** sul server per la documentazione aggiornata;
- **estrarre/calcolare informazioni in tempo reale** (aree, lunghezze, conteggi, totali);
- **scegliere cosa visualizzare** tramite menù a tendina (uno o più tipi di intervento insieme).

Lo strumento deve **crescere nel tempo** raccogliendo anche gli interventi di
altri gruppi, mantenendo gli elementi pesanti (documenti) **linkati** al server
invece che incorporati.

## 2. Contesto e vincoli (decisi insieme)

| Aspetto | Decisione |
|---|---|
| **Chi inserisce i dati** | Solo l'editor (esperta di analisi spaziali, lavora in **Python/geopandas**, all'occorrenza QGIS). |
| **Chi consulta** | Tutti i colleghi, **senza login**, in sola visualizzazione/interrogazione. |
| **Hosting (oggi)** | **Cartella condivisa** sul server aziendale. |
| **Hosting (futuro)** | Possibile passaggio a web server interno se il progetto si allarga alla direzione (da concordare con i sistemisti). |
| **Dati esistenti** | Misti: parte in GIS, parte in tabelle/documenti. |
| **Link cartelle** | Percorsi di rete `\\server\...`. |
| **Internet sui PC** | Da verificare → la mappa di base sarà progettata come **intercambiabile** (online o offline). |

## 3. Strategia / architettura

Separazione netta tra preparazione dati e visualizzazione:

- **Backend "a freddo" (gestito dall'editor, in Python)**
  Uno *script di build* in geopandas legge le fonti, normalizza i dati,
  **calcola gli attributi derivati** (aree, lunghezze, conteggi) ed esporta
  **GeoJSON** pronti per il web. Nessun database server da mantenere.

- **Frontend statico (aperto da tutti)**
  Cartella `web/` con `index.html` + JavaScript + i GeoJSON. Mappa interattiva
  con menù a tendina, popup, link e calcoli dinamici. Funziona da cartella
  condivisa oggi e, identica, su un web server domani.

### Modello dati "config-driven" (cuore del progetto)

Ogni tipo di intervento ha geometria, attributi, calcoli, stile e link diversi.
Invece di scrivere codice nuovo per ognuno, ogni tipo è descritto in un file di
configurazione (`layers.json`). Aggiungere un nuovo intervento o un nuovo gruppo
= aggiungere una voce di config + un GeoJSON, **senza codice nuovo**. È ciò che
rende lo strumento scalabile nel tempo.

Esempio:

```jsonc
{
  "zone30": {
    "label": "Zone 30",
    "geometria": "poligono",
    "stile": { "colore": "#e63946", "opacita": 0.4 },
    "campi": ["nome", "anno", "stato", "cartella"],
    "calcoli": ["area_mq", "perimetro_m"],
    "link_cartella": "cartella"
  },
  "velox": {
    "label": "Velox",
    "geometria": "punto",
    "stile_per_stato": { "previsto": "#999", "in_corso": "#f4a261", "attivo": "#2a9d8f" },
    "campi": ["via", "fase", "data_attivazione", "cartella"],
    "calcoli": ["conteggio"],
    "link_cartella": "cartella"
  }
}
```

## 4. Stack tecnico

- **Mappa**: MapLibre GL JS (gratuita, performante, stili per stato/fase).
- **Calcoli dinamici nel browser**: Turf.js (somme di aree/lunghezze/conteggi
  sugli elementi filtrati o selezionati, in tempo reale).
- **Calcoli pesanti pre-fatti**: geopandas in CRS metrico (es. EPSG:32632) per
  aree/lunghezze corrette, poi riproiezione in EPSG:4326 per il web.
- **UI**: menù a tendina/checkbox per accendere i layer, pannello laterale con
  tabella e totali.

## 5. Vincoli tecnici da gestire

- **Link a cartelle `\\server\...`**: i browser bloccano i link ai percorsi di
  rete da una pagina web. Soluzione robusta: pulsante **"Copia percorso"** nel
  popup (l'utente incolla in Esplora risorse). In alternativa, se i sistemisti
  espongono la documentazione via URL intranet (`http://...`), il link diventa
  cliccabile diretto (basta cambiare un campo nella config).
- **Mappa di base / Internet incerto**: base map intercambiabile. Default
  OpenStreetMap online, con fallback offline (sfondo neutro + geometrie, o tile
  locali). Verifica pratica: aprire la pagina di prova su un PC "tipo".

## 6. Piano step-by-step

- **Fase 0 — Setup**: struttura repo, script di build vuoto, pagina mappa
  minima funzionante. Verifica internet/tile su un PC tipo.
- **Fase 1 — Un intervento end-to-end** (es. Zone 30): schema attributi →
  script geopandas (area/perimetro) → GeoJSON → mappa con popup, link cartella,
  area totale dinamica. Valida l'intera catena su un caso reale.
- **Fase 2 — Sistema config-driven multi-layer**: `layers.json` + velox, radar,
  strade scolastiche. Menù a tendina, più layer insieme, stili per fase/stato.
- **Fase 3 — Interrogazione e analisi**: filtri (anno, stato, zona), tabella
  esportabile (CSV), totali in tempo reale.
- **Fase 4 — Deploy + documentazione**: pacchettizzazione, test dalla share,
  guida per i colleghi e per l'aggiornamento dei dati.
- **Fase 5 (futuro) — Espansione**: web server interno, eventuale login per più
  editor, contributi di altri gruppi.

## 7. Struttura del repository (prevista)

```
database_progettazione/
├── data/sorgenti/        # file grezzi (shp, xlsx…)
├── build/build.py        # pipeline geopandas → GeoJSON
├── web/
│   ├── index.html
│   ├── app.js
│   ├── layers.json       # config degli interventi
│   └── data/             # GeoJSON generati
├── docs/
│   └── RECAP_PROGETTO.md # questo documento
└── README.md
```

## 8. Informazioni ancora da raccogliere

- Per ogni tipo di intervento: campi/attributi da mostrare nel popup e calcoli
  necessari.
- Città/zona di lavoro (per centrare la mappa) e formato attuale dei dati.
- Conferma su disponibilità Internet sui PC dei colleghi.
- Eventuale URL intranet per la documentazione (in alternativa ai percorsi di rete).
