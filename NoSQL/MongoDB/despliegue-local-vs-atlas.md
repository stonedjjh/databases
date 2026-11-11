# Entornos de Despliegue: MongoDB Community Local vs MongoDB Atlas (Cloud)

![Engine](https://img.shields.io/badge/Engine-MongoDB-47A248?style=flat&logo=mongodb&logoColor=white)
![Focus](https://img.shields.io/badge/Focus-Local%20vs%20Cloud%20DBaaS-blue)
![Level](https://img.shields.io/badge/Level-DevOps%20%26%20Architecture-orange)

Guía técnica sobre los dos modelos principales de despliegue y operación de MongoDB: instalación local autocontenida (*Self-Hosted / On-Premise*) mediante **MongoDB Community Server** o Docker, frente al modelo administrado en la nube (*Database-as-a-Service - DBaaS*) con **MongoDB Atlas**.

---

## Indice de Contenido

- [1. Modelos de Operacion: Autoadministrado (Local / On-Premise) vs Cloud DBaaS](#1-modelos-de-operacion-autoadministrado-local--on-premise-vs-cloud-dbaas)
- [2. Entornos Autoadministrados (Self-Managed): Community Edition vs Enterprise Advanced](#2-entornos-autoadministrados-self-managed-community-edition-vs-enterprise-advanced)
  - [MongoDB Community Edition (Uso Libre / Desarrollo)](#mongodb-community-edition-uso-libre--desarrollo)
  - [MongoDB Enterprise Advanced (Entornos Corporativos On-Premise)](#mongodb-enterprise-advanced-entornos-corporativos-on-premise)
  - [Comparativa: Community vs Enterprise Advanced](#comparativa-community-vs-enterprise-advanced)
  - [Ecosistema de Binarios Locales (`mongod`, `mongosh`)](#ecosistema-de-binarios-locales-mongod-mongosh)
  - [Despliegue Aislado con Docker / Docker Compose](#despliegue-aislado-con-docker--docker-compose)
- [3. MongoDB Atlas (Cloud DBaaS)](#3-mongodb-atlas-cloud-dbaas)
  - [Que es MongoDB Atlas](#que-es-mongodb-atlas)
  - [Niveles de Servicio: Free Tier (M0 Sandbox) vs Clusters Dedicados](#niveles-de-servicio-free-tier-m0-sandbox-vs-clusters-dedicados)
  - [Arquitectura de Seguridad: Network Access (IP Access List) y Database Users](#arquitectura-de-seguridad-network-access-ip-access-list-y-database-users)
  - [Despliegue Multi-Cloud (AWS, Google Cloud, Azure)](#despliegue-multi-cloud-aws-google-cloud-azure)
- [4. Anatomia del Connection String (URI de Conexion)](#4-anatomia-del-connection-string-uri-de-conexion)
  - [URI Estandar (`mongodb://`) vs DNS Seedlist (`mongodb+srv://`)](#uri-estandar-mongodb-vs-dns-seedlist-mongodbsrv)
  - [Parametros de Conexion Habituales (`authSource`, `retryWrites`, `w`)](#parametros-de-conexion-habituales-authsource-retrywrites-w)
- [5. Herramientas de Interaccion del Desarrollador](#5-herramientas-de-interaccion-del-desarrollador)
  - [MongoDB Shell (`mongosh`)](#mongodb-shell-mongosh)
  - [MongoDB Compass (Interfaz Grafica Oficial)](#mongodb-compass-interfaz-grafica-oficial)
  - [Extensiones de IDE (VS Code MongoDB Plugin)](#extensiones-de-ide-vs-code-mongodb-plugin)
- [6. Matriz Comparativa: Local vs Atlas](#6-matriz-comparativa-local-vs-atlas)

---

## 1. Modelos de Operacion: Autoadministrado (Local / On-Premise) vs Cloud DBaaS

Al trabajar con MongoDB, el equipo de desarrollo e infraestructura debe elegir entre dos enfoques operacionales radicalmente distintos:

```
[ Entorno Autoadministrado (Self-Managed) ]    [ Enfoque Cloud DBaaS (Atlas) ]
(Community Edition o Enterprise Advanced)
┌────────────────────────────────────────┐     ┌──────────────────────────────┐
│ Codigo de la Aplicacion                │     │ Codigo de la Aplicacion      │
├────────────────────────────────────────┤     ├──────────────────────────────┤
│ Administracion de Indices y Modelado   │     │ Administracion de Indices    │
├────────────────────────────────────────┤     ├──────────────────────────────┤
│ Seguridad, Red, Firewall y Usuarios    │     │ Seguridad y Usuarios         │
├────────────────────────────────────────┤     ├──────────────────────────────┤
│ Estrategia de Backups y Recuperacion   │     │ (Gestionado por Atlas)       │
├────────────────────────────────────────┤     │ - Backups continuos          │
│ Parches de SO, Actualizacion del Motor │     │ - Failover automatico (HA)   │
├────────────────────────────────────────┤     │ - Parches y actualizaciones  │
│ Hardware, Almacenamiento, RAM y Red    │     │ - Escalado con un clic       │
└────────────────────────────────────────┘     └──────────────────────────────┘
Responsabilidad: 100% Tu Equipo                 Responsabilidad: Compartida / Atlas
```

---

## 2. Entornos Autoadministrados (Self-Managed): Community Edition vs Enterprise Advanced

Cuando una organizacion decide ejecutar MongoDB en su propia infraestructura (localmente en laptops de desarrollo, en servidores fisicos de su datacenter o en instancias IaaS de AWS EC2 / Google Compute Engine), tiene dos ediciones para elegir:

### MongoDB Community Edition (Uso Libre / Desarrollo)

**MongoDB Community Edition** es la edicion de codigo fuente disponible y gratuita para la comunidad.
- **Licencia:** Distribuida bajo la licencia **SSPL** (*Server Side Public License*).
- **Proposito:** La opcion predilecta para aprendizaje, proyectos personales, startups en etapa inicial y desarrollo local.
- **Limitaciones en Produccion Corporativa:** No incluye soporte oficial de ingenieria 24/7 de MongoDB Inc., carece de herramientas avanzadas de auditoria de seguridad para normativas bancarias (PCI-DSS, HIPAA) y no dispone de integracion nativa con directorios corporativos avanzados (LDAP, Kerberos) ni cifrado en memoria sin costo extra.

### MongoDB Enterprise Advanced (Entornos Corporativos On-Premise)

**MongoDB Enterprise Advanced** es la suscripcion comercial disenada para grandes corporaciones que necesitan mantener sus datos **autoadministrados on-premise** (por regulaciones de soberania de datos, banca o gobierno) pero con estandares de nivel empresarial:

- **Seguridad y Cumplimiento Avanzado:**
  - **Autenticacion Corporativa:** Integracion nativa con **LDAP**, **Active Directory** y **Kerberos**.
  - **Auditoria Exhaustiva:** Registro detallado de operaciones de lectura, escritura y cambios de esquema para cumplir auditorias regulatorias (SOX, HIPAA).
  - **Cifrado en Reposo Nativo (*Encryption at Rest*):** Cifrado transparente del motor WiredTiger gestionado mediante servidores de claves KMIP (*Key Management Interoperability Protocol*).
- **MongoDB Ops Manager:**
  - Plataforma de gestion centralizada que se instala en el propio datacenter corporativo.
  - Automatiza despliegues, parches con cero tiempo de inactividad (*zero-downtime rolling upgrades*), monitoreo avanzado en tiempo real y copias de seguridad consistentes con recuperacion a un punto en el tiempo (*Point-in-Time Recovery*).
- **Herramientas de Analisis e Integracion:**
  - **MongoDB Enterprise Kubernetes Operator:** Gestion automatizada de clusters sobre OpenShift / Kubernetes corporativo.
  - **MongoDB Connector for BI:** Permite conectar herramientas SQL como Tableau o PowerBI directamente contra MongoDB mediante un protocolo relacional emulado.

### Comparativa: Community vs Enterprise Advanced

| Caracteristica | Community Edition | Enterprise Advanced |
| :--- | :--- | :--- |
| **Modelo de Licencia** | Gratuita (SSPL) | Suscripcion comercial corporativa |
| **Soporte Tecnico Oficial** | Solo foro de la comunidad | Soporte 24/7/365 con SLAs garantizados |
| **Infraestructura** | Autoadministrada (Self-Managed) | Autoadministrada (On-Premise / Datacenter propio) |
| **Herramienta de Gestion** | Scripts manuales / Herramientas de terceros | **MongoDB Ops Manager** integrado |
| **Autenticacion** | SCRAM (usuario y password), Certificados x.509 | LDAP, Active Directory, Kerberos, x.509, SCRAM |
| **Auditoria de Seguridad** | No disponible | Registro de auditoria configurable granular |
| **Cifrado en Reposo** | No nativo | Cifrado transparente con integracion KMIP / Vault |

---

### Ecosistema de Binarios Locales (`mongod`, `mongosh`)

Independientemente de si se utiliza Community o Enterprise, el entorno local interactua con los mismos componentes fundamentales:

1. **`mongod`:** El proceso demonio servidor en segundo plano. Escucha peticiones por defecto en el puerto `27017` y administra los archivos de base de datos en disco (por defecto en `/data/db` en Linux/macOS o en `C:\Program Files\MongoDB\Server\<version>\data` en Windows).
2. **`mongosh` (*Mongo Shell*):** El cliente de linea de comandos moderno basado en Node.js que permite conectarse al servidor `mongod`, ejecutar consultas interactivas, scripts administrativos y sintaxis JavaScript.

### Despliegue Aislado con Docker / Docker Compose

En entornos de desarrollo profesionales, es habitual evitar la instalacion manual en el sistema operativo anfitrion y levantar una instancia de Community Edition aislada mediante Docker:

```yaml
# docker-compose.yml
version: '3.8'

services:
  mongodb:
    image: mongo:latest
    container_name: mongodb_local
    restart: always
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password123
    volumes:
      - mongo_data:/data/db

volumes:
  mongo_data:
```

- **Ventajas:** Costo cero en licencias de nube, latencia de red despreciable (`localhost`), total privacidad de datos sin requerir conexion a internet, y control absoluto de los parametros del motor (`mongod.conf`).
- **Desafios:** El desarrollador o DBA debe encargarse manualmente de configurar Replica Sets para alta disponibilidad, ejecutar scripts de respaldos (`mongodump` / `mongorestore`), aplicar parches de seguridad y monitorear el consumo de disco.

---

## 3. MongoDB Atlas (Cloud DBaaS)

**MongoDB Atlas** es la plataforma totalmente administrada en la nube construida y operada por los propios ingenieros de MongoDB Inc. Elimina por completo las tareas de administracion de servidores y aprovisionamiento de infraestructura.

### Niveles de Servicio: Free Tier (M0 Sandbox) vs Clusters Dedicados

Atlas estructura sus servicios en niveles (*tiers*):

| Nivel | Nombre | Uso Principal | Caracteristicas |
| :--- | :--- | :--- | :--- |
| **M0** | **Free Tier (Sandbox)** | Aprendizaje, prototipos y pruebas de concepto. | 512 MB de almacenamiento, CPU compartida, sin costo de por vida, ideal para cursos y proyectos personales. |
| **M2 / M5** | Shared Clusters | Proyectos en etapa temprana / desarrollo en equipo. | 2 GB a 5 GB de disco, bajo costo mensual. |
| **M10+** | Dedicated Clusters | Entornos de produccion empresarial. | Recursos de hardware dedicados (CPU/RAM), escalado automatico, cifrado avanzado, respaldos continuos con recuperacion granular a un segundo especifico (*Point-in-Time Recovery*). |

> [!NOTE]
> Incluso en el nivel gratuito **M0**, Atlas aprovisiona tras bambalinas un **Replica Set de 3 nodos**, garantizando que el desarrollador aprenda a trabajar contra un cluster con alta disponibilidad y tolerancia a fallos desde el primer dia.

### Arquitectura de Seguridad en Atlas

Para que una aplicacion pueda comunicarse con MongoDB Atlas, deben configurarse obligatoriamente dos capas de seguridad independientes:

1. **Control de Acceso de Red (*Network Access / IP Access List*):**
   - Firewall perimetral en la nube. Ninguna peticion entra a menos que su direccion IP publica este registrada en la lista de permitidas (*Whitelist*).
   - En desarrollo, se suele anadir la IP publica actual del hogar/oficina o, para pruebas no sensibles, la regla temporal `0.0.0.0/0` (permite acceso desde cualquier lugar).
   - En produccion, se utilizan tecnicas de emparejamiento de redes privadas (*VPC Peering* o *AWS PrivateLink*).
2. **Autenticacion de Usuarios de Base de Datos (*Database Users*):**
   - Credenciales (usuario y contrasena con hash SCRAM-SHA-256 o certificados X.509) con roles especificos (`readWriteAnyDatabase`, `readOnly`, o permisos acotados a una sola coleccion).

### Despliegue Multi-Cloud

Atlas permite elegir en que proveedor de nube global se desplegaran los nodos fisicos:
- **Amazon Web Services (AWS)**
- **Google Cloud Platform (GCP)**
- **Microsoft Azure**

Incluso permite clusters distribuidos globalmente con nodos repartidos entre distintos continentes o distintos proveedores de nube a la vez para latencia minima.

---

## 4. Anatomia del Connection String (URI de Conexion)

El **URI de conexion** es la cadena estandarizada que utilizan los drivers (Node.js, Python, Java, etc.) y clientes para negociar la comunicacion y autenticarse con el cluster.

### URI Estandar vs DNS Seedlist (`mongodb+srv://`)

#### 1. Formato Estandar (`mongodb://`)
Utilizado comunmente en instalaciones locales o clusters donde se enumeran explicitamente las direcciones IP y puertos de cada nodo del Replica Set:

```text
mongodb://admin:password123@localhost:27017/mi_base_datos?authSource=admin
```

Si fuera un Replica Set local de 3 nodos:
```text
mongodb://user:pass@host1:27017,host2:27017,host3:27017/mi_base_datos?replicaSet=miReplica
```

#### 2. Formato DNS Seedlist (`mongodb+srv://`)
Es el formato moderno que proporciona **MongoDB Atlas**:

```text
mongodb+srv://usuario_app:ClaveSegura2026@cluster0.abcde.mongodb.net/tienda?retryWrites=true&w=majority
```

> [!IMPORTANT]
> **¿Por qué `mongodb+srv://` no lista puertos ni múltiples servidores?**  
> El prefijo `+srv` le indica al driver que consulte los registros **DNS SRV y TXT** de ese dominio en internet. El servidor DNS responde automaticamente con la lista viva de los tres servidores del Replica Set y sus configuraciones. Si un nodo del cluster en Atlas se destruye o cambia de IP, la aplicacion no necesita cambiar su codigo ni su connection string.

### Parametros de Conexion Habituales en la URL

- `retryWrites=true`: Reintenta automaticamente escrituras que fallan por errores transitorios de red.
- `w=majority`: Exige confirmacion de la mayoria de los nodos del Replica Set antes de reportar la escritura como exitosa.
- `authSource=admin`: Especifica en que base de datos residen las credenciales del usuario autenticado (muy comun en instancias locales).

---

## 5. Herramientas de Interaccion del Desarrollador

Independientemente de si la base de datos corre en Local o en Atlas, el desarrollador cuenta con las mismas herramientas universales:

1. **`mongosh` (CLI):** La terminal interactiva nativa. Rapida, sin consumo de interfaz visual y esencial para diagnostico en servidores y automatizaciones.
2. **MongoDB Compass (GUI Oficial):** Aplicacion de escritorio visual que permite explorar documentos, construir consultas visuales, analizar rendimiento de indices (*Explain Plan*), ver distribucion de tipos de datos en la coleccion (*Schema Analysis*) y gestionar agregaciones complejas paso a paso.
3. **Extensiones de IDE:** Plugins oficiales (como la extension de MongoDB para VS Code) que permiten consultar colecciones, reproducir *Playgrounds* de scripts `.mongodb.js` y ver documentos sin salir del editor de codigo.

---

## 6. Matriz Comparativa: Local vs Atlas

| Criterio | MongoDB Community (Local) | MongoDB Atlas (Cloud) |
| :--- | :--- | :--- |
| **Costo de Licencia / Servicio** | Gratuito / Open Source (SSPL). | Nivel M0 gratis; niveles dedicados de pago por hora/recurso. |
| **Instalacion y Puesta en Marcha** | Requiere instalar software en SO o contenedor Docker. | Despliegue en 2 minutos desde navegador web o Terraform. |
| **Acceso a Internet** | **No requerido** (opera 100% offline). | **Obligatorio** (requiere conectividad a la nube). |
| **Alta Disponibilidad (HA)** | Manual (el usuario debe configurar el Replica Set de 3 instancias). | **Nativa y automatica** (incluso en Free Tier M0). |
| **Copias de Seguridad (Backups)** | Responsabilidad del usuario (`cron` + `mongodump`). | Gestionado por la plataforma (snapshots y PITR automatizados). |
| **Seguridad de Red** | Firewall del host / configuracion en `mongod.conf`. | IP Access List / Peering de red integrado en el panel. |
| **Uso Ideal** | Desarrollo rapido desconectado, pruebas unitarias locales, laboratorios. | Produccion empresarial, staging en equipo, prototipos desplegados en Vercel/AWS. |

