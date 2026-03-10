# DataVault Suchalgorithmus – Dokumentation

## Inhaltsverzeichnis

1. [Überblick](#1-überblick)
2. [Architektur der Suchpipeline](#2-architektur-der-suchpipeline)
3. [Phase 1: Einbettungserzeugung (Indexierung)](#3-phase-1-einbettungserzeugung-indexierung)
4. [Phase 2: Vektorsuche (Semantische Suche)](#4-phase-2-vektorsuche-semantische-suche)
5. [Phase 3: Textsuche (Schlüsselwort-Fallback)](#5-phase-3-textsuche-schlüsselwort-fallback)
6. [Phase 4: Zusammenführung und Deduplizierung](#6-phase-4-zusammenführung-und-deduplizierung)
7. [Zugriffskontrolle bei der Suche](#7-zugriffskontrolle-bei-der-suche)
8. [Frontend-Integration](#8-frontend-integration)
9. [Konfigurationsparameter](#9-konfigurationsparameter)
10. [Ablaufdiagramm](#10-ablaufdiagramm)
11. [Technische Details](#11-technische-details)

---

## 1. Überblick

DataVault verwendet einen **hybriden Suchalgorithmus**, der zwei komplementäre Suchmethoden kombiniert:

1. **Semantische Vektorsuche** – Findet Einträge basierend auf der *Bedeutung* der Suchanfrage, nicht nur auf exakten Wortübereinstimmungen
2. **Schlüsselwort-Textsuche** – Klassische Volltextsuche als Ergänzung und Fallback

Diese Kombination stellt sicher, dass Benutzer relevante Ergebnisse finden, unabhängig davon, ob sie den exakten Titel kennen oder nur eine ungefähre Beschreibung dessen eingeben, was sie suchen.

### Beispiel

Ein Benutzer sucht nach **"WLAN Zugangsdaten"**:

- **Semantische Suche** findet: "Router Passwort", "WiFi-Schlüssel Büro", "Netzwerk-Konfiguration" (bedeutungsähnlich)
- **Textsuche** findet: "WLAN Zugangsdaten Zuhause" (enthält exakten Suchbegriff)

Beide Ergebnismengen werden zusammengeführt und dem Benutzer präsentiert.

---

## 2. Architektur der Suchpipeline

```
                    Suchanfrage: "WLAN Passwort"
                              │
                              ▼
                 ┌────────────────────────┐
                 │     SearchService      │
                 │  search(query, userId, │
                 │    limit, threshold)   │
                 └────────────┬───────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼                               ▼
   ┌─────────────────────┐        ┌─────────────────────┐
   │   VEKTORSUCHE       │        │    TEXTSUCHE         │
   │  (Semantisch)       │        │  (Schlüsselwort)     │
   │                     │        │                      │
   │ 1. Query → Ollama   │        │ 1. Query in Terme    │
   │ 2. Embedding erzeu- │        │    aufteilen          │
   │    gen (1024-dim)   │        │ 2. ILIKE auf title    │
   │ 3. Kosinus-Distanz  │        │    UND value          │
   │    berechnen        │        │ 3. Alle Terme müssen  │
   │ 4. Nach Schwellen-  │        │    zutreffen (AND)    │
   │    wert filtern     │        │                      │
   └──────────┬──────────┘        └──────────┬──────────┘
              │                               │
              │    Semantische Treffer        │    Text-Treffer
              │                               │
              └───────────────┬───────────────┘
                              ▼
                 ┌────────────────────────┐
                 │   ZUSAMMENFÜHRUNG      │
                 │                        │
                 │ 1. Vektor-Ergebnisse   │
                 │    zuerst              │
                 │ 2. Text-Ergebnisse     │
                 │    anfügen (ohne       │
                 │    Duplikate)           │
                 │ 3. Auf Limit kürzen    │
                 └────────────┬───────────┘
                              │
                              ▼
                    Suchergebnisse an Client
```

---

## 3. Phase 1: Einbettungserzeugung (Indexierung)

### Was ist eine Einbettung?

Eine Einbettung (englisch: *Embedding*) ist eine mathematische Darstellung von Text als Zahlenvektor. Texte mit ähnlicher Bedeutung haben ähnliche Vektoren, unabhängig von den verwendeten Wörtern.

### Wann werden Einbettungen erzeugt?

| Ereignis                   | Aktion                                 |
| -------------------------- | -------------------------------------- |
| Eintrag wird erstellt      | Einbettung wird automatisch generiert  |
| Titel wird aktualisiert    | Einbettung wird neu generiert          |
| Nur Wert wird geändert     | Einbettung bleibt unverändert          |
| `entries:reembed` Befehl   | Alle Einbettungen werden neu erstellt  |

### Welche Daten werden eingebettet?

**Nur der Titel** des Eintrags wird eingebettet – nicht der Wert (Inhalt). Diese Entscheidung wurde aus **Datenschutzgründen** getroffen:

- Titel sind in der Regel beschreibende Bezeichnungen ("Bank-Passwort", "Server SSH-Schlüssel")
- Werte enthalten die eigentlichen sensiblen Daten (Passwörter, Schlüssel, usw.)
- Einbettungen könnten theoretisch Rückschlüsse auf den Inhalt ermöglichen

### Technischer Ablauf

```
Titel: "Router Passwort Büro"
         │
         ▼
┌─────────────────────────┐
│      OllamaService      │
│                          │
│  POST /api/embeddings    │
│  {                       │
│    "model": "mxbai-      │
│     embed-large",        │
│    "prompt": "Router     │
│     Passwort Büro"       │
│  }                       │
└────────────┬────────────┘
             │
             ▼
   Vektor: [0.0234, -0.1567, 0.8901, ...]
            (1024 Dimensionen)
             │
             ▼
┌─────────────────────────┐
│     PostgreSQL           │
│                          │
│  UPDATE entries          │
│  SET embedding = '[...]' │
│  WHERE ulid = '...'      │
└─────────────────────────┘
```

### Fehlerbehandlung

Wenn Ollama nicht erreichbar ist oder die Einbettungserzeugung fehlschlägt:

- Der Eintrag wird **trotzdem erstellt/aktualisiert** (ohne Einbettung)
- Die Suche funktioniert weiterhin über den **Text-Fallback**
- Der Fehler wird protokolliert

---

## 4. Phase 2: Vektorsuche (Semantische Suche)

### Funktionsweise

Die Vektorsuche nutzt die **Kosinus-Distanz** zwischen dem Suchvektor und den gespeicherten Einbettungen, um semantisch ähnliche Einträge zu finden.

### Algorithmus (Schritt für Schritt)

```
Eingabe: query = "WLAN Zugangsdaten", threshold = 1.0, limit = 10

1. Einbettung der Suchanfrage erzeugen:
   query_embedding = ollama.embed("WLAN Zugangsdaten")
   → [0.0451, -0.2301, 0.7812, ...]  (1024 Dimensionen)

2. SQL-Abfrage mit pgvector Kosinus-Distanz-Operator (<=>):

   SELECT entries.*,
          (embedding <=> query_embedding) AS distance
   FROM entries
   WHERE embedding IS NOT NULL
     AND user_id = :userId  (Zugriffskontrolle)
     AND (embedding <=> query_embedding) < :threshold
   ORDER BY embedding <=> query_embedding ASC
   LIMIT :limit

3. Ähnlichkeitswert berechnen:
   Für jeden Treffer:
     similarity = ROUND(1 - distance, 4)

   Beispiel:
     distance = 0.0458 → similarity = 0.9542 (95,42% ähnlich)
     distance = 0.3200 → similarity = 0.6800 (68,00% ähnlich)
     distance = 1.0500 → wird herausgefiltert (> threshold 1.0)
```

### Kosinus-Distanz erklärt

Die **Kosinus-Distanz** misst den Winkel zwischen zwei Vektoren:

| Distanz | Ähnlichkeit | Bedeutung                           |
| ------- | ----------- | ----------------------------------- |
| 0.0     | 1.0 (100%)  | Identische Bedeutung                |
| 0.5     | 0.5 (50%)   | Mäßig ähnlich                       |
| 1.0     | 0.0 (0%)    | Keine erkennbare Ähnlichkeit        |
| 2.0     | -1.0        | Entgegengesetzte Bedeutung          |

Der **Standardschwellenwert von 1.0** ist bewusst großzügig gewählt, um möglichst viele relevante Ergebnisse einzuschließen. Benutzer können diesen über den Parameter `threshold` einschränken.

### pgvector-Erweiterung

DataVault nutzt die PostgreSQL-Erweiterung **pgvector** für effiziente Vektoroperationen:

- **Speicherung:** Nativer `vector(1024)`-Spaltentyp
- **Operator:** `<=>` für Kosinus-Distanz
- **Indizierung:** Automatische Indizierung durch PostgreSQL
- **Leistung:** Optimiert für hochdimensionale Vektorvergleiche

---

## 5. Phase 3: Textsuche (Schlüsselwort-Fallback)

### Funktionsweise

Die Textsuche ergänzt die Vektorsuche durch eine klassische Schlüsselwort-basierte Suche. Sie findet Einträge, die die exakten Suchbegriffe im Titel oder Wert enthalten.

### Algorithmus (Schritt für Schritt)

```
Eingabe: query = "WLAN Passwort", limit = 10

1. Suchanfrage in einzelne Terme aufteilen:
   terms = ["WLAN", "Passwort"]

2. SQL-Abfrage mit ILIKE (Groß-/Kleinschreibung ignorierend):

   SELECT entries.*
   FROM entries
   WHERE user_id = :userId
     AND (title ILIKE '%WLAN%' OR value ILIKE '%WLAN%')
     AND (title ILIKE '%Passwort%' OR value ILIKE '%Passwort%')
   LIMIT :limit

3. Ergebnisse als "text"-Typ markieren:
   search_type = "text"
   similarity = null
```

### Wichtige Eigenschaften

| Eigenschaft                | Beschreibung                                           |
| -------------------------- | ------------------------------------------------------ |
| Groß-/Kleinschreibung     | Wird ignoriert (ILIKE)                                 |
| Mehrere Suchbegriffe       | ALLE müssen zutreffen (AND-Verknüpfung)               |
| Durchsuchte Felder         | `title` UND `value` (OR-Verknüpfung pro Term)         |
| Teilübereinstimmungen      | Ja, durch `%`-Wildcards (`%WLAN%` findet "mein WLAN") |
| Einbettung erforderlich?   | Nein – funktioniert auch ohne Einbettung               |

### Wann ist die Textsuche besonders nützlich?

- Wenn Ollama nicht verfügbar ist (Fallback-Szenario)
- Bei exakten Begriffen, die in der semantischen Suche untergehen könnten
- Bei der Suche im `value`-Feld (Einbettungen basieren nur auf dem Titel)
- Bei Sonderzeichen, Codes oder Seriennummern

---

## 6. Phase 4: Zusammenführung und Deduplizierung

### Ablauf

```
Vektor-Ergebnisse:           Text-Ergebnisse:
┌─────────────────────┐      ┌─────────────────────┐
│ 1. Router Passwort  │      │ 1. WLAN Zugangsd.   │
│    (sem, 0.95)      │      │    (text)           │
│ 2. WiFi-Schlüssel   │      │ 2. Router Passwort  │  ← Duplikat!
│    (sem, 0.82)      │      │    (text)           │
│ 3. Netzwerk-Konf.   │      │ 3. VPN Passwort     │
│    (sem, 0.71)      │      │    (text)           │
└─────────────────────┘      └─────────────────────┘

                    │
                    ▼   Zusammenführung

┌──────────────────────────────────────────┐
│ 1. Router Passwort      (sem, 0.95)      │  ← Aus Vektorsuche
│ 2. WiFi-Schlüssel       (sem, 0.82)      │  ← Aus Vektorsuche
│ 3. Netzwerk-Konf.       (sem, 0.71)      │  ← Aus Vektorsuche
│ 4. WLAN Zugangsdaten    (text, null)     │  ← Aus Textsuche
│ 5. VPN Passwort         (text, null)     │  ← Aus Textsuche
│    [Router Passwort übersprungen – dup.] │
└──────────────────────────────────────────┘
```

### Regeln

1. **Vektor-Ergebnisse haben Vorrang** – Sie werden zuerst in die Ergebnisliste aufgenommen
2. **Deduplizierung** – Text-Ergebnisse, die bereits in den Vektor-Ergebnissen enthalten sind, werden übersprungen
3. **Limit-Einhaltung** – Die kombinierte Ergebnismenge wird auf den angeforderten `limit`-Wert gekürzt

### Ergebnis-Metadaten

Jedes Suchergebnis enthält zusätzliche Metadaten:

| Feld          | Typ     | Beschreibung                                    |
| ------------- | ------- | ----------------------------------------------- |
| `search_type` | String  | `"semantic"` oder `"text"`                      |
| `similarity`  | Float   | Ähnlichkeitswert (0–1), nur bei semantischen Treffern, sonst `null` |
| `is_shared`   | Boolean | `true` wenn der Eintrag mit dem Benutzer geteilt wurde |

---

## 7. Zugriffskontrolle bei der Suche

Die Suche berücksichtigt automatisch die Zugriffsrechte des Benutzers. Ein Benutzer findet nur Einträge, die er:

1. **Selbst besitzt** – `entries.user_id = aktueller_benutzer.id`
2. **Geteilt bekommen hat** – Eintrag existiert in `entry_shares` mit `shared_with_id = aktueller_benutzer.id`

```sql
-- Zugriffsfilter (vereinfacht)
WHERE (
    entries.user_id = :userId                    -- Eigene Einträge
    OR EXISTS (                                  -- ODER geteilte Einträge
        SELECT 1 FROM entry_shares
        WHERE entry_shares.entry_id = entries.id
        AND entry_shares.shared_with_id = :userId
    )
)
```

---

## 8. Frontend-Integration

### Suchablauf im Frontend

```
┌──────────────────────────────────────────────────────┐
│                     SearchBar                         │
│  ┌─────────────────────────────────────────────────┐  │
│  │ 🔍  WLAN Passwort                            ✕ │  │
│  └─────────────────────────────────────────────────┘  │
└──────────────────────────┬───────────────────────────┘
                           │ onChange
                           ▼
                ┌──────────────────────┐
                │    useSearch Hook    │
                │                      │
                │  ┌────────────────┐  │
                │  │ 300ms Debounce │  │
                │  └───────┬────────┘  │
                │          │           │
                │  ┌───────▼────────┐  │
                │  │ searchService  │  │
                │  │  .search({    │  │
                │  │   q: "WLAN    │  │
                │  │   Passwort",  │  │
                │  │   limit: 20   │  │
                │  │  })           │  │
                │  └───────┬────────┘  │
                └──────────┼───────────┘
                           │
                           ▼
              GET /api/v1/entries/search
                ?q=WLAN+Passwort&limit=20
              Authorization: Bearer <token>
                           │
                           ▼
                ┌──────────────────────┐
                │    Ergebnisliste     │
                │                      │
                │  ┌────────────────┐  │
                │  │ Router Passw.  │  │
                │  │ SEMANTIC 95%   │  │
                │  ├────────────────┤  │
                │  │ WiFi-Schlüssel │  │
                │  │ SEMANTIC 82%   │  │
                │  ├────────────────┤  │
                │  │ WLAN Zugangsd. │  │
                │  │ TEXT           │  │
                │  └────────────────┘  │
                └──────────────────────┘
```

### Debouncing

Das Frontend verwendet ein **300ms Debounce**-Intervall, um übermäßige API-Aufrufe während der Eingabe zu vermeiden:

```
Benutzer tippt: W → WL → WLA → WLAN
                │    │    │     │
                ▼    ▼    ▼     ▼
Timer:        300ms reset reset reset
                              │
                              ▼ (300ms nach letztem Tastendruck)
                         API-Aufruf mit "WLAN"
```

### Anzeige der Suchergebnisse

| Suchtyp      | Badge-Farbe | Zusatzinfo           |
| ------------ | ----------- | -------------------- |
| `semantic`   | Violett     | Ähnlichkeit in %     |
| `text`       | Bernstein   | Keine Zusatzinfo     |

---

## 9. Konfigurationsparameter

### Serverseitige Konfiguration

| Parameter                    | Wert                | Beschreibung                            |
| ---------------------------- | ------------------- | --------------------------------------- |
| Einbettungsmodell            | `mxbai-embed-large` | Ollama-Modell für Einbettungen          |
| Vektordimension              | 1024                | Anzahl der Dimensionen pro Einbettung   |
| Ollama-Timeout (Embedding)   | 30 Sekunden         | Zeitlimit für Einbettungserzeugung      |
| Ollama-Timeout (Health)      | 5 Sekunden          | Zeitlimit für Statusprüfung             |
| Ollama-Timeout (Model Pull)  | 300 Sekunden        | Zeitlimit für Modell-Download           |

### Suchparameter (API-Aufruf)

| Parameter   | Standard | Bereich  | Beschreibung                                         |
| ----------- | -------- | -------- | ---------------------------------------------------- |
| `q`         | –        | 1–500    | Suchbegriff (Pflichtfeld)                            |
| `limit`     | 10       | 1–50     | Maximale Anzahl zurückgegebener Ergebnisse           |
| `threshold` | 1.0      | 0.0–2.0  | Kosinus-Distanz-Schwellenwert für Vektorsuche        |

**Threshold-Empfehlungen:**

| Wert   | Effekt                                                     |
| ------ | ---------------------------------------------------------- |
| 0.3    | Nur sehr ähnliche Ergebnisse (streng)                      |
| 0.5    | Gute Balance zwischen Relevanz und Vollständigkeit         |
| 1.0    | Standard – breit gefächerte semantische Ergebnisse         |
| 1.5+   | Sehr großzügig – auch entfernt ähnliche Ergebnisse         |

---

## 10. Ablaufdiagramm

### Vollständiger Suchalgorithmus

```
START: search(query, userId, limit=10, threshold=1.0)
  │
  ├─── Phase 1: Vektorsuche ──────────────────────────────┐
  │    │                                                    │
  │    ├── Sende query an Ollama /api/embeddings           │
  │    │   Modell: mxbai-embed-large                       │
  │    │                                                    │
  │    ├── Embedding erhalten?                              │
  │    │   ├── JA → SQL: SELECT * FROM entries             │
  │    │   │         WHERE embedding <=> query_vec          │
  │    │   │               < threshold                      │
  │    │   │         AND (user_id = :uid                    │
  │    │   │              OR shared_with = :uid)            │
  │    │   │         ORDER BY distance ASC                  │
  │    │   │         LIMIT :limit                           │
  │    │   │                                                │
  │    │   │    Für jeden Treffer:                          │
  │    │   │      similarity = round(1 - distance, 4)       │
  │    │   │      search_type = "semantic"                  │
  │    │   │                                                │
  │    │   └── NEIN → vectorResults = leer                  │
  │    │              (Ollama nicht erreichbar)              │
  │    │                                                    │
  │                                                         │
  ├─── Phase 2: Textsuche ────────────────────────────────┐
  │    │                                                    │
  │    ├── Query in Terme aufteilen (Leerzeichen)          │
  │    │   "WLAN Passwort" → ["WLAN", "Passwort"]          │
  │    │                                                    │
  │    ├── SQL: SELECT * FROM entries                       │
  │    │   WHERE (title ILIKE '%WLAN%'                      │
  │    │          OR value ILIKE '%WLAN%')                   │
  │    │     AND (title ILIKE '%Passwort%'                   │
  │    │          OR value ILIKE '%Passwort%')               │
  │    │     AND (user_id = :uid OR shared_with = :uid)     │
  │    │   LIMIT :limit                                     │
  │    │                                                    │
  │    ├── Für jeden Treffer:                               │
  │    │     search_type = "text"                           │
  │    │     similarity = null                              │
  │    │                                                    │
  │                                                         │
  ├─── Phase 3: Zusammenführung ──────────────────────────┐
  │    │                                                    │
  │    ├── merged = vectorResults                           │
  │    │                                                    │
  │    ├── Für jeden textResult:                            │
  │    │   └── Wenn textResult.id NICHT in merged:          │
  │    │       └── merged.append(textResult)                │
  │    │                                                    │
  │    ├── merged = merged[0:limit]  (auf Limit kürzen)    │
  │    │                                                    │
  │    ├── Für jeden Treffer:                               │
  │    │   └── is_shared = ist der Eintrag geteilt?        │
  │    │                                                    │
  │    └── RETURN merged                                    │
  │                                                         │
  └──────────────────────────────────────────────────────────┘
```

---

## 11. Technische Details

### Verwendete Technologien

| Komponente   | Technologie           | Zweck                                    |
| ------------ | --------------------- | ---------------------------------------- |
| Vektorspeicher| PostgreSQL + pgvector | Speicherung und Vergleich von Vektoren  |
| KI-Modell    | mxbai-embed-large     | Texte in 1024-dim. Vektoren umwandeln   |
| KI-Server    | Ollama (selbst geh.)  | Lokale Ausführung des Einbettungsmodells |
| Distanzmaß   | Kosinus-Distanz (`<=>`)| Ähnlichkeit zwischen Vektoren messen    |
| Text-Matching| PostgreSQL ILIKE      | Groß-/Kleinschreibung-unabhängige Suche |

### Warum selbst gehostetes Ollama?

DataVault verwendet **Ollama** statt eines Cloud-KI-Dienstes (wie OpenAI) aus folgenden Gründen:

1. **Datenschutz** – Keine Benutzerdaten verlassen den eigenen Server
2. **Keine API-Kosten** – Kein Drittanbieter-Abonnement erforderlich
3. **Offline-fähig** – Funktioniert ohne Internetverbindung
4. **Volle Kontrolle** – Modell und Infrastruktur werden selbst verwaltet

### Warum pgvector statt eines dedizierten Vektorspeichers?

Anstatt einen separaten Vektorspeicher wie Pinecone, Weaviate oder Milvus zu verwenden, nutzt DataVault die pgvector-Erweiterung von PostgreSQL:

1. **Architekturvereinfachung** – Keine zusätzliche Infrastruktur erforderlich
2. **Transaktionskonsistenz** – Einträge und Einbettungen in derselben Transaktion
3. **Einfache Wartung** – Nur eine Datenbank zu verwalten
4. **Ausreichende Leistung** – Für typische Datenvault-Größen (tausende bis zehntausende Einträge) ist pgvector performant genug

### Neuindizierung

Bei Änderungen am Einbettungsmodell oder zur Fehlerbehebung können alle Einbettungen neu generiert werden:

```bash
# Alle Einträge neu einbetten
php artisan entries:reembed

# Nur Einträge eines bestimmten Benutzers
php artisan entries:reembed --user=1
```

Der Befehl zeigt einen Fortschrittsbalken und zählt fehlgeschlagene Einbettungen.

---

*Zurück zur [REST-Server-Dokumentation](REST-Server.md)*
