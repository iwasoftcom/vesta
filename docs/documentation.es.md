Vesta

el almacenamiento de objetos compatible con S3 que cierra las brechas · v0.1.0

Un sistema de almacenamiento de objetos compatible con S3, escrito en Rust — un único binario que escala desde un portátil hasta un clúster replicado por Raft, sin cambiar de software.

**¿Desarrollas con IA?** Dale a tu agente de código/LLM este enlace en lugar de esta página — una referencia densa y optimizada para máquinas (instalación, cada variable de entorno con su valor por defecto, llamadas API exactas) que puede usar directamente, sin tener que reconstruir hechos a partir de texto de marketing: [documentation.ai.md](https://iwasoft.com/products/vesta/0.1.0/docs/documentation.ai.md)

## Qué es Vesta

Vesta apunta a las carencias funcionales presentes en las soluciones de almacenamiento de objetos actuales (S3, GCS, Azure Blob, R2, B2, Wasabi, MinIO, Ceph, SeaweedFS, Garage). Habla la API S3 real — firma SigV4, carga multiparte, versionado, solicitudes condicionales, eliminación por lotes — y separa por completo el **plano de control** (metadatos: buckets, índice de objetos, IAM) del **plano de datos** (bloques direccionados por contenido en disco), de modo que cada uno pueda reemplazarse o escalarse de forma independiente.

## Principios de diseño

**Separación entre plano de control y plano de datos.**  
Metadatos y bytes viven detrás de fronteras de trait separadas. Motores de almacenamiento, backends de consenso y capas de cifrado se intercambian sin tocar la capa de la API S3.

**Ningún interruptor de administración se queda olvidado en un archivo de configuración.**  
Límites de tasa, intervalos de GC, CORS, cuotas y políticas son ajustes en tiempo de ejecución — replicados y cambiados en vivo desde la consola de administración, no variables de entorno que requieren reinicio.

**La compatibilidad es un contrato, no una aproximación.**  
SigV4 (cabeceras, URLs prefirmadas, fragmentos en streaming), multiparte, versionado y solicitudes condicionales se verifican continuamente frente a suites de pruebas reales del SDK de AWS, no ejemplos elegidos a mano.

## En qué se diferencia de un almacén de objetos típico de un solo binario

|  | Almacén típico al estilo MinIO | Vesta |
| --- | --- | --- |
| Consenso | Modelo fijo de conjunto de borrado / gateway | Raft en red con membresía dinámica — un motor probado ([openraft](#architecture)) se conecta de forma opcional detrás de la misma ruta de escritura |
| Configuración en tiempo de ejecución | Variables de entorno, reinicio para cambiar | La consola de administración modifica ajustes en vivo (límite de tasa, intervalo de GC, CORS, cuotas) a través del log replicado — sin reinicio |
| Durabilidad de metadatos | Varía según el backend | WAL de solo anexado con compactación por instantánea; cada nodo persiste de forma independiente y se pone al día mediante replicación normal del log |
| Multiinquilino | Añadido después o ausente | Inquilinos (tenants) de primera clase, cuotas de buckets por inquilino y aislamiento de identidad SigV4 |
| Acceso de agentes de IA | No aplicable | Un [servidor MCP](#more) de solo lectura expone la búsqueda nativa y S3 Select como herramientas de agente, con aislamiento por inquilino y por clave |

## Inicio rápido

Ejecuta el servidor (imagen de contenedor o binario independiente):

```
# Docker
docker run -p 9000:9000 iwasoftcom/vesta:0.1.0

# o el binario
VESTA_DATA_DIR=/var/lib/vesta vesta
```

Habla con él usando cualquier cliente S3:

```
aws --endpoint-url http://127.0.0.1:9000 s3 mb s3://fotos
aws --endpoint-url http://127.0.0.1:9000 s3 cp ./x.jpg s3://fotos/x.jpg
aws --endpoint-url http://127.0.0.1:9000 s3 ls s3://fotos
```

## Qué incluye

**Limitación de tasa**  
Cubo de tokens por inquilino, activado y ajustado en vivo desde la consola de administración; un cliente que se comporta mal recibe un `SlowDown` apropiado, no una conexión cortada.

**Consenso distribuido**  
Un Raft en red con elección de líder, membresía dinámica y replicación duradera del log — o activa `openraft`, una implementación probada, detrás de la misma ruta de escritura.

**Codificación de borrado y cifrado**  
Almacenamiento codificado por borrado Reed-Solomon y cifrado AES-256-GCM en reposo, ambos seguros para la deduplicación (bloques direccionados por contenido).

**Versionado y bloqueo de objetos**  
Historial completo de versiones, marcadores de eliminación y retención WORM (GOVERNANCE/COMPLIANCE) con retención legal.

**Multiinquilino**  
Los inquilinos son de primera clase: cuotas de buckets por inquilino, aislamiento de identidad SigV4, políticas de bucket y ACL predefinidas.

**Búsqueda, Select y Lambda**  
Búsqueda nativa de metadatos por índice invertido, S3 Select (SQL sobre CSV) y transformación en lectura (al estilo Object Lambda).

**Replicación y eventos**  
Replicación geográfica asíncrona, un bus de eventos de flujo de cambios y entrega de notificaciones webhook conectable.

**Ciclo de vida e inventario**  
Reglas de expiración y transición de clase de almacenamiento, además de informes de inventario CSV bajo demanda.

## Consola de administración

Una aplicación de gestión separada y sin estado (interfaz React + MUI embebida) retransmite las escrituras a un nodo de almacenamiento — no conserva datos propios; cada cambio se replica a través del mismo log de consenso que usa la API S3.

<table><tbody><tr><th>Dirección</th><td><code>http://localhost:9500</code> (env <code>VESTA_ADMIN_ADDR</code>, por defecto <code>0.0.0.0:9500</code>)</td></tr><tr><th>Se conecta a</th><td>la API de administración de un nodo de almacenamiento, por defecto <code>http://127.0.0.1:9000</code> (env <code>VESTA_ADMIN_NODES</code>)</td></tr><tr><th>Usuario por defecto</th><td>ninguno — la consola está abierta y actúa como admin hasta que se crea el <b>primer</b> usuario de consola (pantalla Usuarios), lo que cierra esa ventana. O precárgalo al iniciar el nodo: <code>VESTA_ADMIN_USER</code>/<code>VESTA_ADMIN_PASS</code></td></tr></tbody></table>

Cada nodo también expone las mismas operaciones como una API JSON sencilla en `http://<nodo>:9000/_vesta/admin/*` (los mismos endpoints que llama la propia consola) — útil para automatizar. Las tres primeras cosas que harás:

```
# 1) Crear un bucket
curl -X POST http://127.0.0.1:9000/_vesta/admin/buckets \
  -H 'content-type: application/json' -d '{"name":"fotos"}'

# 2) Crear un inquilino (requerido antes de una clave API)
curl -X POST http://127.0.0.1:9000/_vesta/admin/tenants \
  -H 'content-type: application/json' -d '{"name":"acme-corp"}'

# 3) Crear una clave API (par access/secret SigV4) para ese inquilino
curl -X POST http://127.0.0.1:9000/_vesta/admin/keys \
  -H 'content-type: application/json' -d '{"tenant":"acme-corp"}'
# → {"access_key":"VESTA...","secret_key":"...","tenant":"acme-corp"}
```

En cuanto exista un usuario de consola o una clave API, estas llamadas necesitan las cabeceras `x-vesta-user`/`x-vesta-pass` (las credenciales de un usuario de consola) — y crear esa primera clave activa automáticamente la obligatoriedad de SigV4 para la API S3, en todo el clúster, sin reiniciar.

-   **Usuarios, claves e inquilinos** — cuentas de consola, claves de API SigV4, cuotas por inquilino.
-   **Buckets y políticas** — crear/eliminar, JSON de política de bucket, interruptores de lectura pública.
-   **Clúster** — salud de nodos, añadir/quitar miembros, interruptores de escritura minoritaria y auto-reducción.
-   **Ajustes en tiempo de ejecución** — límite de tasa, intervalo de GC de bloques, origen CORS: cambiados en vivo, replicados a cada nodo, persistentes entre reinicios.

## Arquitectura

Un único binario, dos puertas de red y una regla de estratificación estricta: la capa de la API S3 nunca toca el almacenamiento directamente — todo pasa por el coordinador, y toda mutación que deba aplicarse a todo el clúster pasa por el log de consenso antes de ser visible en una lectura.

SDKs de S3 · aws-cli SigV4 · multiparte · versionado Consola admin · agentes de IA (MCP) proxy sin estado · herramientas por inquilino API S3 · :9000 API admin · :9500 coordinador (Rust): buckets · objetos · multiparte · políticas · ciclo de vida · búsqueda log de consenso (Raft propio u openraft) — las mutaciones se confirman antes de poder leerse metadatos (WAL) · almacenamiento de bloques (codificado por borrado, cifrado, deduplicado)

## Descargas y fuente

-   **Descargas:** artefactos compilados (Windows, Debian `.deb`, RedHat `.rpm`) y la imagen Docker se publican por versión en [iwasoft.com](https://iwasoft.com) → Productos → Vesta. El código fuente no forma parte de las descargas.
-   **Compatibilidad:** la superficie de la API S3 (SigV4, multiparte, versionado, solicitudes condicionales) se verifica continuamente frente a pruebas de integración reales del SDK de AWS.
-   **Estado honesto:** desarrollo temprano — aún sin auditoría de seguridad independiente, sin kilometraje en producción todavía. Son divulgaciones, no reservas sobre la hoja de ruta.

Vesta v0.1.0 · compatible con S3 · Rust, almacenamiento direccionado por contenido, Raft en red (propio u openraft). © iwasoft.
