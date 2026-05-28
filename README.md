# Hands-On Lab: Select AI con OCI Generative AI en Autonomous Database

> **Audiencia:** Desarrolladores y administradores de base de datos Oracle  
> **Nivel:** Intermedio  
> **Duración estimada:** 3 horas  
> **Plataforma:** Oracle Cloud Infrastructure (OCI) — Región US Midwest (Chicago)

---

## Descripción general

Este laboratorio guía al participante en la configuración completa de **Select AI** dentro de **Oracle Autonomous Database (ADB)**, integrando el servicio de **OCI Generative AI** para traducir preguntas en lenguaje natural a consultas SQL ejecutables.

Al finalizar, el participante será capaz de:

- Crear y gestionar API Keys en OCI Identity & Security
- Descargar y configurar el Wallet de conexión a una base de datos autónoma
- Configurar conexiones en SQL Developer Extension para VS Code
- Integrar el agente Cline con **Oracle Code Assist (OCA)** como proveedor de IA
- Registrar el MCP Server de Oracle SQLcl en Cline
- Crear credenciales y AI Profiles dentro de ADB
- Ejecutar consultas con `SELECT AI` en los modos `showsql`, `runsql` y `narrate`

---

## Prerrequisitos

- Acceso a una tenancy activa en OCI
- Usuario con permisos en **Identity & Security**
- Visual Studio Code instalado con la extensión **Oracle SQL Developer**
- Extensión **Cline** instalada en VS Code
- Base de datos **Autonomous AI Database** aprovisionada (en este lab: `CCSSLABS`)

---

## Arquitectura del laboratorio

```
┌─────────────────────────────────────────────────────┐
│                  VS Code / Cline                    │
│  ┌────────────────┐     ┌──────────────────────┐   │
│  │  Oracle Code   │     │  SQLcl MCP Server    │   │
│  │  Assist (OCA)  │     │  (stdio / Java)      │   │
│  └────────┬───────┘     └──────────┬───────────┘   │
└───────────┼──────────────────────  ┼───────────────┘
            │                        │
            ▼                        ▼
   OCI Generative AI         Oracle ADB (CCSSLABS)
   (LLM: xai.grok-4)         23.26.x — 26ai
```

---

## Parte 1: Pasos previos (Pre-Lab)

### Paso 1: Crear el API Key en OCI

El API Key es un par de claves RSA en formato PEM que se utiliza para firmar las solicitudes hacia los servicios de OCI, incluyendo OCI Generative AI. Este par de claves queda asociado al usuario de IAM que ejecutará las llamadas desde ADB.

**1.1** Inicie sesión en la consola de OCI en [cloud.oracle.com](https://cloud.oracle.com).

**1.2** En el menú principal, navegue a **Identity & Security → Domains**.

![OCI Console - Identity & Security](./images/paso1_identity_security.png)

**1.3** En la lista de dominios, seleccione el dominio **Default** (dominio activo de la tenancy).

![Dominios OCI - Default](./images/paso1_dominios.png)

**1.4** Dentro del dominio, acceda a la pestaña **User management** y localice su usuario (ejemplo: `daniel.a.hernandez@oracle.com`). Haga clic sobre el nombre de usuario.

![User Management](./images/paso1_user_management.png)

**1.5** En el detalle del usuario, seleccione la pestaña **API keys** y haga clic en **Add API key**.

![API Keys - Add](./images/paso1_api_keys.png)

**1.6** En el diálogo que aparece:

- Seleccione la opción **Generate API key pair**
- Haga clic en **Download private key** — guarde este archivo en una ubicación segura; **no se mostrará nuevamente**
- Haga clic en **Download public key**
- Una vez descargadas ambas claves, haga clic en **Add**

![Add API Key - Download](./images/paso1_add_apikey.png)

**1.7** Al finalizar, el sistema presenta la **Configuration file preview** con los valores necesarios para el archivo `~/.oci/config`. Tome nota o copie el contenido para utilizarlo en pasos posteriores:

```ini
[DEFAULT]
user=ocid1.user.oc1..aaaaaaa...
fingerprint=d2:45:dc:2e:39:71:03:25:c5:bc...
tenancy=ocid1.tenancy.oc1..aaaaaaa...
region=us-chicago-1
key_file=<path to your private key file>
```

![Configuration File Preview](./images/paso1_config_preview.png)

> **Importante:** Actualice el parámetro `key_file` con la ruta absoluta donde guardó la llave privada descargada.

---

### Paso 2: Descargar el Wallet de la Base de Datos Autónoma

El Wallet contiene los certificados y cadenas de conexión necesarios para establecer una conexión mTLS segura hacia la base de datos autónoma.

**2.1** En el menú principal de OCI, navegue a **Oracle AI Database → Autonomous AI Database**.

**2.2** Asegúrese de que el filtro de compartimento apunte a `database`. Localice la instancia **CCSSLABS** y haga clic sobre su nombre.

![Autonomous AI Databases - CCSSLABS](./images/paso2_adb_list.png)

**2.3** En la página de detalle de la base de datos, haga clic en el botón **Database connection**.

**2.4** En la sección **Download client credentials (Wallet)**, seleccione el tipo de wallet **Instance wallet** y haga clic en **Download wallet**.

![Database Connection - Download Wallet](./images/paso2_db_connection.png)

**2.5** El sistema solicitará una contraseña para proteger el wallet:

```
EjemploPass2026$$
```

Ingrese la contraseña en ambos campos y haga clic en **Download**.

![Download Wallet - Password](./images/paso2_download_wallet.png)

**2.6** El archivo descargado tendrá el nombre `Wallet_CCSSLABS.zip`. Extráigalo en una carpeta de fácil acceso (por ejemplo, `C:\Users\opc\wallet\CCSSLABS\`). El wallet contiene, entre otros archivos:

| Archivo | Descripción |
|---|---|
| `tnsnames.ora` | Cadenas de conexión TNS disponibles |
| `sqlnet.ora` | Configuración de red Oracle |
| `ewallet.p12` | Certificado del wallet en formato PKCS#12 |
| `keystore.jks` | Keystore Java |
| `truststore.jks` | Truststore Java |

---

### Paso 3: Configurar la conexión en SQL Developer (VS Code)

En este paso se registra la conexión a la base de datos autónoma dentro de la extensión **Oracle SQL Developer para VS Code**, haciendo uso del wallet descargado.

**3.1** Abra **Visual Studio Code** y acceda al panel de la extensión **SQL Developer** desde la barra lateral izquierda.

**3.2** Haga clic en **Create Connection**.

![SQL Developer - Create Connection](./images/paso3_sqldeveloper.png)

**3.3** Complete el formulario de conexión con los siguientes valores:

| Campo | Valor |
|---|---|
| Connection Name | `CCSSLABS` |
| Authentication Type | Default |
| Username | `TUUSUARIO` |
| Password | `CCSSlabs2026$$` |
| Connection Type | **Cloud Wallet** |
| Configuration File | Seleccionar el archivo `Wallet_CCSSLABS.zip` |
| Service | `CCSSLABS_HIGH` |

![Create Connection Form](./images/paso3_create_connection.png)

**3.4** Haga clic en **Test**. Si la conexión es exitosa, aparecerá el mensaje:

```
Test passed for connection: CCSSLABS
```

![Test Connection - Passed](./images/paso3_test_passed.png)

**3.5** Haga clic en **Connect** para guardar la conexión y conectarse en un solo paso, o en **Save** si únicamente desea registrarla sin conectarse de inmediato.

---

### Paso 4: Configurar el agente Cline con Oracle Code Assist (OCA)

Oracle Code Assist es el proveedor de IA de Oracle integrado en Cline. Su configuración permite que el agente utilice los modelos LLM de Oracle sin necesidad de gestionar una API Key externa.

**4.1** En VS Code, haga clic en el ícono de **Cline** en la barra lateral.

**4.2** En la pantalla de bienvenida, seleccione la opción **Bring my own API key** y haga clic en **Continue**.

![Cline - How will you use Cline](./images/paso4_cline_setup.png)

**4.3** En el campo **API Provider**, seleccione **Oracle Code Assist** de la lista desplegable.

**4.4** Haga clic en **Sign in with Oracle Code Assist**.

![Cline - Oracle Code Assist Provider](./images/paso4_oca_provider.png)

**4.5** VS Code mostrará un diálogo solicitando autorización para abrir el sitio de autenticación de Oracle. Haga clic en **Open**.

> **Nota:** En este paso se utilizará el usuario del instructor para completar la autenticación.

**4.6** Complete el flujo de autenticación en el navegador con las credenciales proporcionadas y regrese a VS Code una vez finalizado.

---

### Paso 5: Configurar el MCP Server de Oracle SQLcl en Cline

El **MCP Server** (Model Context Protocol) permite que Cline se comunique directamente con Oracle SQLcl, habilitando al agente para ejecutar comandos SQL y administrar la base de datos en lenguaje natural.

**5.1** En el panel de Cline, haga clic en el ícono de configuración (⚙) y seleccione **MCP Servers**.

**5.2** En la pantalla de MCP Servers, seleccione la pestaña **Configure** y haga clic en **Configure MCP Servers**.

![Cline - MCP Servers](./images/paso5_mcp_servers.png)

**5.3** Se abrirá el archivo `cline_mcp_settings.json`. Reemplace su contenido con la siguiente configuración, ajustando las rutas de acuerdo con la versión instalada de la extensión SQL Developer en su equipo:

```json
{
  "mcpServers": {
    "SQLcl - SQL Developer": {
      "timeout": 60,
      "type": "stdio",
      "command": "C:\\Users\\opc\\.vscode\\extensions\\oracle.sql-developer-26.1.2-win32-x64\\dbtools\\jdk\\bin\\java.exe",
      "args": [
        "-Djava.awt.headless=true",
        "-Djava.net.useSystemProxies=true",
        "-Duser.language=en",
        "-p",
        "C:\\Users\\opc\\.vscode\\extensions\\oracle.sql-developer-26.1.2-win32-x64\\dbtools\\launch\\;C:\\Users\\opc\\.vscode\\extensions\\oracle.sql-developer-26.1.2-win32-x64\\dbtools\\sqlcl\\launch\\",
        "--add-modules",
        "ALL-DEFAULT",
        "-m",
        "com.oracle.dbtools.launch",
        "sql",
        "-mcp"
      ],
      "env": {}
    }
  }
}
```

> **Nota:** Verifique que la versión en la ruta (`26.1.2`) coincida con la versión de la extensión instalada en su equipo. Puede consultarla en **Extensions → Oracle SQL Developer → Details**.

**5.4** Guarde el archivo. Cline detectará automáticamente el servidor MCP registrado.

---

### Paso 6: Verificar la integración Cline + MCP + ADB

Antes de iniciar los laboratorios, se debe comprobar que el agente puede conectarse a la base de datos y responder correctamente.

**6.1** En el chat de Cline, escriba el siguiente mensaje de prueba:

```
Hola. Puedes conectarte a la Base de Datos CCSSLabs y darme la versión
y la fecha y la hora de la DB en UTC-6.
```

**6.2** El agente ejecutará una consulta a través del MCP Server y devolverá un resultado similar al siguiente:

![Cline - Prueba MCP Exitosa](./images/paso6_prueba_mcp.png)

```
Conexión a CCSSLABS realizada correctamente.
Resultados consultados:

| Dato                        | Valor                       |
|-----------------------------|-----------------------------|
| Versión de la base de datos | 23.26.2.2.0                 |
| Fecha y hora de la DB UTC-6 | 2026-05-27 19:48:41 -06:00  |
```

Si obtiene este resultado, el entorno está correctamente configurado y puede continuar con los laboratorios.

---

## Parte 2: Laboratorios

---

## Lab 1: Administración Agéntica en Lenguaje Natural

**Objetivo:** Demostrar cómo un agente LLM puede ejecutar tareas de administración de base de datos Oracle utilizando únicamente lenguaje natural, a través de la integración entre Cline y el MCP Server de SQLcl.

### Ejercicio 1.1 — Explorar el esquema actual

En el chat de Cline, ejecute el siguiente prompt:

```
Muéstrame todas las tablas y relaciones que existen en el esquema del usuario actual.
```

El agente consultará las vistas del diccionario de datos (`USER_TABLES`, `USER_CONSTRAINTS`) y presentará un resumen de la estructura del esquema activo.

---

### Ejercicio 1.2 — Crear una nueva tabla con auditoría

```
Crea una tabla llamada PATRONO_DIRECCION con: DIRECCION_ID como llave primaria,
PATRONO_ID como llave foránea referenciando la tabla de patronos, una columna
DIRECCION para la dirección, y los campos de auditoría estándar de creación y
modificación (usuario y fecha). Usa tipos de datos apropiados.
```

El agente generará y ejecutará el DDL correspondiente, incluyendo la restricción de llave foránea y los campos de auditoría (`CREATED_BY`, `CREATED_DATE`, `MODIFIED_BY`, `MODIFIED_DATE`).

---

### Ejercicio 1.3 — Poblar la tabla con datos de prueba

```
Inserta al menos 20 registros aleatorios en PATRONO_DIRECCION. Asegúrate de que
todos los valores de PATRONO_ID existan en la tabla de patronos para cumplir la
integridad referencial.
```

El agente consultará previamente los `PATRONO_ID` existentes y generará los `INSERT` respetando las restricciones de integridad referencial.

---

### Ejercicio 1.4 — Análisis de impacto de cambio de tipo de dato

```
Genera un reporte con todas las tablas que referencian el ID del patrono, e indica
el impacto que tendría cambiar el tipo de dato de ese campo — incluyendo efectos
sobre llaves foráneas, índices y objetos dependientes.
```

El agente consultará `USER_CONSTRAINTS`, `USER_IND_COLUMNS` y `USER_DEPENDENCIES` para construir un análisis de impacto completo.

---

### Ejercicio 1.5 — Generar y aplicar script de cambios

```
Genera un script SQL con todos los cambios del análisis anterior y guárdalo en la
carpeta Mis Documentos. Luego aplica ese script en la base de datos.
```

> **Nota:** Este prompt está intencionalmente dividido en dos oraciones para que el agente las trate como acciones secuenciales: primero genera y persiste el archivo, y luego lo ejecuta.

---

## Lab 2: Select AI con OCI Generative AI

**Objetivo:** Configurar y utilizar **Select AI** dentro de Oracle Autonomous Database para interpretar preguntas en lenguaje natural y traducirlas a consultas SQL mediante OCI Generative AI.

---

### Paso 2.0: Preparación del entorno — Ejecutado por el instructor

> ⚠️ **Este paso es ejecutado por el instructor con el usuario `ADMIN` antes de iniciar el laboratorio. Los participantes no deben ejecutar este bloque.**

Para que un usuario de base de datos pueda utilizar Select AI, el administrador debe otorgarle los privilegios necesarios y configurar el acceso de red hacia los endpoints de OCI. Los siguientes pasos se ejecutan una única vez por usuario, conectado con `ADMIN`:

**Otorgar privilegios de ejecución sobre los paquetes de AI:**

```sql
-- Permite al usuario invocar Select AI y DBMS_CLOUD_AI
GRANT EXECUTE ON DBMS_CLOUD_AI TO TUUSUARIO;

-- Permite al usuario utilizar Select AI con RAG (pipelines de documentos)
-- Si el usuario ya posee el rol DWROLE, este privilegio ya está incluido
GRANT EXECUTE ON DBMS_CLOUD_PIPELINE TO TUUSUARIO;
```

**Configurar el acceso de red hacia OCI Generative AI:**

```sql
-- Habilita al usuario para conectarse al endpoint de OCI desde ADB
BEGIN
  DBMS_NETWORK_ACL_ADMIN.APPEND_HOST_ACE(
    host => '*.oci.oraclecloud.com',
    ace  => xs$ace_type(
      privilege_list => xs$name_list('connect', 'resolve'),
      principal_name => 'TUUSUARIO',
      principal_type => xs_acl.ptype_db
    )
  );
END;
/
```

**Verificar que el ACL fue registrado correctamente:**

```sql
SELECT HOST, LOWER_PORT, UPPER_PORT,
       ACE_ORDER, PRINCIPAL, PRINCIPAL_TYPE,
       GRANT_TYPE, INVERTED_PRINCIPAL, PRIVILEGE,
       START_DATE, END_DATE
  FROM (
    SELECT ACES.*,
           DBMS_NETWORK_ACL_UTILITY.CONTAINS_HOST(
             '*.oci.oraclecloud.com', HOST
           ) PRECEDENCE
      FROM DBA_HOST_ACES ACES
  )
 WHERE PRECEDENCE IS NOT NULL
 ORDER BY PRECEDENCE DESC,
          LOWER_PORT  NULLS LAST,
          UPPER_PORT  NULLS LAST,
          ACE_ORDER;
```

El resultado debe mostrar una fila con el `PRINCIPAL` correspondiente al usuario, el `HOST` `*.oci.oraclecloud.com` y los privilegios `connect` y `resolve` con `GRANT_TYPE = GRANT`.

---

### Paso 2.1: Crear la credencial en ADB

La credencial almacena de forma segura los datos de autenticación de OCI dentro de la base de datos, permitiendo que `DBMS_CLOUD` y `DBMS_CLOUD_AI` se conecten a los servicios externos.

Navegue a **ADB → Development → SQL** y ejecute el siguiente bloque PL/SQL, sustituyendo los valores indicados con su información de OCI:

```sql
BEGIN
  DBMS_CLOUD.CREATE_CREDENTIAL(
    credential_name => 'GENAI_CRED_<TUUSUARIO>',
    user_ocid       => 'ocid1.user.oc1..aaaaaaaaxxxxxxxxxxxxxxxxxxxxxxxxxxxxx',
    tenancy_ocid    => 'ocid1.tenancy.oc1..aaaaaaaayyyyyyyyyyyyyyyyyyyyyyyy',
    private_key     => '-----BEGIN PRIVATE KEY-----
<PEGAR_AQUI_TU_LLAVE_PRIVADA>
-----END PRIVATE KEY-----',
    fingerprint     => 'aa:bb:cc:dd:ee:ff:11:22:33:44:55:66:77:88:99:00'
  );
END;
/
```

| Parámetro | Descripción |
|---|---|
| `credential_name` | Nombre lógico de la credencial dentro del esquema ADB |
| `user_ocid` | OCID del usuario de IAM que posee el API Key |
| `tenancy_ocid` | OCID de la tenancy OCI |
| `private_key` | Contenido completo de la llave privada descargada en el Paso 1 |
| `fingerprint` | Huella digital del API Key, visible en la sección **API keys** del usuario |

---

### Paso 2.2: Crear el AI Profile

El **AI Profile** es la configuración que conecta Select AI con el proveedor de inteligencia artificial. Define el modelo a utilizar, las credenciales de acceso, la región del servicio y los objetos del esquema que podrán ser consultados en lenguaje natural.

```sql
BEGIN
  DBMS_CLOUD_AI.CREATE_PROFILE(
    profile_name => 'OCIGenAI_PROF_<TUUSUARIO>',
    attributes   => '{
      "provider":         "oci",
      "credential_name":  "GENAI_CRED_<TUUSUARIO>",
      "region":           "us-chicago-1",
      "model":            "xai.grok-4",
      "object_list": [
        {"owner": "<TUUSUARIO>", "name": "APORTACIONES"},
        {"owner": "<TUUSUARIO>", "name": "EMPLEADOS"},
        {"owner": "<TUUSUARIO>", "name": "PATRONOS"},
        {"owner": "<TUUSUARIO>", "name": "PERIODOS_APORTACION"},
        {"owner": "<TUUSUARIO>", "name": "RELACIONES_LABORALES"},
        {"owner": "<TUUSUARIO>", "name": "TIPOS_APORTACION"}
      ]
    }'
  );
END;
/
```

| Atributo | Descripción |
|---|---|
| `provider` | Proveedor de AI. En este laboratorio se utiliza `oci` |
| `credential_name` | Credencial creada en el paso anterior |
| `region` | Región donde se invoca el servicio de OCI Generative AI |
| `model` | Modelo LLM que interpretará las preguntas en lenguaje natural |
| `object_list` | Tablas del esquema que Select AI tiene autorización para consultar |

> **Nota:** Este paso únicamente registra la configuración. No ejecuta consultas ni valida la conectividad con el servicio de AI en este momento.

---

### Paso 2.3: Validar el AI Profile

Verifique que el profile fue creado correctamente ejecutando las siguientes consultas:

```sql
-- Validación 1: Confirmar que el profile existe en el esquema
SELECT * FROM USER_CLOUD_AI_PROFILES;

-- Validación 2: Revisar los atributos configurados
SELECT * FROM USER_CLOUD_AI_PROFILE_ATTRIBUTES;
```

El resultado debe mostrar el profile `OCIGenAI_<TUUSUARIO>` con todos los atributos configurados en el paso anterior (proveedor, modelo, región, credencial y lista de objetos).

> Si el profile no aparece en los resultados, verifique que el bloque PL/SQL del paso anterior se ejecutó sin errores y que la sesión activa corresponde al esquema correcto.

---

### Paso 2.4: Ejecutar consultas con Select AI

Select AI soporta tres modos de operación para interpretar preguntas en lenguaje natural:

| Modo | Comportamiento |
|---|---|
| `showsql` | Genera el SQL correspondiente a la pregunta, pero **no lo ejecuta** |
| `runsql` | Genera el SQL y lo **ejecuta** contra las tablas autorizadas |
| `narrate` | Ejecuta la consulta y devuelve el resultado en **lenguaje natural** |

#### Sintaxis para SQL Developer Desktop / VS Code

```sql
-- Ver el SQL generado (sin ejecutar)
SELECT AI showsql 'tu pregunta aquí';

-- Ejecutar la consulta generada
SELECT AI 'tu pregunta aquí';

-- Obtener respuesta en lenguaje natural
SELECT AI narrate 'tu pregunta aquí';
```

#### Sintaxis para SQL Developer Web (ADB Console)

```sql
-- Mostrar el SQL generado
SELECT DBMS_CLOUD_AI.GENERATE(
  prompt       => 'TU PREGUNTA',
  action       => 'showsql',
  profile_name => 'OCIGenAI_TUUSUARIO'
) FROM dual;

-- Ejecutar la consulta generada
SELECT DBMS_CLOUD_AI.GENERATE(
  prompt       => 'TU PREGUNTA',
  action       => 'runsql',
  profile_name => 'OCIGenAI_TUUSUARIO'
) FROM dual;

-- Respuesta en lenguaje natural
SELECT DBMS_CLOUD_AI.GENERATE(
  prompt       => 'TU PREGUNTA',
  action       => 'narrate',
  profile_name => 'OCIGenAI_TUUSUARIO'
) FROM dual;
```

---

### Paso 2.5: Consultas de prueba del laboratorio

Utilice las siguientes preguntas de negocio para validar el funcionamiento de Select AI. Pruebe cada una con los tres modos disponibles (`showsql`, `runsql`, `narrate`):

**Consulta 1 — Listado de empleados activos**
```
Mostrar el listado de empleados activos con su identificación, nombre completo,
correo, teléfono y fecha de creación del registro.
```

**Consulta 2 — Relaciones laborales activas**
```
Consultar las relaciones laborales activas, mostrando el nombre completo del empleado,
el patrono asociado, el número de empleado, la fecha de inicio y el salario base.
```

**Consulta 3 — Total de aportaciones por patrono y periodo**
```
Obtener el total de aportaciones registradas por patrono y por periodo, indicando
el año, mes, nombre del patrono y monto total aportado.
```

**Consulta 4 — Aportaciones de un empleado específico**
```
Listar las aportaciones de un empleado específico, mostrando el periodo, tipo de
aportación, salario reportado, porcentaje aplicado, monto de la aportación,
origen y estado.
```

**Consulta 5 — Patronos sin aportaciones en un periodo**
```
Identificar los patronos que tienen empleados con relaciones laborales activas,
pero que no poseen aportaciones registradas en un periodo determinado.
```

---

## Resumen

En este laboratorio se completaron los siguientes objetivos:

- Generación de API Keys en OCI y descarga del Wallet de ADB
- Configuración de conexiones en SQL Developer Extension para VS Code
- Integración de Cline con Oracle Code Assist como proveedor LLM
- Registro del MCP Server de SQLcl para administración agéntica en lenguaje natural
- Creación de credenciales OCI y AI Profiles dentro de Autonomous Database
- Ejecución de consultas en lenguaje natural utilizando Select AI con los modos `showsql`, `runsql` y `narrate`

---

## Referencias

- [Oracle Autonomous Database — Select AI](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/autonomous-select-ai.html)
- [DBMS_CLOUD_AI — Documentación oficial](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/dbms-cloud-ai-package.html)
- [OCI Generative AI Service](https://docs.oracle.com/en-us/iaas/Content/generative-ai/overview.htm)
- [Oracle SQL Developer Extension for VS Code](https://marketplace.visualstudio.com/items?itemName=Oracle.sql-developer)
- [Cline — Model Context Protocol](https://github.com/cline/cline)

---

*Laboratorio preparado para CCSS — Oracle Cloud Infrastructure · Mayo 2026*
