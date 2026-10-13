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

### 2.1 Crear el repositorio de imagenes

En Docker Hub crea estos repositorios bajo tu usuario:

```text
frontend
backend
database
```

Si tu usuario de Docker Hub es `TU_DOCKERHUB_USER`, las imagenes seran:

```text
TU_DOCKERHUB_USER/frontend
TU_DOCKERHUB_USER/backend
TU_DOCKERHUB_USER/database
```

### 2.2 Crear un Access Token

En Docker Hub:

1. Abre tu perfil.
2. Entra a **Account settings**.
3. Abre **Personal access tokens**.
4. Selecciona **Generate new token**.
5. Dale un nombre, por ejemplo `github-actions-cicd-lab`.
6. Usa permiso **Read & Write**.
7. Copia el token inmediatamente.

El token se muestra una sola vez. No lo guardes en el repositorio.

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

Conectate a `cicd-lab-backend` y ejecuta:

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
```

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
DB_USER=usuario-creado-en-RDS
DB_PASSWORD=contraseña-de-RDS
```

La contraseña nunca debe escribirse en el codigo ni en un workflow.

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
| `DOCKERHUB_USERNAME` | Tu usuario de Docker Hub |
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

El runner debe poder conectarse por SSH a frontend y backend:

```text
runner → frontend:22
runner → backend:22
```

No abras SSH a `0.0.0.0/0`. Usa el Security Group del runner como origen si las reglas de AWS lo permiten, o la IP privada del runner para este laboratorio.

---

## Fase 7: Configurar CI

El CI debe construir las imagenes en un mismo job o repetir el checkout y la construccion en cada job. Los archivos e imagenes de un job no aparecen automaticamente en otro runner.

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
TU_DOCKERHUB_USER/frontend:${{ github.sha }}
TU_DOCKERHUB_USER/backend:${{ github.sha }}
```

No dependas unicamente de `latest`.

### Verificacion de CI

Haz un cambio pequeno en `github/frontend/index.html`, por ejemplo un texto visible, y ejecuta:

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

El CD se ejecuta solo despues de un CI exitoso.

Para cada EC2, el despliegue debe:

1. Conectarse por SSH.
2. Hacer login en Docker Hub.
3. Descargar la imagen indicada por SHA.
4. Detener el contenedor anterior si existe.
5. Eliminarlo si existe.
6. Iniciar el nuevo contenedor.
7. Comprobar el endpoint local.
8. Fallar si la comprobacion no es exitosa.

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
  TU_DOCKERHUB_USER/backend:TAG
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
  TU_DOCKERHUB_USER/frontend:TAG
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
