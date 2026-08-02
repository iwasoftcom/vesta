Vesta

der S3-kompatible Objektspeicher, der die Lücken schließt · v0.1.0

Ein geschichtetes, S3-kompatibles Objektspeichersystem in Rust — eine einzige Binärdatei, die vom Laptop bis zum Raft-replizierten Cluster skaliert, ohne die Software zu wechseln.

**Entwickeln Sie mit KI?** Geben Sie Ihrem Coding-Agenten/LLM statt dieser Seite diesen Link — eine dichte, maschinenoptimierte Referenz (Installation, jede Umgebungsvariable und ihr Standardwert, exakte API-Aufrufe), die er direkt nutzen kann, statt Marketingtext zurückentwickeln zu müssen: [documentation.ai.md](https://iwasoft.com/products/vesta/0.1.0/docs/documentation.ai.md)

## Was Vesta ist

Vesta zielt auf die Funktionslücken heutiger Objektspeicher (S3, GCS, Azure Blob, R2, B2, Wasabi, MinIO, Ceph, SeaweedFS, Garage). Es spricht die echte S3-API — SigV4-Signierung, Multipart-Upload, Versionierung, bedingte Anfragen, Batch-Löschung — und trennt die **Kontrollebene** (Metadaten: Buckets, Objektindex, IAM) vollständig von der **Datenebene** (inhalts­ adressierte Blöcke auf der Festplatte), sodass beide unabhängig voneinander ausgetauscht oder skaliert werden können.

## Designprinzipien

**Trennung von Kontroll- und Datenebene.**  
Metadaten und Bytes leben hinter getrennten Trait-Grenzen. Speicher-Engines, Konsens-Backends und Verschlüsselungsschichten werden ausgetauscht, ohne die S3-API-Schicht anzufassen.

**Kein Admin-Schalter bleibt in einer Konfigdatei liegen.**  
Ratenbegrenzung, GC-Intervalle, CORS, Kontingente und Richtlinien sind Laufzeiteinstellungen — repliziert und live über die Admin-Konsole geändert, nicht über Umgebungsvariablen, die einen Neustart erfordern.

**Kompatibilität ist ein Vertrag, keine Annäherung.**  
SigV4 (Header, Presigned URLs, Streaming-Chunks), Multipart, Versionierung und bedingte Anfragen werden gegen echte AWS-SDK-Testsuiten geprüft, nicht gegen handverlesene Beispiele.

## Der Unterschied zu einem typischen Single-Binary-Objektspeicher

|  | Typischer MinIO-artiger Speicher | Vesta |
| --- | --- | --- |
| Konsens | Festes Erasure-Set- / Gateway-Modell | Netzwerk-Raft mit dynamischer Mitgliedschaft — eine bewährte Engine ([openraft](#architecture)) lässt sich als Opt-in-Backend hinter demselben Schreibpfad einsetzen |
| Laufzeitkonfiguration | Umgebungsvariablen, Neustart zum Ändern | Admin-Konsole ändert Live-Einstellungen (Ratenlimit, GC-Intervall, CORS, Kontingente) über das replizierte Log — kein Neustart |
| Metadaten-Dauerhaftigkeit | Je nach Backend unterschiedlich | Append-only-WAL mit Snapshot-Kompaktierung; jeder Knoten schreibt unabhängig dauerhaft und holt über normale Log-Replikation auf |
| Mandantenfähigkeit | Nachträglich angeflanscht oder fehlend | Mandanten erster Klasse mit Bucket-Kontingenten pro Mandant und SigV4-Identitätsisolierung |
| KI-Agentenzugriff | Nicht zutreffend | Ein schreibgeschützter [MCP-Server](#more) stellt native Suche und S3 Select als Agenten-Tools bereit, mit mandantenscharfer Isolierung pro Schlüssel |

## Schnellstart

Server starten (Container-Image oder eigenständige Binärdatei):

```
# Docker
docker run -p 9000:9000 iwasoftcom/vesta:0.1.0

# oder die Binärdatei
VESTA_DATA_DIR=/var/lib/vesta vesta
```

Mit jedem beliebigen S3-Client ansprechen:

```
aws --endpoint-url http://127.0.0.1:9000 s3 mb s3://fotos
aws --endpoint-url http://127.0.0.1:9000 s3 cp ./x.jpg s3://fotos/x.jpg
aws --endpoint-url http://127.0.0.1:9000 s3 ls s3://fotos
```

## Was drinsteckt

**Ratenbegrenzung**  
Token-Bucket pro Mandant, live über die Admin-Konsole aktiviert und feinjustiert; Fehlverhalten wird mit einem sauberen `SlowDown` beantwortet, nicht mit einer gekappten Verbindung.

**Verteilter Konsens**  
Ein Netzwerk-Raft mit Leader-Wahl, dynamischer Mitgliedschaft und dauerhafter Log-Replikation — oder Opt-in zu `openraft`, einer bewährten Implementierung, hinter identischem Schreibpfad.

**Erasure Coding & Verschlüsselung**  
Reed-Solomon-erasure-codierter Speicher und AES-256-GCM-Verschlüsselung im Ruhezustand, beide dedup-sicher (inhaltsadressierte Blöcke).

**Versionierung & Objektsperre**  
Vollständige Versionshistorie, Löschmarkierungen und WORM-Aufbewahrung (GOVERNANCE/COMPLIANCE) mit rechtlicher Aufbewahrung.

**Mandantenfähigkeit**  
Mandanten sind erster Klasse: Bucket-Kontingente pro Mandant, SigV4-Identitätsisolierung, Bucket-Richtlinien und vordefinierte ACLs.

**Suche, Select & Lambda**  
Native invertierte-Index-Metadatensuche, S3 Select (CSV-SQL) und Transform-on-Read (im Stil von Object Lambda).

**Replikation & Ereignisse**  
Asynchrone Geo-Replikation, ein Change-Stream-Event-Bus und steckbare Webhook-Benachrichtigungszustellung.

**Lifecycle & Bestand**  
Ablauf- und Speicherklassen-Übergangsregeln sowie CSV-Bestandsberichte auf Anfrage.

## Admin-Konsole

Eine separate, zustandslose Verwaltungsanwendung (eingebettete React+MUI-Oberfläche) leitet Schreibvorgänge an einen Speicherknoten weiter — sie hält keine eigenen Daten; jede Änderung wird über dasselbe Konsens-Log repliziert, das auch die S3-API nutzt.

<table><tbody><tr><th>Adresse</th><td><code>http://localhost:9500</code> (Env <code>VESTA_ADMIN_ADDR</code>, Standard <code>0.0.0.0:9500</code>)</td></tr><tr><th>Verbindet zu</th><td>der Admin-API eines Speicherknotens, Standard <code>http://127.0.0.1:9000</code> (Env <code>VESTA_ADMIN_NODES</code>)</td></tr><tr><th>Standardbenutzer</th><td>keiner — die Konsole ist offen und agiert als Admin, bis der <b>erste</b> Konsolenbenutzer angelegt wird (Benutzer-Bildschirm), was dieses Fenster schließt. Oder beim Knotenstart vorbelegen: <code>VESTA_ADMIN_USER</code>/<code>VESTA_ADMIN_PASS</code></td></tr></tbody></table>

Jeder Knoten bietet dieselben Operationen auch als einfache JSON-API unter `http://<knoten>:9000/_vesta/admin/*` (dieselben Endpunkte, die die Konsole selbst aufruft) — praktisch zum Skripten. Die ersten drei Schritte:

```
# 1) Bucket anlegen
curl -X POST http://127.0.0.1:9000/_vesta/admin/buckets \
  -H 'content-type: application/json' -d '{"name":"fotos"}'

# 2) Mandant anlegen (vor einem API-Schlüssel erforderlich)
curl -X POST http://127.0.0.1:9000/_vesta/admin/tenants \
  -H 'content-type: application/json' -d '{"name":"acme-corp"}'

# 3) API-Schlüssel (SigV4 Access/Secret-Paar) für diesen Mandanten anlegen
curl -X POST http://127.0.0.1:9000/_vesta/admin/keys \
  -H 'content-type: application/json' -d '{"tenant":"acme-corp"}'
# → {"access_key":"VESTA...","secret_key":"...","tenant":"acme-corp"}
```

Sobald ein Konsolenbenutzer oder API-Schlüssel existiert, benötigen diese Aufrufe die Header `x-vesta-user`/`x-vesta-pass` (die Zugangsdaten eines Konsolenbenutzers) — und das Anlegen des ersten Schlüssels schaltet die SigV4-Pflicht für die S3-API clusterweit automatisch ein, ohne Neustart.

-   **Benutzer, Schlüssel & Mandanten** — Konsolenkonten, SigV4-API-Schlüssel, Kontingente pro Mandant.
-   **Buckets & Richtlinien** — Erstellen/Löschen, Bucket-Richtlinien-JSON, Public-Read-Schalter.
-   **Cluster** — Knotenzustand, Mitglieder hinzufügen/entfernen, Minority-Write- und Auto-Shrink-Schalter.
-   **Laufzeiteinstellungen** — Ratenlimit, Block-GC-Intervall, CORS-Origin: live geändert, an jeden Knoten repliziert, über Neustarts hinweg persistent.

## Architektur

Eine einzige Binärdatei, zwei Netzwerktüren und eine strikte Schichtungsregel: Die S3-API-Schicht fasst den Speicher nie direkt an — alles läuft über den Koordinator, und jede Mutation, die clusterweit gelten muss, durchläuft das Konsens-Log, bevor sie für eine Leseoperation sichtbar wird.

S3-SDKs · aws-cli SigV4 · Multipart · Versionierung Admin-Konsole · KI-Agenten (MCP) zustandsloser Proxy · mandantenscharfe Tools S3-API · :9000 Admin-API · :9500 Koordinator (Rust): Buckets · Objekte · Multipart · Richtlinien · Lifecycle · Suche Konsens-Log (eigenes Raft oder openraft) — Mutationen werden committed, bevor sie lesbar sind Metadaten (WAL) · Blockspeicher (erasure-codiert, verschlüsselt, dedupliziert)

## Downloads & Quelle

-   **Downloads:** kompilierte Artefakte (Windows, Debian `.deb`, RedHat `.rpm`) und das Docker-Image werden pro Version auf [iwasoft.com](https://iwasoft.com) → Produkte → Vesta veröffentlicht. Quellcode ist nicht Teil der Downloads.
-   **Kompatibilität:** Die S3-API-Oberfläche (SigV4, Multipart, Versionierung, bedingte Anfragen) wird laufend gegen echte AWS-SDK-Integrationstests geprüft.
-   **Ehrlicher Status:** frühe Entwicklungsphase — noch nicht unabhängig sicherheitsgeprüft, noch keine Produktionserfahrung. Das sind Offenlegungen, keine Einschränkungen der Roadmap.

Vesta v0.1.0 · S3-kompatibel · Rust, inhaltsadressierter Speicher, Netzwerk-Raft (eigen oder openraft). © iwasoft.
