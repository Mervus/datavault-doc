# DataVault REST-Server Dokumentation

## Inhaltsverzeichnis

1. [Technologie-Stack](#1-technologie-stack)
2. [Architekturübersicht](#2-architekturübersicht)
3. [Authentifizierung](#3-authentifizierung)
4. [API-Endpunkte](#4-api-endpunkte)
5. [Datenmodelle und Datenbankschema](#5-datenmodelle-und-datenbankschema)
6. [Datei-Speicherung](#6-datei-speicherung)
7. [Ratenbegrenzung](#7-ratenbegrenzung)
8. [Antwortformate](#8-antwortformate)
9. [Validierungsregeln](#9-validierungsregeln)
10. [Docker-Infrastruktur](#10-docker-infrastruktur)

---

## 1. Technologie-Stack

| Komponente        | Technologie                                      |
| ----------------- | ------------------------------------------------ |
| Backend-Framework | Laravel 12 (PHP 8.5)                             |
| Datenbank         | PostgreSQL mit pgvector-Erweiterung              |
| Authentifizierung | Laravel Sanctum (Token-basiert)                  |
| Einbettungsmodell | Ollama (selbst gehostet), Modell: `mxbai-embed-large` |
| Containerisierung | Docker Compose                                   |
| Dateispeicherung  | Lokal oder S3-kompatibel (konfigurierbar)        |

---

## 2. Architekturübersicht

DataVault folgt einer **Service-orientierten Architektur** innerhalb des Laravel-Frameworks:

```
┌─────────────────────────────────────────────────┐
│                   Client (App)                  │
│             React + Capacitor (Web/Mobil)        │
└──────────────────────┬──────────────────────────┘
                       │ HTTPS / Bearer-Token
                       ▼
┌─────────────────────────────────────────────────┐
│              Laravel REST-API (v1)              │
│  ┌───────────┐ ┌──────────┐ ┌───────────────┐  │
│  │   Auth-   │ │ Entry-   │ │  Category-    │  │
│  │ Controller│ │Controller│ │  Controller   │  │
│  └─────┬─────┘ └────┬─────┘ └──────┬────────┘  │
│        │             │              │            │
│  ┌─────┴─────────────┴──────────────┴────────┐  │
│  │              Service-Schicht              │  │
│  │  SearchService │ OllamaService │ Storage  │  │
│  └───────────────────────────────────────────┘  │
└──────────────────────┬──────────────────────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
   ┌────────────┐ ┌─────────┐ ┌──────────┐
   │ PostgreSQL │ │ Ollama  │ │ Datei-   │
   │ + pgvector │ │  (KI)   │ │ speicher │
   └────────────┘ └─────────┘ └──────────┘
```

**Entwurfsmuster:**

- **Service-basierte Architektur** – Geschäftslogik in dedizierte Services ausgelagert (SearchService, OllamaService, StorageService)
- **Eloquent ORM** – Datenbankzugriff über das Repository-Pattern von Laravel
- **Resource-Klassen** – Standardisierte API-Antwortformatierung
- **ULID-Trait** – Nicht-sequenzielle öffentliche Bezeichner für alle Entitäten (statt auto-increment IDs)
- **Enum für Berechtigungen** – Typsichere Freigabeberechtigungen

---

## 3. Authentifizierung

### Token-basierte Authentifizierung mit Laravel Sanctum

DataVault verwendet **Bearer-Token-Authentifizierung**. Bei der Anmeldung erhält der Client einen API-Token, der bei jedem nachfolgenden Request im `Authorization`-Header mitgesendet wird.

**Ablauf:**

```
1. Client sendet POST /api/v1/auth/login mit E-Mail + Passwort
2. Server validiert Anmeldedaten
3. Server erstellt einen Sanctum-Token
4. Client erhält den Token (Klartext, wird nur einmal zurückgegeben)
5. Client speichert den Token (localStorage / Capacitor Preferences)
6. Alle folgenden Requests enthalten: Authorization: Bearer <token>
```

**Sicherheitsmerkmale:**

- Passwörter werden mit **bcrypt** gehasht (12 Runden)
- Token verfallen nach **120 Minuten** Inaktivität
- Benutzer können einzelne Token widerrufen
- ULIDs als öffentliche Bezeichner (nicht vorhersagbar)

---

## 4. API-Endpunkte

Alle Endpunkte befinden sich unter dem Präfix `/api/v1`.

### 4.1 Systemstatus

| Methode | Pfad      | Auth | Beschreibung                                    |
| ------- | --------- | ---- | ----------------------------------------------- |
| GET     | `/health` | Nein | Gibt den Status von Datenbank und Ollama zurück |

### 4.2 Authentifizierung

| Methode | Pfad                   | Auth | Rate-Limit  | Beschreibung                         |
| ------- | ---------------------- | ---- | ----------- | ------------------------------------ |
| POST    | `/auth/register`       | Nein | 10/Min (IP) | Neuen Benutzer registrieren          |
| POST    | `/auth/login`          | Nein | 10/Min (IP) | Anmelden und API-Token erhalten      |
| GET     | `/auth/tokens`         | Ja   | 60/Min      | Alle aktiven Tokens auflisten        |
| DELETE  | `/auth/tokens/{id}`    | Ja   | 60/Min      | Einen bestimmten Token widerrufen    |

### 4.3 Kategorien (CRUD)

| Methode | Pfad                  | Auth | Rate-Limit | Beschreibung                              |
| ------- | --------------------- | ---- | ---------- | ----------------------------------------- |
| GET     | `/categories`         | Ja   | 60/Min     | Alle Kategorien des Benutzers auflisten   |
| POST    | `/categories`         | Ja   | 60/Min     | Neue Kategorie erstellen                  |
| GET     | `/categories/{ulid}`  | Ja   | 60/Min     | Einzelne Kategorie mit Eintragsanzahl     |
| PUT     | `/categories/{ulid}`  | Ja   | 60/Min     | Kategorie aktualisieren                   |
| DELETE  | `/categories/{ulid}`  | Ja   | 60/Min     | Kategorie löschen (kaskadiert zu Einträgen) |

**Felder einer Kategorie:**

| Feld    | Typ     | Pflicht | Beschreibung                          |
| ------- | ------- | ------- | ------------------------------------- |
| `name`  | String  | Ja      | Name der Kategorie (max. 100 Zeichen) |
| `icon`  | Integer | Nein    | Icon-Index                            |
| `color` | String  | Nein    | Hex-Farbcode (max. 7 Zeichen, z.B. `#ff0000`) |

### 4.4 Einträge (CRUD + Suche)

| Methode | Pfad                    | Auth | Rate-Limit | Beschreibung                          |
| ------- | ----------------------- | ---- | ---------- | ------------------------------------- |
| GET     | `/entries`              | Ja   | 60/Min     | Einträge mit Paginierung auflisten    |
| POST    | `/entries`              | Ja   | 60/Min     | Neuen Eintrag erstellen (+ Embedding) |
| GET     | `/entries/{ulid}`       | Ja   | 60/Min     | Einzelnen Eintrag abrufen             |
| PUT     | `/entries/{ulid}`       | Ja   | 60/Min     | Eintrag aktualisieren                 |
| DELETE  | `/entries/{ulid}`       | Ja   | 60/Min     | Eintrag löschen (nur Eigentümer)      |
| GET     | `/entries/search?q=...` | Ja   | 20/Min     | Hybride semantische + Textsuche       |

**Felder eines Eintrags:**

| Feld            | Typ     | Pflicht | Beschreibung                               |
| --------------- | ------- | ------- | ------------------------------------------ |
| `title`         | String  | Ja      | Titel des Eintrags (max. 500 Zeichen)      |
| `value`         | String  | Ja      | Inhalt des Eintrags (max. 5000 Zeichen)    |
| `category_ulid` | String  | Nein    | ULID der zugehörigen Kategorie             |
| `is_sensitive`  | Boolean | Nein    | Markiert den Eintrag als sensibel          |
| `embedding`     | Vektor  | Auto    | 1024-dimensionaler Vektor (versteckt in API-Antworten) |

**Suchparameter:**

| Parameter   | Typ    | Standard | Beschreibung                            |
| ----------- | ------ | -------- | --------------------------------------- |
| `q`         | String | –        | Suchbegriff (Pflicht, 1–500 Zeichen)    |
| `limit`     | Int    | 10       | Maximale Ergebnisanzahl (max. 50)       |
| `threshold` | Float  | 1.0      | Kosinus-Distanz-Schwellenwert (0–2)     |

### 4.5 Dateien

| Methode | Pfad                                           | Auth | Beschreibung                       |
| ------- | ---------------------------------------------- | ---- | ---------------------------------- |
| POST    | `/entries/{ulid}/files`                        | Ja   | Datei hochladen (max. 10 MB)      |
| DELETE  | `/entries/{ulid}/files/{fileUlid}`             | Ja   | Datei löschen (nur Eigentümer)    |
| GET     | `/entries/{ulid}/files/{fileUlid}/download`    | Ja   | Datei herunterladen                |

### 4.6 Freigaben

| Methode | Pfad                                        | Auth | Beschreibung                          |
| ------- | ------------------------------------------- | ---- | ------------------------------------- |
| GET     | `/entries/{ulid}/shares`                    | Ja   | Alle Freigaben eines Eintrags listen  |
| POST    | `/entries/{ulid}/shares`                    | Ja   | Eintrag mit einem Benutzer teilen     |
| PUT     | `/entries/{ulid}/shares/{shareUlid}`        | Ja   | Freigabeberechtigung ändern           |
| DELETE  | `/entries/{ulid}/shares/{shareUlid}`        | Ja   | Freigabe widerrufen                   |

**Berechtigungsstufen:**

| Wert | Berechtigung | Beschreibung                                    |
| ---- | ------------ | ----------------------------------------------- |
| `0`  | Lesen        | Eintrag nur ansehen                             |
| `1`  | Schreiben    | Eintragswert bearbeiten (keine Metadaten)       |

**Einschränkungen:**
- Man kann nicht mit sich selbst teilen
- Jeder Benutzer kann einen Eintrag nur einmal erhalten (Unique-Constraint)
- Nur der Eigentümer kann Freigaben verwalten

---

## 5. Datenmodelle und Datenbankschema

### Entitätsbeziehungsdiagramm

```
┌──────────┐       ┌────────────┐       ┌──────────────┐
│  users   │1    N │ categories │       │personal_access│
│          ├───────┤            │       │   _tokens     │
│  - ulid  │       │  - ulid    │       │  - token      │
│  - name  │       │  - name    │       │  - last_used  │
│  - email │       │  - icon    │       └───────────────┘
│  - pass  │       │  - color   │
└────┬─────┘       └─────┬──────┘
     │1                  │1
     │                   │
     │N                  │N
┌────┴─────┐       ┌─────┴──────┐
│ entries  ├───────┤            │
│          │N    1 │            │
│  - ulid  │       └────────────┘
│  - title │
│  - value │       ┌──────────────┐
│  - embed │1    N │ entry_files  │
│  - sens. ├───────┤              │
└────┬─────┘       │ - orig_name  │
     │1            │ - mime_type  │
     │             │ - file_size  │
     │N            └──────────────┘
┌────┴──────────┐
│ entry_shares  │
│               │
│ - permission  │
│ - shared_with │
└───────────────┘
```

### Tabellen im Detail

**users**

| Spalte       | Typ       | Beschreibung              |
| ------------ | --------- | ------------------------- |
| `id`         | Integer   | Primärschlüssel           |
| `ulid`       | String    | Öffentlicher Bezeichner   |
| `name`       | String    | Benutzername              |
| `email`      | String    | E-Mail (eindeutig)        |
| `password`   | String    | Bcrypt-gehashtes Passwort |
| `created_at` | Timestamp | Erstellungszeitpunkt      |
| `updated_at` | Timestamp | Aktualisierungszeitpunkt  |

**categories**

| Spalte    | Typ     | Beschreibung                                      |
| --------- | ------- | ------------------------------------------------- |
| `id`      | Integer | Primärschlüssel                                   |
| `ulid`    | String  | Öffentlicher Bezeichner                           |
| `user_id` | Integer | Fremdschlüssel → users (kaskadierendes Löschen)   |
| `name`    | String  | Kategoriename (max. 255, eindeutig pro Benutzer)  |
| `icon`    | Integer | Icon-Index (Standard: 0)                          |
| `color`   | String  | Hex-Farbcode (Standard: #ffffff)                  |

**entries**

| Spalte        | Typ         | Beschreibung                                    |
| ------------- | ----------- | ----------------------------------------------- |
| `id`          | Integer     | Primärschlüssel                                 |
| `ulid`        | String      | Öffentlicher Bezeichner                         |
| `user_id`     | Integer     | Fremdschlüssel → users                          |
| `category_id` | Integer     | Fremdschlüssel → categories (nullable)          |
| `title`       | String      | Titel des Eintrags                              |
| `value`       | Text        | Inhalt (max. 5000 Zeichen)                      |
| `is_sensitive` | Boolean    | Sensibilitätsmarkierung (Standard: false)       |
| `embedding`   | Vector(1024)| pgvector-Einbettung (nullable, versteckt in API)|

**entry_shares**

| Spalte          | Typ      | Beschreibung                                |
| --------------- | -------- | ------------------------------------------- |
| `id`            | Integer  | Primärschlüssel                             |
| `ulid`          | String   | Öffentlicher Bezeichner                     |
| `entry_id`      | Integer  | Fremdschlüssel → entries                    |
| `shared_with_id`| Integer  | Fremdschlüssel → users                      |
| `permission`    | Tinyint  | 0 = Lesen, 1 = Schreiben                   |

**entry_files**

| Spalte         | Typ      | Beschreibung               |
| -------------- | -------- | -------------------------- |
| `id`           | Integer  | Primärschlüssel            |
| `ulid`         | String   | Öffentlicher Bezeichner    |
| `entry_id`     | Integer  | Fremdschlüssel → entries   |
| `original_name`| String   | Ursprünglicher Dateiname   |
| `stored_path`  | String   | Speicherpfad               |
| `mime_type`    | String   | MIME-Typ (max. 127)        |
| `file_size`    | Unsigned Int | Dateigröße in Bytes    |

---

## 6. Datei-Speicherung

DataVault unterstützt ein **steckbares Speichersystem** mit zwei Treibern:

### Lokaler Speicher (Standard)

Dateien werden auf dem Server-Dateisystem gespeichert.

**Speicherpfad-Muster:**
```
entries/{entry_ulid}/{dateiname_hash}
```

### S3-kompatibler Speicher

Pro Benutzer konfigurierbar über die Tabelle `user_storage_configs`. Die S3-Zugangsdaten werden **verschlüsselt** in der Datenbank gespeichert.

**S3-Konfigurationsfelder:**
- `key` – Zugriffsschlüssel
- `secret` – Geheimschlüssel
- `region` – AWS-Region
- `bucket` – Bucket-Name
- `endpoint` – Benutzerdefinierter Endpunkt (optional, für S3-kompatible Dienste)

### Beschränkungen

- Maximale Dateigröße: **10 MB** pro Datei
- Upload-Format: `multipart/form-data`
- Nur der Eigentümer kann Dateien löschen
- Eigentümer und Benutzer mit Freigabe können Dateien herunterladen

---

## 7. Ratenbegrenzung

| Endpunkt-Typ          | Limit         | Geltungsbereich            |
| ---------------------- | ------------- | -------------------------- |
| Authentifizierung      | 10 Anfragen/Min | Pro IP-Adresse           |
| Suche                  | 20 Anfragen/Min | Pro Benutzer-ID (oder IP)|
| Allgemeine API         | 60 Anfragen/Min | Pro Benutzer-ID (oder IP)|

---

## 8. Antwortformate

Alle Antworten werden als **JSON** zurückgegeben.

### Beispiel: Einzelner Eintrag

```json
{
  "entry": {
    "ulid": "01HXYZ...",
    "title": "Mein Passwort",
    "value": "geheim123",
    "is_sensitive": true,
    "category": {
      "ulid": "01HABC...",
      "name": "Passwörter",
      "icon": 0,
      "color": "#ff0000"
    },
    "files": [
      {
        "ulid": "01HDEF...",
        "original_name": "quittung.pdf",
        "mime_type": "application/pdf",
        "file_size": 102400
      }
    ],
    "created_at": "2026-02-25T12:00:00Z",
    "updated_at": "2026-02-25T12:00:00Z"
  }
}
```

### Beispiel: Suchergebnis

```json
{
  "query": "passwort",
  "count": 2,
  "results": [
    {
      "ulid": "01HXYZ...",
      "title": "E-Mail Passwort",
      "value": "...",
      "search_type": "semantic",
      "similarity": 0.9542,
      "is_shared": false
    },
    {
      "ulid": "01HABC...",
      "title": "Router Zugangsdaten",
      "value": "...",
      "search_type": "text",
      "similarity": null,
      "is_shared": true
    }
  ]
}
```

### Beispiel: Systemstatus

```json
{
  "status": "healthy",
  "services": {
    "database": "connected",
    "ollama": {
      "status": "ok",
      "model": "mxbai-embed-large",
      "model_loaded": true
    }
  }
}
```

---

## 9. Validierungsregeln

### Registrierung

| Feld                    | Regeln                              |
| ----------------------- | ----------------------------------- |
| `name`                  | Pflicht, String, max. 255           |
| `email`                 | Pflicht, E-Mail, max. 255, eindeutig|
| `password`              | Pflicht, min. 8 Zeichen, bestätigt  |
| `password_confirmation` | Pflicht, muss mit Passwort übereinstimmen |
| `device_name`           | Optional, String, max. 255         |

### Eintrag erstellen

| Feld            | Regeln                          |
| --------------- | ------------------------------- |
| `title`         | Pflicht, String, max. 500       |
| `value`         | Pflicht, String, max. 5000      |
| `category_ulid` | Optional, muss existieren       |
| `is_sensitive`  | Optional, Boolean               |

### Freigabe erstellen

| Feld         | Regeln                                   |
| ------------ | ---------------------------------------- |
| `email`      | Pflicht, muss in Benutzertabelle existieren |
| `permission` | Optional, Integer (0 oder 1)             |

---

## 10. Docker-Infrastruktur

DataVault wird über **Docker Compose** bereitgestellt mit folgenden Diensten:

```yaml
services:
  app:        # Laravel-Anwendung
  postgres:   # PostgreSQL mit pgvector-Erweiterung (pgvector/pgvector:pg18)
  ollama:     # Ollama KI-Dienst für Einbettungen
```

### Artisan-Befehle

| Befehl                          | Beschreibung                                        |
| ------------------------------- | --------------------------------------------------- |
| `php artisan entries:reembed`   | Alle Einbettungen neu generieren                    |
| `--user={id}`                   | Nur für einen bestimmten Benutzer neu generieren    |

---

*Weiterführend: Siehe [Suchalgorithmus-Dokumentation](Suchalgorithmus.md) für eine detaillierte Erklärung der hybriden Suchfunktion.*
