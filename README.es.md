# PAIO8 — Guía de instalación

[English](README.md) | **Español**

**OpenEMR 8.0.0.2 + Jitsi Meet + CIE-11 (OMS) + Módulos PAHO**

Versión 2.0 · Junio 2026 · Instalación limpia desde cero · Servidor Ubuntu 24.x · Docker

---

## Tabla de contenido

1. [Introducción y arquitectura](#1-introducción-y-arquitectura)
2. [Prerrequisitos](#2-prerrequisitos)
3. [Estructura del proyecto](#3-estructura-del-proyecto)
4. [Certificados SSL (Let's Encrypt)](#4-certificados-ssl-lets-encrypt)
5. [Archivo de configuración `.env`](#5-archivo-de-configuración-env)
6. [Archivo `docker-compose.yml`](#6-archivo-docker-composeyml)
7. [Iniciar los contenedores](#7-iniciar-los-contenedores)
8. [Clonar los módulos PAHO](#8-clonar-los-módulos-paho)
9. [Registrar, instalar y habilitar los módulos](#9-registrar-instalar-y-habilitar-los-módulos)
10. [Configurar cada módulo](#10-configurar-cada-módulo)
11. [Configuración inicial de OpenEMR](#11-configuración-inicial-de-openemr)
12. [Flujo de Tele Visita](#12-flujo-de-tele-visita)
13. [Mantenimiento y actualización de módulos](#13-mantenimiento-y-actualización-de-módulos)
14. [Comandos de referencia rápida](#14-comandos-de-referencia-rápida)

---

## 1. Introducción y arquitectura

Esta guía describe la instalación completa de PAIO8 desde cero sobre un servidor Linux con Docker. La plataforma se compone de software de terceros que se descarga automáticamente como imágenes de contenedor, más cuatro módulos propios de PAHO que se clonan desde GitHub.

**Importante sobre el origen de cada componente.** Usted no copia ni compila manualmente OpenEMR, Jitsi ni la API de CIE-11. Todos se descargan como imágenes oficiales y esa descarga la realiza Docker Compose la primera vez que levanta los servicios (paso 7). Solo los cuatro módulos PAHO se obtienen con `git clone`.

| Componente | Origen | Cómo se obtiene |
| --- | --- | --- |
| OpenEMR 8.0.0.2 | Repositorio oficial de OpenEMR en GitHub, publicado como imagen `openemr/openemr:8.0.0.2` | Docker Compose descarga la imagen al ejecutar `docker compose up` |
| Base de datos (MariaDB 11.8) | Imagen oficial `mariadb:11.8` | Docker Compose |
| Jitsi Meet (web, prosody, jicofo, jvb) | Imágenes oficiales de Jitsi `jitsi/web`, `jitsi/prosody`, `jitsi/jicofo`, `jitsi/jvb` (etiqueta stable) | Docker Compose descarga las imágenes; Jitsi corre dentro de los contenedores |
| API CIE-11 (OMS) | Imagen oficial de la OMS `whoicd/icd-api` | Docker Compose; el contenido CIE-11 lo provee directamente la OMS dentro de la imagen |
| Módulos PAHO (4) | Repositorios en `github.com/diazjuan` | Se clonan con `git clone` dentro de `custom_modules/` (paso 8) |

Los cuatro módulos PAHO incluidos en esta instalación son:

- **`paho_jitsi_televisit_module`** — Tele Visita con Jitsi Meet embebido y autenticación por JWT.
- **`paho_openemr_icd_11`** — Búsqueda y registro de diagnósticos CIE-11 (OMS) en el flujo del paciente. En la interfaz de OpenEMR el módulo aparece como "PAHO ICD-11 Integration" (ICD-11 es la sigla en inglés de CIE-11).
- **`paho_openemr_translations`** — Traducciones al español de OpenEMR y de los módulos PAHO.
- **`paio_login_theme`** — Marca visual (logo y colores OPS) en la pantalla de inicio de sesión.

---

## 2. Prerrequisitos

Antes de iniciar, asegúrese de contar con lo siguiente:

- Servidor Linux (Ubuntu 24.x) con al menos **25 GB** de espacio libre en disco.
- **Docker** y **Docker Compose** instalados en el servidor.
- **git** instalado (para clonar los módulos PAHO).
- Acceso **SSH** al servidor con un usuario con permisos `sudo`.
- Un **dominio** (por ejemplo `emr.sudominio.com`) cuyo registro DNS apunte a la IP pública del servidor.
- Puertos abiertos en el firewall: **80/TCP** (emisión y renovación del certificado), **443/TCP** (OpenEMR), **8444/TCP** (Jitsi web) y **10000/UDP** (medios de Jitsi/JVB).

Verifique que Docker y Docker Compose estén disponibles:

```bash
docker --version
docker compose version
```

> **Nota:** Esta guía asume un único dominio para OpenEMR (puerto 443) y Jitsi (puerto 8444). No se utiliza proxy inverso: cada contenedor expone su propio puerto.

---

## 3. Estructura del proyecto

Conéctese al servidor por SSH y cree la carpeta de trabajo:

```bash
mkdir -p ~/paio8/custom_modules
cd ~/paio8
```

Al terminar la instalación, la estructura será la siguiente:

```text
~/paio8/
|- .env                         <- configuración (paso 5)
|- docker-compose.yml           <- definición de servicios (paso 6)
|- jitsi-config/                <- config de Jitsi (la generan los contenedores)
|- custom_modules/              <- módulos PAHO (se clonan en el paso 8)
|   |- paho_jitsi_televisit_module/
|   |- paho_openemr_icd_11/
|   |- paho_openemr_translations/
|   `- paio_login_theme/
```

> **Nota:** La carpeta `jitsi-config/` la generan automáticamente los contenedores de Jitsi la primera vez que arrancan; no necesita crearla ni editarla a mano.

---

## 4. Certificados SSL (Let's Encrypt)

Emita el certificado antes de levantar los contenedores, mientras el puerto 80 está libre. Un único certificado para el dominio cubre tanto OpenEMR (443) como Jitsi (8444), porque ambos usan el mismo nombre de host.

```bash
sudo apt update && sudo apt install -y certbot
sudo certbot certonly --standalone -d emr.sudominio.com
```

El certificado y la clave quedan en `/etc/letsencrypt/live/emr.sudominio.com/`. El archivo `docker-compose.yml` del paso 6 monta esos archivos dentro de los contenedores de OpenEMR y de Jitsi en modo solo lectura.

> **Nota:** Para la renovación automática (`certbot renew`) el puerto 80 debe estar libre. Detenga brevemente OpenEMR durante la renovación, o use un hook de despliegue que reinicie los contenedores tras renovar.

---

## 5. Archivo de configuración `.env`

Cree el archivo `.env` en `~/paio8/`:

```bash
nano ~/paio8/.env
```

Pegue el siguiente contenido y reemplace cada valor según su entorno. Sustituya `emr.sudominio.com` por su dominio real (también en el paso 6):

```ini
# =============================================================
# PAIO8 - OpenEMR 8.0.0.2 + Jitsi + CIE-11
# Archivo de configuracion (.env)
# =============================================================

# Dominio publico del servidor (debe apuntar por DNS a la IP del servidor)
SERVER_HOST=emr.sudominio.com

# Puertos OpenEMR (el 80 redirige automaticamente al 443)
OPENEMR_PORT=80
OPENEMR_HTTPS_PORT=443

# Puertos Jitsi
JITSI_PORT=8444
JITSI_UDP_PORT=10000

# Base de datos (MariaDB)
MYSQL_ROOT_PASSWORD=CambieEstaClaveSegura1
MYSQL_PASSWORD=CambieEstaClaveSegura2

# Administrador inicial de OpenEMR
OE_USER=admin
OE_PASS=CambieEstaClaveSegura3

# Secretos internos de Jitsi (XMPP)
JICOFO_COMPONENT_SECRET=CambieEsteSecreto1
JICOFO_AUTH_PASSWORD=CambieEsteSecreto2
JVB_AUTH_PASSWORD=CambieEsteSecreto3

# Autenticacion JWT de Jitsi (modulo Tele Visita)
# Genere el secreto con:  openssl rand -hex 32
JWT_APP_ID=paho-jitsi-televisit
JWT_APP_SECRET=PEGUE_AQUI_EL_RESULTADO_DE_openssl_rand_hex_32
```

Genere el secreto JWT con el siguiente comando y péguelo en `JWT_APP_SECRET`:

```bash
openssl rand -hex 32
```

Guarde y salga de nano: `Ctrl+O` → `Enter` → `Ctrl+X`.

> **Nota:** El mismo `JWT_APP_SECRET` y `JWT_APP_ID` se inyectan en OpenEMR (emisor del token) y en los contenedores de Jitsi (verificadores). Deben coincidir exactamente o la sala de video rechazará la conexión. Use un secreto distinto por entorno.

---

## 6. Archivo `docker-compose.yml`

Cree el archivo `docker-compose.yml` en `~/paio8/`:

```bash
nano ~/paio8/docker-compose.yml
```

Define los siguientes 7 servicios. Recuerde que estas imágenes se descargan solas: no hay que copiar ni compilar OpenEMR, Jitsi ni la API de CIE-11.

| Servicio | Imagen | Puerto host |
| --- | --- | --- |
| mysql | `mariadb:11.8` | interno |
| openemr | `openemr/openemr:8.0.0.2` | 80, 443 |
| jitsi-web | `jitsi/web:stable` | 8444 |
| prosody | `jitsi/prosody:stable` | interno |
| jicofo | `jitsi/jicofo:stable` | interno |
| jvb | `jitsi/jvb:stable` | 10000/UDP |
| icd11-api | `whoicd/icd-api:latest` | interno |

Pegue el contenido completo (reemplace `emr.sudominio.com` por su dominio en las cuatro rutas de certificado):

```yaml
services:
  mysql:
    image: mariadb:11.8
    container_name: paio_mysql
    restart: unless-stopped
    command:
      - mariadbd
      - --character-set-server=utf8mb4
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "/usr/local/bin/healthcheck.sh", "--su-mysql", "--connect", "--innodb_initialized"]
      start_period: 1m
      start_interval: 10s
      interval: 1m
      timeout: 5s
      retries: 3
    networks:
      - paio_network

  openemr:
    image: openemr/openemr:8.0.0.2
    container_name: paio_openemr
    restart: unless-stopped
    depends_on:
      mysql:
        condition: service_healthy
    ports:
      - "${OPENEMR_PORT}:80"
      - "${OPENEMR_HTTPS_PORT}:443"
    environment:
      MYSQL_HOST: mysql
      MYSQL_ROOT_PASS: ${MYSQL_ROOT_PASSWORD}
      MYSQL_USER: openemr
      MYSQL_PASS: ${MYSQL_PASSWORD}
      OE_USER: ${OE_USER}
      OE_PASS: ${OE_PASS}
      # Material de firma JWT de Jitsi (lado emisor). El modulo Tele Visita
      # lo lee primero del entorno; los globales de la BD son el respaldo.
      JWT_APP_ID: ${JWT_APP_ID}
      JWT_APP_SECRET: ${JWT_APP_SECRET}
    volumes:
      - openemr_logs:/var/log
      - openemr_sites:/var/www/localhost/htdocs/openemr/sites
      # Modulos PAHO (se clonan en el paso 8)
      - ./custom_modules:/var/www/localhost/htdocs/openemr/interface/modules/custom_modules
      # Certificados TLS de produccion (Let's Encrypt)
      - /etc/letsencrypt/live/emr.sudominio.com/fullchain.pem:/etc/ssl/certs/webserver.cert.pem:ro
      - /etc/letsencrypt/live/emr.sudominio.com/privkey.pem:/etc/ssl/private/webserver.key.pem:ro
    healthcheck:
      test: ["CMD", "/usr/bin/curl", "--fail", "--insecure", "--location", "--show-error", "--silent", "https://localhost/meta/health/readyz"]
      start_period: 3m
      start_interval: 10s
      interval: 1m
      timeout: 5s
      retries: 3
    networks:
      - paio_network

  jitsi-web:
    image: jitsi/web:stable
    container_name: paio_jitsi_web
    restart: unless-stopped
    ports:
      - "${JITSI_PORT}:443"
    environment:
      TZ: America/Chicago
      PUBLIC_URL: https://${SERVER_HOST}:${JITSI_PORT}
      DISABLE_HTTPS: "0"
      XMPP_SERVER: prosody
      XMPP_DOMAIN: meet.jitsi
      XMPP_AUTH_DOMAIN: auth.meet.jitsi
      XMPP_GUEST_DOMAIN: guest.meet.jitsi
      XMPP_MUC_DOMAIN: muc.meet.jitsi
      XMPP_INTERNAL_MUC_DOMAIN: internal-muc.meet.jitsi
      XMPP_BOSH_URL_BASE: http://prosody:5280
      # Solo los JWT validos pueden entrar a la sala.
      ENABLE_AUTH: "1"
      ENABLE_GUESTS: "0"
      AUTH_TYPE: jwt
      JWT_APP_ID: ${JWT_APP_ID}
      JWT_APP_SECRET: ${JWT_APP_SECRET}
      ENABLE_XMPP_WEBSOCKET: "1"
      ENABLE_PREJOIN_PAGE: "1"
      ENABLE_WELCOME_PAGE: "1"
      ENABLE_P2P: "1"
      JVB_PORT: "10000"
      JVB_TCP_HARVESTER_DISABLED: "true"
    volumes:
      - ./jitsi-config/web:/config
      - ./jitsi-config/web/crontabs:/var/spool/cron/crontabs
      - ./jitsi-config/transcripts:/usr/share/jitsi-meet/transcripts
      # El contenedor Jitsi reutiliza el mismo certificado del dominio
      - /etc/letsencrypt/live/emr.sudominio.com/fullchain.pem:/config/keys/cert.crt:ro
      - /etc/letsencrypt/live/emr.sudominio.com/privkey.pem:/config/keys/cert.key:ro
    depends_on:
      - prosody
      - jvb
    networks:
      - paio_network

  prosody:
    image: jitsi/prosody:stable
    container_name: paio_jitsi_prosody
    restart: unless-stopped
    expose:
      - "5222"
      - "5280"
      - "5347"
    environment:
      TZ: America/Chicago
      PUBLIC_URL: https://${SERVER_HOST}:${JITSI_PORT}
      XMPP_DOMAIN: meet.jitsi
      XMPP_AUTH_DOMAIN: auth.meet.jitsi
      XMPP_GUEST_DOMAIN: guest.meet.jitsi
      XMPP_MUC_DOMAIN: muc.meet.jitsi
      XMPP_INTERNAL_MUC_DOMAIN: internal-muc.meet.jitsi
      XMPP_RECORDER_DOMAIN: recorder.meet.jitsi
      AUTH_TYPE: jwt
      ENABLE_AUTH: "1"
      ENABLE_GUESTS: "0"
      ENABLE_LOBBY: "0"
      JWT_APP_ID: ${JWT_APP_ID}
      JWT_APP_SECRET: ${JWT_APP_SECRET}
      ENABLE_XMPP_WEBSOCKET: "1"
      JICOFO_AUTH_USER: focus
      JICOFO_AUTH_PASSWORD: ${JICOFO_AUTH_PASSWORD}
      JICOFO_COMPONENT_SECRET: ${JICOFO_COMPONENT_SECRET}
      JVB_AUTH_USER: jvb
      JVB_AUTH_PASSWORD: ${JVB_AUTH_PASSWORD}
      LOG_LEVEL: info
    volumes:
      - ./jitsi-config/prosody/config:/config
      - ./jitsi-config/prosody/prosody-plugins-custom:/prosody-plugins-custom
    networks:
      paio_network:
        aliases:
          - xmpp.meet.jitsi

  jicofo:
    image: jitsi/jicofo:stable
    container_name: paio_jitsi_jicofo
    restart: unless-stopped
    environment:
      TZ: America/Chicago
      XMPP_SERVER: prosody
      XMPP_DOMAIN: meet.jitsi
      XMPP_AUTH_DOMAIN: auth.meet.jitsi
      XMPP_INTERNAL_MUC_DOMAIN: internal-muc.meet.jitsi
      XMPP_MUC_DOMAIN: muc.meet.jitsi
      JICOFO_AUTH_USER: focus
      JICOFO_AUTH_PASSWORD: ${JICOFO_AUTH_PASSWORD}
      JICOFO_COMPONENT_SECRET: ${JICOFO_COMPONENT_SECRET}
      JVB_BREWERY_MUC: jvbbrewery
      ENABLE_AUTH: "0"
    volumes:
      - ./jitsi-config/jicofo:/config
    depends_on:
      - prosody
    networks:
      - paio_network

  jvb:
    image: jitsi/jvb:stable
    container_name: paio_jitsi_jvb
    restart: unless-stopped
    ports:
      - "${JITSI_UDP_PORT}:10000/udp"
    environment:
      TZ: America/Chicago
      PUBLIC_URL: https://${SERVER_HOST}:${JITSI_PORT}
      XMPP_SERVER: prosody
      XMPP_DOMAIN: meet.jitsi
      XMPP_AUTH_DOMAIN: auth.meet.jitsi
      XMPP_INTERNAL_MUC_DOMAIN: internal-muc.meet.jitsi
      JVB_AUTH_USER: jvb
      JVB_AUTH_PASSWORD: ${JVB_AUTH_PASSWORD}
      JVB_BREWERY_MUC: jvbbrewery
      JVB_PORT: "10000"
      JVB_TCP_HARVESTER_DISABLED: "true"
      DOCKER_HOST_ADDRESS: ${SERVER_HOST}
    volumes:
      - ./jitsi-config/jvb:/config
    depends_on:
      - prosody
    networks:
      - paio_network

  icd11-api:
    image: whoicd/icd-api:latest
    container_name: paio_icd11_api
    restart: unless-stopped
    environment:
      acceptLicense: "true"
      saveAnalytics: "false"
      include: "2026-01_en-es-pt"
    expose:
      - "80"
    networks:
      paio_network:
        aliases:
          - icd11-api

volumes:
  mysql_data:
  openemr_sites:
  openemr_logs:

networks:
  paio_network:
    driver: bridge
```

Guarde y salga: `Ctrl+O` → `Enter` → `Ctrl+X`.

---

## 7. Iniciar los contenedores

### 7.1 Levantar todos los servicios

```bash
cd ~/paio8
docker compose up -d
```

> **Nota:** La primera vez puede tardar varios minutos mientras Docker descarga las imágenes de OpenEMR, Jitsi, MariaDB y la API CIE-11 de la OMS. La descarga del contenido CIE-11 puede tardar más que el resto.

### 7.2 Verificar los contenedores

```bash
docker compose ps
```

Deberían aparecer los 7 contenedores con estado **Up**: `paio_mysql`, `paio_openemr`, `paio_jitsi_web`, `paio_jitsi_prosody`, `paio_jitsi_jicofo`, `paio_jitsi_jvb` y `paio_icd11_api`.

### 7.3 Monitorear la configuración inicial de OpenEMR

```bash
docker compose logs -f openemr
```

Espere hasta ver la línea:

```text
Setup Complete!
```

> **Nota:** No abra el navegador hasta que aparezca esa línea. Salga del log con `Ctrl+C`.

### 7.4 Verificar acceso

Abra OpenEMR en el navegador:

```text
https://emr.sudominio.com
```

Debe ver la página de inicio de sesión de OpenEMR. Inicie sesión con el usuario y la contraseña definidos en `.env` (`OE_USER` / `OE_PASS`).

Verifique también que Jitsi responde:

```text
https://emr.sudominio.com:8444
```

---

## 8. Clonar los módulos PAHO

Los cuatro módulos se clonan desde `github.com/diazjuan` dentro de la carpeta `custom_modules/`, que está montada en el contenedor de OpenEMR. Primero tome la propiedad de la carpeta:

```bash
sudo chown -R $USER:$(id -gn) ~/paio8/custom_modules
cd ~/paio8/custom_modules
```

Clone los cuatro repositorios (cada uno crea su propia carpeta con el nombre del repositorio):

```bash
git clone https://github.com/diazjuan/paho_jitsi_televisit_module.git
git clone https://github.com/diazjuan/paho_openemr_icd_11.git
git clone https://github.com/diazjuan/paho_openemr_translations.git
git clone https://github.com/diazjuan/paio_login_theme.git
```

> **Nota:** Respete los nombres tal cual: el módulo del tema de login es `paio_login_theme` (con **paio**, no paho).

Reinicie OpenEMR para que detecte los módulos recién clonados:

```bash
cd ~/paio8 && docker compose restart openemr
```

Espere unos 30 segundos antes de continuar.

---

## 9. Registrar, instalar y habilitar los módulos

Inicie sesión en OpenEMR como administrador y vaya a **Módulos → Gestionar Módulos** (*Modules → Manage Modules*). Los cuatro módulos aparecerán en la lista de **No registrados** (*Unregistered*). Para cada uno, en este orden:

1. Haga clic en **Registrar** (*Register*).
2. Haga clic en **Instalar** (*Install*).
3. Haga clic en **Habilitar** (*Enable*).

Realice los tres pasos para los cuatro módulos:

- PAHO Jitsi Televisit
- PAHO ICD-11 Integration
- PAIO Translations
- PAHO OPS Login Theme

> **Nota:** Una vez habilitado, cada módulo muestra un botón **Configurar** (*Configure* / icono de engranaje) en su fila. La configuración específica se detalla en el paso 10.

---

## 10. Configurar cada módulo

### 10.1 Tele Visita (`paho_jitsi_televisit_module`)

En **Gestionar Módulos**, abra **Configurar** en la fila de PAHO Jitsi Televisit y ajuste:

- **URL Base de Jitsi** (*Jitsi Base URL*): `https://emr.sudominio.com:8444`
- **Categoría de Tele Visita** (*Televisit Category*): seleccione la categoría de calendario que identifica las citas de Tele Visita (vea el paso 11).
- **Estrategia de nombres de sala:** deje **Determinística** (la misma cita siempre obtiene la misma sala).

En la sección **Autenticación JWT** el estado debe mostrarse en verde: *"La autenticación por token está configurada"*, con origen **environment**. Esto confirma que el `JWT_APP_SECRET` del archivo `.env` llegó correctamente al contenedor.

Haga clic en **Guardar Configuración**.

> **Nota:** El secreto y el App ID de JWT se leen del entorno (`.env`) y se muestran de solo lectura; no se editan desde esta pantalla.

### 10.2 CIE-11 (`paho_openemr_icd_11`)

Abra **Configurar** en la fila de PAHO ICD-11 Integration. Los valores predeterminados ya funcionan con la API que levantó Docker Compose; solo verifique que coincidan:

| Ajuste | Valor | Nota |
| --- | --- | --- |
| URL base de la API CIE-11 | `http://icd11-api` | URL interna de Docker; nunca se expone al navegador. |
| Release de CIE-11 | `2026-01` | Debe estar cargado en el contenedor de la API (clave `include` del compose). |
| Linearización | `mms` | Linearización clínica. |
| Idioma por defecto | `es` | Respaldo cuando el idioma de la interfaz no mapea a `en`, `es` o `pt`. |

Si los valores son correctos, guarde. El módulo consulta la API CIE-11 de la OMS que corre en el contenedor `paio_icd11_api`, accesible solo dentro de la red de Docker.

### 10.3 Traducciones (`paho_openemr_translations`)

Abra **Configurar** en la fila de PAIO Translations y haga clic en **Ejecutar** (*Run*). La página muestra el conteo de filas de `lang_constants` / `lang_definitions` antes y después, y la duración.

> **Nota:** El script de traducciones es idempotente: puede ejecutarlo varias veces sin riesgo. El módulo no crea respaldos automáticos; si desea poder revertir, haga un `mysqldump` de `lang_constants` y `lang_definitions` antes de ejecutarlo.

Limpie la caché y reinicie OpenEMR para que el idioma se aplique en toda la interfaz:

```bash
docker exec paio_openemr rm -rf /tmp/php-file-cache/*
cd ~/paio8 && docker compose restart openemr
```

### 10.4 Tema de inicio de sesión (`paio_login_theme`)

Este módulo no requiere configuración: al habilitarlo, la pantalla de inicio de sesión ya muestra el logo y los colores de OPS. Abra la página de login en una ventana privada para confirmarlo.

Cada sitio puede además añadir su propio logo sin tocar código:

1. **Administración → Logos:** suba su logo como **Secondary Login Logo** (`core/login/secondary`).
2. **Administración → Globales → Apariencia:** active el logo secundario de login.
3. El logo del sitio aparece debajo del logo de OPS.

---

## 11. Configuración inicial de OpenEMR

Antes de crear la primera Tele Visita, complete la configuración básica en este orden:

| Paso | Ubicación | Qué configurar |
| --- | --- | --- |
| 1 | Admin → Config → Locale | Idioma, zona horaria, formato de fecha |
| 2 | Admin → Config → Portal | Activar el portal del paciente y su URL |
| 3 | Admin → Clinic → Facilities | Nombre y datos de la clínica |
| 4 | Admin → Users | Médico: nombre, usuario, contraseña, rol Proveedor |
| 5 | Patient → New/Search | Paciente: datos, email y acceso al portal |
| 6 | Calendario → Categorías | Crear/identificar la categoría 'Tele Visita' (se vincula en el paso 10.1) |
| 7 | Calendario → clic en horario | Cita: categoría 'Tele Visita', paciente y proveedor |

---

## 12. Flujo de Tele Visita

### Flujo del médico

1. Abrir el Calendario en OpenEMR y hacer clic en la cita de Tele Visita.
2. En el panel "Tele Visita", hacer clic en "Abrir Vista Unificada de Tele Visita".
3. Se abre la vista con el video de Jitsi y el formulario de encuentro.
4. Hacer clic en "Abrir Conferencia de Video" para iniciar la sesión.
5. Documentar el encuentro en el formulario durante la consulta.

### Flujo del paciente (Portal)

1. El paciente accede al portal: `https://emr.sudominio.com/portal`.
2. Inicia sesión con sus credenciales.
3. Va a Citas y busca la cita con el indicador "Tele Visita".
4. Hace clic en "Unirse a Tele Visita".

> **Nota:** El acceso a la sala está protegido por JWT: tanto el médico como el paciente entran con un token firmado que emite OpenEMR. Nadie sin token válido puede unirse a la sala.

---

## 13. Mantenimiento y actualización de módulos

Para actualizar un módulo a la última versión publicada en `github.com/diazjuan`, entre en su carpeta y haga `git pull`; luego reinicie OpenEMR:

```bash
cd ~/paio8/custom_modules/paho_jitsi_televisit_module && git pull
cd ~/paio8 && docker compose restart openemr
```

> **Nota:** Repita el `git pull` en la carpeta del módulo que desee actualizar. Si una actualización incluye cambios de esquema, vuelva a Gestionar Módulos y siga la indicación de actualización del Module Manager.

---

## 14. Comandos de referencia rápida

| Comando | Descripción |
| --- | --- |
| `docker compose ps` | Ver el estado de todos los contenedores |
| `docker compose logs -f openemr` | Ver los logs de OpenEMR en tiempo real |
| `docker compose logs -f jitsi-web` | Ver los logs de Jitsi en tiempo real |
| `docker compose restart openemr` | Reiniciar solo OpenEMR |
| `docker compose down` | Detener todos los servicios |
| `docker compose up -d` | Iniciar todos los servicios |
| `docker exec paio_openemr rm -rf /tmp/php-file-cache/*` | Limpiar la caché de OpenEMR |
| `cd ~/paio8/custom_modules/<modulo> && git pull` | Actualizar un módulo desde GitHub |
