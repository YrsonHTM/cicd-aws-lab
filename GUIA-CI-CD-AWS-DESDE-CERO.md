# Guia CI/CD en AWS desde cero

Esta guia explica como llevar este proyecto a un repositorio propio de GitHub y desplegarlo en las instancias EC2 usando Docker Hub y GitHub Actions.

La infraestructura AWS ya creada se conserva. Esta guia comienza en GitHub y termina verificando el flujo completo:

```text
GitHub
  -> GitHub Actions en el runner de AWS
  -> Docker Hub
  -> EC2 frontend y EC2 backend
  -> RDS PostgreSQL
  -> ALB
```

## Resultado esperado

Al terminar:

- El repositorio pertenecera a tu cuenta de GitHub.
- El runner aparecera como `Idle` en GitHub.
- CI ejecutara pruebas, construira imagenes y las publicara en Docker Hub.
- CD conectara el runner con las EC2 y desplegara las imagenes.
- El frontend respondera a traves del ALB.
- El backend usara RDS, no un contenedor PostgreSQL.

## Estado que debe existir antes de comenzar

No repitas la creacion de estos recursos:

- VPC `cicd-lab-vpc`.
- Subnets publicas y privadas.
- `alb-sg`, `frontend-sg`, `backend-sg`, `rds-sg` y `runner-sg`.
- RDS PostgreSQL `cicd-lab-postgres`.
- EC2 `cicd-lab-frontend`.
- EC2 `cicd-lab-backend`.
- EC2 `cicd-lab-runner`.
- Private hosted zone y registro privado del backend.
- ALB `cicd-lab-alb`.
- Target Group `cicd-lab-frontend-tg`.

La hoja de control no se actualiza en esta fase porque se solicito continuar sin modificarla.

---

## Fase 1: Crear tu repositorio propio

### 1.1 Crear el repositorio vacio

1. Entra a GitHub con tu cuenta.
2. Selecciona **New repository**.
3. Usa un nombre, por ejemplo:

   ```text
   cicd-aws-lab
   ```

4. Selecciona **Private** mientras el laboratorio este en construccion.
5. No marques:
   - Add a README file.
   - Add `.gitignore`.
   - Choose a license.
6. Selecciona **Create repository**.

El repositorio debe quedar vacio para poder subir el contenido local sin conflictos.

### 1.2 Cambiar el remoto local

En la terminal abierta en `C:\cursoDevOps\cicd`, ejecuta:

```bash
git remote set-url origin https://github.com/TU_USUARIO/cicd-aws-lab.git
git remote -v
```

El resultado debe mostrar la URL de tu repositorio, no la URL de la organizacion anterior.

### 1.3 Revisar archivos antes de subirlos

No subas:

- Archivos `.pem`.
- Contraseñas.
- Tokens.
- Archivos `.env`.
- Claves privadas.
- Salidas de comandos que contengan secretos.

Revisa el estado:

```bash
git status
```

Si existe un archivo `.env`, agregalo al `.gitignore` antes de continuar:

```text
.env
.env.*
!.env.example
*.pem
```

### 1.4 Hacer el primer push

```bash
git add .
git commit -m "Prepare AWS CI CD lab"
git push -u origin main
```

Confirma en GitHub que aparezcan:

- `github/frontend`.
- `github/backend`.
- `github/db`.
- `.github/workflows`.
- Los Dockerfiles.

---

## Fase 2: Preparar Docker Hub

Docker Hub sera el registro donde CI publicara las imagenes y desde donde CD las descargara en las EC2.

### 2.1 Crear o confirmar la cuenta

1. Abre [Docker Hub](https://hub.docker.com/).
2. Selecciona **Sign up** si todavia no tienes cuenta.
3. Confirma el correo electronico.
4. Inicia sesion.
5. Selecciona tu avatar, normalmente en la esquina superior derecha.
6. Copia exactamente tu **Docker ID**.

El Docker ID no siempre coincide con el correo o con el nombre visible. Guardalo temporalmente con este formato:

```text
Docker ID: yersonhtm
```

Usaremos ese valor en los nombres de las imagenes. No uses espacios, mayusculas ni el correo electronico.

### 2.2 Crear el repositorio `frontend`

1. En Docker Hub selecciona **My Hub** o **Repositories**.
2. Selecciona **Create repository**.
3. En **Repository name**, escribe exactamente:

   ```text
   frontend
   ```

4. En **Visibility**, selecciona **Public**.
5. No agregues un Dockerfile desde Docker Hub.
6. No selecciones una plantilla de build.
7. Selecciona **Create**.

El nombre final debe verse asi:

```text
yersonhtm/frontend
```

### 2.3 Crear los repositorios restantes

Repite el mismo procedimiento dos veces:

1. Selecciona **Create repository**.
2. Usa el nombre `backend`.
3. En **Visibility**, selecciona **Public**.
4. Selecciona **Create**.
5. Vuelve a seleccionar **Create repository**.
6. Usa el nombre `database`.
7. Selecciona **Create**.

Al terminar, en **Repositories** deben aparecer exactamente:

```text
yersonhtm/frontend
yersonhtm/backend
yersonhtm/database
```

No crees repositorios con nombres como:

```text
cicd-frontend
frontend-image
postgres
```

El workflow usara los tres nombres anteriores.

### 2.4 Comprobar la visibilidad

Abre cada repositorio y confirma:

- El propietario es tu Docker ID.
- El nombre coincide exactamente.
- La visibilidad es `Public`.
- El repositorio esta vacio.
- No hay imagenes publicadas todavia.

Que esten vacios es correcto. Las primeras imagenes seran publicadas por GitHub Actions.

Los repositorios publicos son suficientes para este laboratorio. No incluyas nunca en una imagen:

- Contraseñas de RDS.
- Tokens de Docker Hub.
- Claves privadas.
- Archivos `.env`.
- Credenciales de AWS.

Las credenciales se entregaran al contenedor mediante variables de entorno durante el despliegue, no durante el build de la imagen.

### 2.5 Crear el Access Token

No uses tu contraseña de Docker Hub en GitHub Actions. Crea un token exclusivo para este laboratorio:

1. Selecciona tu avatar en Docker Hub.
2. Entra a **Account settings**.
3. Busca **Personal access tokens** o **Access tokens**.
4. Selecciona **Generate new token**.
5. En **Description**, escribe:

   ```text
   github-actions-cicd-lab
   ```

6. Si Docker Hub solicita una fecha de expiracion, elige una fecha cercana al final del laboratorio.
7. Selecciona el permiso minimo que permita publicar imagenes:
   - `Read & Write`, o
   - `Image: Read & Write`, segun la pantalla mostrada.
8. No selecciones permisos de administracion de la cuenta.
9. Selecciona **Generate**.
10. Copia el token en un administrador de contraseñas o en un lugar temporal seguro.

El token solo se muestra una vez. Si lo pierdes, revocalo y genera uno nuevo. No lo pegues en:

- El codigo fuente.
- `ci.yml`.
- `cd.yml`.
- Un archivo `.env` que vaya a GitHub.
- Un mensaje de commit.
- Una captura de pantalla.

### 2.6 Resultado que debes tener antes de continuar

Al finalizar esta fase debes conocer solamente estos dos datos:

```text
DOCKERHUB_USERNAME = yersonhtm
DOCKERHUB_TOKEN    = el token recien creado
```

El token no se escribe aun en ningun archivo. En la Fase 6 se guardara como un **GitHub Actions secret**.

Usa esta comprobacion visual:

| Comprobacion | Resultado esperado |
|---|---|
| Cuenta | Puedes iniciar sesion en Docker Hub |
| Docker ID | Identificado exactamente |
| Repositorio `frontend` | Existe, es publico y esta vacio |
| Repositorio `backend` | Existe, es publico y esta vacio |
| Repositorio `database` | Existe, es publico y esta vacio |
| Token | Creado y guardado de forma segura |
| Codigo fuente | No contiene el token |

No avances a la Fase 3 hasta tener los tres repositorios creados. No necesitas ejecutar `docker login` en tu computador: el login se hara desde GitHub Actions usando el secret.

---

## Fase 3: Preparar y registrar el runner

El runner es la EC2 que ejecutara los jobs de GitHub Actions. Crear la instancia EC2 no registra automaticamente el runner en GitHub.

### 3.1 Crear el runner en GitHub

1. Abre tu repositorio.
2. Ve a **Settings**.
3. Entra a **Actions**.
4. Selecciona **Runners**.
5. Selecciona **New self-hosted runner**.
6. Selecciona:
   - Operating system: `Linux`.
   - Architecture: `x64`.
7. Deja esta pagina abierta.

GitHub mostrara comandos personalizados con un token temporal. Usa esos comandos; no copies un token de otra instalacion.

### 3.2 Preparar la EC2 runner

Conectate a `cicd-lab-runner` por EC2 Instance Connect y ejecuta:

```bash
sudo apt-get update
sudo apt-get install -y docker.io git curl
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"
```

Cierra la terminal y vuelve a conectarte para que el grupo Docker se aplique.

Comprueba:

```bash
docker version
git --version
```

### 3.3 Instalar el runner

En la EC2 runner, ejecuta los comandos que GitHub mostro en la pagina **New self-hosted runner**.

Normalmente el flujo es:

```bash
mkdir actions-runner
cd actions-runner
curl -o actions-runner.tar.gz -L URL_MOSTRADA_POR_GITHUB
tar xzf actions-runner.tar.gz
./config.sh --url https://github.com/TU_USUARIO/cicd-aws-lab --token TOKEN_TEMPORAL
./run.sh
```

No reutilices el token de ejemplo. El token real debe venir de la pantalla de GitHub.

Cuando GitHub muestre el runner como `Idle`, abre otra terminal en la EC2 y dejalo como servicio:

```bash
cd ~/actions-runner
sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status
```

En GitHub debe permanecer como:

```text
Idle
```

`Idle` significa que esta conectado y esperando trabajos. `Offline` significa que el CD no podra ejecutarse.

---

## Fase 4: Preparar las EC2 para el despliegue

Esta preparacion se hace una sola vez en frontend y backend.

### 4.1 Frontend

Conectate a `cicd-lab-frontend` y ejecuta:

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
```

El puerto 80 ya debe estar permitido desde `alb-sg`.

### 4.2 Backend

El backend esta en una subnet privada y no tiene IPv4 publica. No intentes usar **EC2 Instance Connect** desde el navegador para esta instancia.

Primero agrega temporalmente en `backend-sg` esta regla:

| Type | Port | Source |
|---|---:|---|
| SSH | `22` | `runner-sg` |

No uses `0.0.0.0/0` como origen.

Desde tu computador, copia temporalmente la clave `.pem` a la EC2 runner usando la IPv4 publica de la runner:

```powershell
scp -i "C:\ruta\laboratorio.pem" "C:\ruta\laboratorio.pem" ubuntu@IP_PUBLICA_RUNNER:/home/ubuntu/backend-key.pem
```

Si Windows muestra `Bad permissions` o `UNPROTECTED PRIVATE KEY FILE`, corrige los permisos de la clave antes de repetir el comando:

```powershell
$Key = "C:\ruta\laboratorio.pem"
icacls $Key /inheritance:r
icacls $Key /remove "BUILTIN\Usuarios"
icacls $Key /remove "BUILTIN\Users"
icacls $Key /remove "NT AUTHORITY\Authenticated Users"
icacls $Key /grant:r "$($env:USERNAME):(R)"
icacls $Key
```

La salida debe dejar la clave accesible para tu usuario y no para grupos generales ni para `Authenticated Users`.

En la terminal de `cicd-lab-runner`, protege la clave:

```bash
chmod 400 ~/backend-key.pem
```

Desde esa misma terminal, conecta usando la IP privada de `cicd-lab-backend`:

```bash
ssh -i ~/backend-key.pem ubuntu@IP_PRIVADA_BACKEND
```

Acepta la huella con `yes` solamente si la IP privada corresponde a tu instancia backend.

Ya dentro del backend, ejecuta:

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
```

La clave no debe subirse al repositorio ni pegarse en un workflow. Mas adelante CD usara una copia protegida mediante un GitHub Actions secret.

El puerto 5000 debe estar permitido solamente desde `frontend-sg`.

### 4.3 No instales PostgreSQL en el backend

La base de datos de AWS es RDS. El contenedor de `github/db` se conserva para desarrollo y pruebas, pero no se debe iniciar en la EC2 backend.

---

## Fase 5: Configurar la aplicacion para AWS

Antes de ejecutar CD hay que quitar las referencias locales de Docker Compose.

### 5.1 Backend: usar variables de entorno

El backend no debe usar:

```python
host="database"
```

Debe leer la configuracion de RDS desde variables de entorno:

```text
DB_HOST
DB_PORT
DB_NAME
DB_USER
DB_PASSWORD
```

Valores esperados en AWS:

```text
DB_HOST=cicd-lab-postgres.cyx26me4q2u8.us-east-1.rds.amazonaws.com
DB_PORT=5432
DB_NAME=saludos_db
DB_USER=saludos_admin
DB_PASSWORD=contraseña-de-RDS
```

Estos nombres son variables que el contenedor recibira durante el despliegue. No son nombres de archivos ni columnas de la base de datos.

La contraseña nunca debe escribirse en el codigo, en un workflow, en Docker Hub ni en un commit.

#### Donde crear estos valores

Se crearan como **Repository secrets de GitHub Actions**:

1. Abre tu repositorio propio en GitHub.
2. Selecciona **Settings**.
3. En el menu izquierdo entra a **Secrets and variables**.
4. Selecciona **Actions**.
5. En la pestaña **Secrets**, selecciona **New repository secret**.
6. Crea uno por uno:

   ```text
   DB_HOST
   DB_PORT
   DB_NAME
   DB_USER
   DB_PASSWORD
   ```

7. En cada formulario escribe el valor real y selecciona **Add secret**.

Usa los valores definidos al crear RDS:

| Secret | Valor |
|---|---|
| `DB_HOST` | Endpoint de RDS, sin `http://` |
| `DB_PORT` | `5432` |
| `DB_NAME` | `saludos_db` |
| `DB_USER` | `saludos_admin` |
| `DB_PASSWORD` | Contraseña maestra de RDS |

GitHub no volvera a mostrar el valor de un secret despues de guardarlo. Verifica cada valor antes de seleccionar **Add secret**.

No debes crear estos valores en **Docker Hub** ni guardarlos en un archivo `.env` que se suba a GitHub. El workflow los recibira como secrets y CD los pasara al contenedor con `-e`.

### 5.2 Frontend: usar el DNS privado del backend

En `github/frontend/nginx.conf`, el proxy debe apuntar al nombre privado creado en Route 53:

```nginx
proxy_pass http://backend.internal.example:5000/;
```

No uses:

```nginx
proxy_pass http://backend:5000/;
```

El nombre `backend` solo funciona cuando ambos contenedores comparten una red Docker.

### 5.3 Probar la red antes de CD

Desde la EC2 frontend:

```bash
getent hosts backend.internal.example
curl http://backend.internal.example:5000/health
```

Desde la EC2 backend, prueba RDS:

```bash
nc -vz cicd-lab-postgres.cyx26me4q2u8.us-east-1.rds.amazonaws.com 5432
```

No avances si estas pruebas fallan. CD no puede corregir un problema de DNS o Security Groups.

---

## Fase 6: Crear los Secrets de GitHub

En el repositorio abre:

**Settings → Secrets and variables → Actions → New repository secret**

Crea estos secrets:

| Nombre | Valor |
|---|---|
| `DOCKERHUB_USERNAME` | `yersonhtm` |
| `DOCKERHUB_TOKEN` | Token de Docker Hub |
| `DB_HOST` | Endpoint de RDS |
| `DB_PORT` | `5432` |
| `DB_NAME` | Nombre de la base de datos |
| `DB_USER` | Usuario de RDS |
| `DB_PASSWORD` | Contraseña de RDS |
| `FRONTEND_HOST` | IP privada de la EC2 frontend |
| `BACKEND_HOST` | IP privada de la EC2 backend |
| `SSH_PRIVATE_KEY` | Contenido de la clave privada EC2 |

### Regla para la clave privada

`SSH_PRIVATE_KEY` debe contener el archivo completo, incluyendo:

```text
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
```

o el formato equivalente que tenga tu archivo.

No subas ese archivo al repositorio. GitHub debe mostrar el secret como configurado, pero nunca volvera a mostrar su valor.

### Acceso de red para CD

Para que CD pueda desplegar, el runner debe conectarse por SSH a las dos EC2 usando sus IP privadas:

```text
runner → IP privada frontend:22
runner → IP privada backend:22
```

#### Regla SSH en `frontend-sg`

1. Abre **EC2 → Security Groups**.
2. Selecciona `frontend-sg`.
3. Abre **Inbound rules**.
4. Selecciona **Edit inbound rules**.
5. Selecciona **Add rule**.
6. Configura:

   ```text
   Type: SSH
   Protocol: TCP
   Port: 22
   Source: Custom
   Source value: ID de runner-sg
   ```

7. Guarda con **Save rules**.

#### Regla SSH en `backend-sg`

Repite el mismo procedimiento seleccionando `backend-sg`:

```text
Type: SSH
Protocol: TCP
Port: 22
Source: ID de runner-sg
```

En ambos casos selecciona el ID del Security Group `runner-sg`, no el nombre escrito manualmente y no `0.0.0.0/0`.

La regla final debe permitir:

```text
runner-sg → frontend:22
runner-sg → backend:22
```

El **Outbound** de `runner-sg` debe permitir el trafico necesario, normalmente:

```text
All traffic → 0.0.0.0/0
```

#### Verificar las reglas en AWS

En cada instancia:

1. Abre **EC2 → Instances**.
2. Selecciona la instancia.
3. Abre la pestaña **Security**.
4. Confirma que el Security Group asociado sea el correcto:
   - Frontend: `frontend-sg`.
   - Backend: `backend-sg`.
   - Runner: `runner-sg`.

En la pestaña **Networking**, copia las IP privadas de frontend y backend. No uses las IP publicas para CD.

#### Probar desde la runner

Desde la terminal de la EC2 runner prueba primero los puertos:

```bash
nc -vz IP_PRIVADA_FRONTEND 22
nc -vz IP_PRIVADA_BACKEND 22
```

El resultado esperado es:

```text
succeeded
```

Si `nc` no esta instalado:

```bash
sudo apt-get update
sudo apt-get install -y netcat-openbsd
```

Despues prueba la autenticacion SSH:

```bash
ssh -i ~/backend-key.pem ubuntu@IP_PRIVADA_FRONTEND
ssh -i ~/backend-key.pem ubuntu@IP_PRIVADA_BACKEND
```

La clave debe corresponder al key pair con el que se creo cada instancia. Si frontend y backend usan key pairs diferentes, usa la clave correspondiente en cada comando.

No abras SSH a `0.0.0.0/0`. La regla temporal de EC2 Instance Connect (`18.206.107.24/29`) es diferente y no reemplaza el acceso del runner.

---

## Fase 7: Configurar CI

El CI debe construir las imagenes en un mismo job o repetir el checkout y la construccion en cada job. Los archivos e imagenes de un job no aparecen automaticamente en otro runner.

### 7.1 Confirmar los requisitos del runner

Antes de modificar el workflow, entra por EC2 Instance Connect a `cicd-lab-runner` y comprueba:

```bash
docker version
git --version
sudo systemctl is-active docker
```

Los resultados esperados son:

```text
Docker version ...
git version ...
active
```

Si Docker no esta activo:

```bash
sudo systemctl enable --now docker
```

El runner debe aparecer como `Idle` en:

```text
Repositorio → Settings → Actions → Runners
```

No continues si aparece como `Offline`.

### 7.2 Confirmar los secrets de CI

En:

```text
Repositorio → Settings → Secrets and variables → Actions → Secrets
```

confirma que existan:

```text
DOCKERHUB_USERNAME
DOCKERHUB_TOKEN
```

El valor de `DOCKERHUB_USERNAME` debe ser:

```text
yersonhtm
```

El valor de `DOCKERHUB_TOKEN` debe ser el Access Token de Docker Hub con permiso de lectura y escritura.

No agregues los valores directamente al archivo YAML. El workflow los recibe mediante `secrets`.

### 7.3 Revisar el workflow del repositorio

El archivo que se ejecutara es:

```text
.github/workflows/ci.yml
```

Debe contener un solo job principal con este orden:

```text
checkout
  → comprobar Docker
  → ejecutar pruebas backend
  → configurar Buildx
  → iniciar sesion en Docker Hub
  → construir y publicar frontend
  → construir y publicar backend
  → construir y publicar database
```

Se usa un solo job porque los archivos del checkout y las imagenes locales no se comparten automaticamente entre jobs que usan runners separados.

El workflow del repositorio ya usa:

```text
Runner: self-hosted
Usuario Docker Hub: yersonhtm
Tag inmutable: github.sha
Tag adicional: latest
```

Las imagenes publicadas seran:

```text
yersonhtm/frontend:SHA_DEL_COMMIT
yersonhtm/backend:SHA_DEL_COMMIT
yersonhtm/database:SHA_DEL_COMMIT
```

### 7.4 Revisar el cambio del backend

El archivo `github/backend/app.py` debe leer:

```python
DB_HOST
DB_PORT
DB_NAME
DB_USER
DB_PASSWORD
```

No debe contener:

```python
host="database"
```

ni una contraseña escrita directamente.

El endpoint `/health` no consulta RDS, por lo que puede usarse para comprobar que Flask arranco. El endpoint `/saludo` si consulta RDS y requiere que CD entregue correctamente las variables de base de datos.

### 7.5 Ejecutar el primer CI

Desde PowerShell, ubicado en la carpeta del repositorio local, revisa los cambios:

```powershell
git status
git diff --check
```

Agrega y publica los cambios:

```powershell
git add .github/workflows/ci.yml github/backend/app.py github/frontend/nginx.conf
git commit -m "Prepare CI images for AWS deployment"
git push origin main
```

### 7.6 Seguir la ejecucion en GitHub

1. Abre tu repositorio.
2. Selecciona **Actions**.
3. Selecciona **Continuous Integration**.
4. Abre la ejecucion mas reciente.
5. Abre el job **Test, build and publish images**.

Comprueba los pasos en este orden:

1. `Checkout`: termina en verde.
2. `Check Docker`: muestra la version de Docker.
3. `Run backend tests`: muestra los tests exitosos.
4. `Set up Docker Buildx`: termina en verde.
5. `Log in to Docker Hub`: termina en verde sin mostrar el token.
6. `Build and push frontend`: termina en verde.
7. `Build and push backend`: termina en verde.
8. `Build and push database`: termina en verde.

Si un paso falla, no avances a CD. Abre ese paso y usa su mensaje exacto para corregirlo.

### 7.7 Confirmar las imagenes en Docker Hub

En Docker Hub abre:

```text
yersonhtm/frontend
yersonhtm/backend
yersonhtm/database
```

Cada repositorio debe tener:

- Un tag largo igual al SHA del commit.
- El tag `latest`.

El SHA se puede consultar en GitHub en el commit que disparo el workflow.

El flujo minimo es:

```text
checkout
  -> instalar dependencias y ejecutar pruebas
  -> build frontend
  -> build backend
  -> login en Docker Hub
  -> push con tag SHA
```

Usa el SHA del commit como tag inmutable:

```text
yersonhtm/frontend:${{ github.sha }}
yersonhtm/backend:${{ github.sha }}
```

No dependas unicamente de `latest`.

### Verificacion de CI

Para una segunda prueba, haz un cambio pequeno en `github/frontend/index.html`, por ejemplo un texto visible, y ejecuta:

```bash
git add .
git commit -m "Test CI pipeline"
git push
```

En GitHub abre **Actions** y confirma:

1. El workflow inicia.
2. El runner aparece ejecutando el job.
3. Las pruebas terminan correctamente.
4. Las tres imagenes aparecen en Docker Hub.
5. El tag coincide con el SHA del commit.

---

## Fase 8: Configurar CD

El archivo que ejecuta el despliegue es:

```text
.github/workflows/cd.yml
```

El CD se ejecuta solo despues de un CI exitoso sobre la rama `main`. Usa el SHA del commit que termino correctamente en CI, no `latest`, para que frontend y backend correspondan a la misma version.

Antes de hacer el primer push que dispare CD, confirma que existan estos secrets:

```text
FRONTEND_HOST
BACKEND_HOST
DB_HOST
DB_PORT
DB_NAME
DB_USER
DB_PASSWORD
SSH_PRIVATE_KEY
```

Usa como hosts las IP privadas:

```text
FRONTEND_HOST=10.0.1.216
BACKEND_HOST=10.0.11.25
```

`SSH_PRIVATE_KEY` debe ser la clave que permite conectarse como `ubuntu` a las dos EC2. Si frontend y backend fueron creados con key pairs diferentes, este workflow requiere una adaptacion para recibir una clave por instancia.

Para cada EC2, el despliegue debe:

1. Conectarse por SSH.
2. Hacer login en Docker Hub.
3. Descargar la imagen indicada por SHA.
4. Detener el contenedor anterior si existe.
5. Eliminarlo si existe.
6. Iniciar el nuevo contenedor.
7. Comprobar el endpoint local.
8. Fallar si la comprobacion no es exitosa.

El CD no despliega la imagen `database` en AWS. La persistencia del ambiente AWS es RDS PostgreSQL.

### Despliegue del backend

El contenedor debe publicarse en el puerto 5000:

```bash
docker run -d \
  --name backend \
  --restart unless-stopped \
  -p 5000:5000 \
  -e DB_HOST="$DB_HOST" \
  -e DB_PORT="$DB_PORT" \
  -e DB_NAME="$DB_NAME" \
  -e DB_USER="$DB_USER" \
  -e DB_PASSWORD="$DB_PASSWORD" \
  yersonhtm/backend:TAG
```

Validacion:

```bash
curl http://localhost:5000/health
```

Debe responder con un estado exitoso.

### Despliegue del frontend

El contenedor debe publicarse en el puerto 80:

```bash
docker run -d \
  --name frontend \
  --restart unless-stopped \
  -p 80:80 \
  yersonhtm/frontend:TAG
```

Validacion:

```bash
curl -I http://localhost
```

Debe responder:

```text
HTTP/1.1 200 OK
```

El Target Group debe volver a mostrar el frontend como `Healthy`.

---

## Fase 9: Validacion completa

Ejecuta las comprobaciones en este orden:

### 9.1 Backend

Desde frontend:

```bash
curl http://backend.internal.example:5000/health
curl http://backend.internal.example:5000/saludo
```

### 9.2 Frontend

Desde el DNS del ALB:

```text
http://DNS_DEL_ALB
```

Abre la pagina y selecciona **Obtener saludo**.

### 9.3 Resultado esperado

```text
Usuario
  -> ALB:80
  -> frontend:80
  -> backend.internal.example:5000
  -> RDS:5432
```

---

## Errores frecuentes

| Sintoma | Causa probable | Revision |
|---|---|---|
| Runner `Offline` | Servicio detenido | `sudo ./svc.sh status` |
| CI no encuentra Docker | Docker no esta instalado en runner | `docker version` |
| Imagen no aparece en Docker Hub | Token o nombre incorrecto | Revisar secrets |
| Frontend `Unhealthy` | Nada escucha en 80 | `curl -I http://localhost` |
| Backend no responde | Contenedor detenido o puerto cerrado | `docker ps` y `backend-sg` |
| Backend devuelve error 500 | Variables RDS incorrectas | `docker logs backend` |
| No resuelve `backend.internal.example` | Hosted zone o VPC incorrecta | Route 53 y `getent hosts` |
| RDS rechaza conexion | `rds-sg` no permite desde `backend-sg` | Revisar regla 5432 |
| CD no conecta por SSH | Regla 22 o clave incorrecta | Security Group y secret |

---

## Fase 10: Cerrar el acceso temporal

Cuando CD ya funcione y no necesites EC2 Instance Connect:

1. Elimina SSH temporal de `frontend-sg`.
2. Elimina SSH publico innecesario de `backend-sg`.
3. Mantiene SSH solo desde el runner o desde la IP administrativa necesaria.
4. Deten las EC2 cuando no estes practicando.
5. Elimina el NAT Gateway si no lo necesitaras durante un periodo largo, porque genera costo.
6. Deten o elimina RDS segun las opciones disponibles y la politica del laboratorio.

No elimines ningun recurso hasta haber confirmado que no lo necesitas para la practica actual.

## Criterio de finalizacion

La fase CI/CD termina cuando:

- El repositorio propio recibe un push.
- CI termina exitosamente.
- Las imagenes aparecen en Docker Hub.
- CD actualiza backend y frontend.
- El Target Group muestra `Healthy`.
- `/health` responde.
- El boton del frontend obtiene un saludo desde RDS.
- No quedan secretos en el codigo.
- Los puertos no estan abiertos publicamente sin necesidad.
