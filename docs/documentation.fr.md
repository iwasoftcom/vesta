Vesta

le stockage d'objets compatible S3 qui comble les lacunes · v0.1.0

Un système de stockage d'objets compatible S3, écrit en Rust — un seul binaire qui évolue d'un ordinateur portable jusqu'à un cluster répliqué par Raft, sans changer de logiciel.

**Vous développez avec l'IA ?** Donnez à votre agent de codage/LLM ce lien plutôt que cette page — une référence dense, optimisée pour la machine (installation, chaque variable d'environnement et sa valeur par défaut, appels API exacts) qu'il peut utiliser directement, sans avoir à rétro-ingénierer un texte marketing : [documentation.ai.md](https://iwasoft.com/products/vesta/0.1.0/docs/documentation.ai.md)

## Ce qu'est Vesta

Vesta cible les lacunes fonctionnelles observées dans les solutions de stockage d'objets actuelles (S3, GCS, Azure Blob, R2, B2, Wasabi, MinIO, Ceph, SeaweedFS, Garage). Il parle la véritable API S3 — signature SigV4, téléversement multipart, versionnage, requêtes conditionnelles, suppression par lot — et sépare entièrement le **plan de contrôle** (métadonnées : buckets, index d'objets, IAM) du **plan de données** (blocs adressés par contenu sur disque), afin que chacun puisse être remplacé ou mis à l'échelle indépendamment.

## Principes de conception

**Séparation plan de contrôle / plan de données.**  
Métadonnées et octets vivent derrière des frontières de traits séparées. Moteurs de stockage, backends de consensus et couches de chiffrement se remplacent sans toucher à la couche API S3.

**Aucun réglage d'administration oublié dans un fichier de config.**  
Limitation de débit, intervalles de GC, CORS, quotas et politiques sont des paramètres d'exécution — répliqués et modifiés en direct depuis la console d'administration, pas des variables d'environnement nécessitant un redémarrage.

**La compatibilité est un contrat, pas une approximation.**  
SigV4 (en-têtes, URLs présignées, chunks en streaming), multipart, versionnage et requêtes conditionnelles sont vérifiés en continu face à de vraies suites de tests du SDK AWS, pas des exemples triés sur le volet.

## Différence avec un stockage d'objets mono-binaire classique

|  | Stockage typique façon MinIO | Vesta |
| --- | --- | --- |
| Consensus | Modèle fixe d'ensemble à effacement / passerelle | Raft en réseau à appartenance dynamique — un moteur éprouvé ([openraft](#architecture)) s'enclenche en option derrière le même chemin d'écriture |
| Configuration à l'exécution | Variables d'environnement, redémarrage pour modifier | La console d'administration modifie les paramètres en direct (limite de débit, intervalle de GC, CORS, quotas) via le journal répliqué — sans redémarrage |
| Durabilité des métadonnées | Variable selon le backend | WAL en ajout seul avec compaction par instantané ; chaque nœud persiste indépendamment et rattrape son retard par réplication normale du journal |
| Multi-tenant | Ajouté après coup ou absent | Tenants de première classe, quotas de buckets par tenant et isolation d'identité SigV4 |
| Accès par agent IA | Non applicable | Un [serveur MCP](#more) en lecture seule expose la recherche native et S3 Select comme outils d'agent, avec isolation par tenant et par clé |

## Démarrage rapide

Lancer le serveur (image conteneur, ou binaire autonome) :

```
# Docker
docker run -p 9000:9000 iwasoftcom/vesta:0.1.0

# ou le binaire
VESTA_DATA_DIR=/var/lib/vesta vesta
```

Communiquer avec n'importe quel client S3 :

```
aws --endpoint-url http://127.0.0.1:9000 s3 mb s3://photos
aws --endpoint-url http://127.0.0.1:9000 s3 cp ./x.jpg s3://photos/x.jpg
aws --endpoint-url http://127.0.0.1:9000 s3 ls s3://photos
```

## Ce qu'il y a à l'intérieur

**Limitation de débit**  
Seau à jetons par tenant, activé et ajusté en direct depuis la console d'administration ; un client indiscipliné reçoit un `SlowDown` propre, pas une connexion coupée.

**Consensus distribué**  
Un Raft en réseau avec élection du leader, appartenance dynamique et réplication durable du journal — ou optez pour `openraft`, une implémentation éprouvée, derrière un chemin d'écriture identique.

**Codage à effacement et chiffrement**  
Stockage codé à effacement Reed-Solomon et chiffrement AES-256-GCM au repos, tous deux compatibles avec la déduplication (blocs adressés par contenu).

**Versionnage et verrouillage d'objet**  
Historique complet des versions, marqueurs de suppression et rétention WORM (GOVERNANCE/COMPLIANCE) avec conservation légale.

**Multi-tenant**  
Les tenants sont de première classe : quotas de buckets par tenant, isolation d'identité SigV4, politiques de bucket et ACL prédéfinies.

**Recherche, Select et Lambda**  
Recherche de métadonnées native par index inversé, S3 Select (SQL sur CSV) et transformation à la lecture (façon Object Lambda).

**Réplication et événements**  
Réplication géographique asynchrone, un bus d'événements en flux de changements, et une livraison de notifications webhook enfichable.

**Cycle de vie et inventaire**  
Règles d'expiration et de transition de classe de stockage, plus des rapports d'inventaire CSV à la demande.

## Console d'administration

Une application de gestion séparée et sans état (interface React + MUI embarquée) relaie les écritures vers un nœud de stockage — elle ne conserve aucune donnée propre ; chaque modification est répliquée via le même journal de consensus que celui utilisé par l'API S3.

<table><tbody><tr><th>Adresse</th><td><code>http://localhost:9500</code> (env <code>VESTA_ADMIN_ADDR</code>, défaut <code>0.0.0.0:9500</code>)</td></tr><tr><th>Se connecte à</th><td>l'API d'administration d'un nœud de stockage, défaut <code>http://127.0.0.1:9000</code> (env <code>VESTA_ADMIN_NODES</code>)</td></tr><tr><th>Utilisateur par défaut</th><td>aucun — la console est ouverte et agit comme admin jusqu'à la création du <b>premier</b> utilisateur de la console (écran Utilisateurs), ce qui referme cette fenêtre. Ou préconfigurez-en un au démarrage du nœud : <code>VESTA_ADMIN_USER</code>/<code>VESTA_ADMIN_PASS</code></td></tr></tbody></table>

Chaque nœud expose aussi les mêmes opérations comme une simple API JSON à `http://<nœud>:9000/_vesta/admin/*` (les mêmes points de terminaison que la console elle-même appelle) — pratique pour scripter. Les trois premières choses à faire :

```
# 1) Créer un bucket
curl -X POST http://127.0.0.1:9000/_vesta/admin/buckets \
  -H 'content-type: application/json' -d '{"name":"photos"}'

# 2) Créer un tenant (requis avant une clé API)
curl -X POST http://127.0.0.1:9000/_vesta/admin/tenants \
  -H 'content-type: application/json' -d '{"name":"acme-corp"}'

# 3) Créer une clé API (paire access/secret SigV4) pour ce tenant
curl -X POST http://127.0.0.1:9000/_vesta/admin/keys \
  -H 'content-type: application/json' -d '{"tenant":"acme-corp"}'
# → {"access_key":"VESTA...","secret_key":"...","tenant":"acme-corp"}
```

Dès qu'un utilisateur de la console ou une clé API existe, ces appels nécessitent les en-têtes `x-vesta-user`/`x-vesta-pass` (les identifiants d'un utilisateur de la console) — et la création de cette première clé active automatiquement l'obligation SigV4 pour l'API S3, sur tout le cluster, sans redémarrage.

-   **Utilisateurs, clés & tenants** — comptes de la console, clés API SigV4, quotas par tenant.
-   **Buckets & politiques** — création/suppression, JSON de politique de bucket, bascules de lecture publique.
-   **Cluster** — santé des nœuds, ajout/retrait de membres, bascules d'écriture minoritaire et de réduction automatique.
-   **Paramètres d'exécution** — limite de débit, intervalle de GC des blocs, origine CORS : modifiés en direct, répliqués sur chaque nœud, persistants entre les redémarrages.

## Architecture

Un seul binaire, deux portes réseau, et une règle de stratification stricte : la couche API S3 ne touche jamais directement au stockage — tout passe par le coordinateur, et toute mutation devant être valable pour l'ensemble du cluster passe par le journal de consensus avant d'être visible en lecture.

SDK S3 · aws-cli SigV4 · multipart · versionnage Console admin · agents IA (MCP) proxy sans état · outils par tenant API S3 · :9000 API admin · :9500 coordinateur (Rust) : buckets · objets · multipart · politiques · cycle de vie · recherche journal de consensus (Raft propre ou openraft) — les mutations sont commitées avant d'être lisibles métadonnées (WAL) · stockage de blocs (codé à effacement, chiffré, dédupliqué)

## Téléchargements & source

-   **Téléchargements :** artefacts compilés (Windows, Debian `.deb`, RedHat `.rpm`) et l'image Docker sont publiés pour chaque version sur [iwasoft.com](https://iwasoft.com) → Produits → Vesta. Le code source ne fait pas partie des téléchargements.
-   **Compatibilité :** la surface de l'API S3 (SigV4, multipart, versionnage, requêtes conditionnelles) est vérifiée en continu face à de véritables tests d'intégration du SDK AWS.
-   **État honnête :** développement précoce — pas encore audité indépendamment en matière de sécurité, aucun kilométrage en production. Ce sont des divulgations, pas des réserves sur la feuille de route.

Vesta v0.1.0 · compatible S3 · Rust, stockage adressé par contenu, Raft en réseau (propre ou openraft). © iwasoft.
