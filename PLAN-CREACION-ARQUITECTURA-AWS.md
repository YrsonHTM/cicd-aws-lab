# Plan de creación de la arquitectura AWS

Este documento define cómo construiremos paso a paso la arquitectura descrita en [ARQUITECTURA-AWS.md](./ARQUITECTURA-AWS.md). El objetivo es crear una plataforma reproducible para aprender AWS, redes, Docker y CI/CD sin configurar recursos manualmente de forma desordenada.

La creación se hará por capas: primero la red y la seguridad, después la persistencia, luego el cómputo y finalmente el balanceador y el pipeline. Cada fase tendrá una validación antes de continuar.

## Resultado esperado

Al finalizar, el usuario podrá acceder al frontend mediante el DNS del Application Load Balancer:

```text
http://<dns-del-alb>/
```

El flujo esperado será:

```text
Usuario
  -> ALB público
  -> Frontend en EC2 pública
  -> Backend en EC2 privada
  -> RDS PostgreSQL privado
```

El despliegue de nuevas versiones será:

```text
GitHub Actions
  -> Docker Hub
  -> Runner self-hosted
  -> EC2 frontend/backend
  -> Verificación mediante health checks
```

## Estrategia de implementación

Todos los recursos se crearán directamente desde la **AWS Management Console**. El objetivo de esta primera iteración es aprender dónde se encuentra cada servicio, cómo se relacionan sus configuraciones y cómo validar la comunicación entre las capas.

La consola se utilizará para:

- Crear cada recurso en el orden correcto.
- Revisar visualmente las relaciones entre VPC, subredes, rutas y Security Groups.
- Observar estados, health checks y eventos de los servicios.
- Modificar parámetros durante el aprendizaje de forma controlada.
- Eliminar los recursos al finalizar la práctica.
- Abrir sesiones de administración mediante EC2 Instance Connect cuando sea necesario.

No se utilizarán Terraform, AWS CLI ni scripts locales de aprovisionamiento. La creación, configuración y revisión de los servicios AWS se hará desde la interfaz gráfica.

Los comandos de Linux y Docker que sean necesarios para configurar o probar los contenedores se ejecutarán dentro de la instancia mediante **EC2 Instance Connect**, abierto desde la consola. Así también se practicará la operación de EC2 sin instalar herramientas de administración AWS en el equipo local.

### Alcance inicial

La primera versión tendrá:

- Una VPC.
- Dos subredes públicas en dos Availability Zones.
- Una subred privada para la aplicación.
- Dos subredes privadas para el DB subnet group.
- Internet Gateway.
- Un NAT Gateway compartido.
- Route tables públicas y privadas.
- Security Groups por capa.
- Una instancia RDS PostgreSQL.
- Una EC2 para frontend.
- Una EC2 para backend.
- Una EC2 para el GitHub Runner.
- Un Application Load Balancer.
- Target group y health check para el frontend.

### Elementos fuera de la primera versión

No se crearán inicialmente:

- Auto Scaling Groups.
- RDS Multi-AZ.
- NAT Gateway por Availability Zone.
- WAF.
- Bastion host.
- ECS o EKS.
- Pipeline basado en CodePipeline.
- HTTPS con dominio propio, hasta contar con dominio y certificado.

Estas capacidades podrán añadirse después como una iteración independiente.

## Prerrequisitos

Antes de iniciar se debe tener:

- Una cuenta AWS activa.
- MFA activado para el usuario raíz.
- Un usuario o rol administrativo para el laboratorio.
- Git instalado para trabajar con el repositorio.
- Una cuenta Docker Hub.
- Un repositorio GitHub propio.
- Permisos para crear VPC, EC2, RDS, ELB, IAM y CloudWatch.

### Verificación desde la consola

Antes de crear cualquier recurso:

1. Ingresar a la AWS Management Console.
2. Confirmar en la esquina superior derecha la cuenta y la región seleccionada.
3. Abrir **IAM �?' Dashboard** y verificar el usuario o rol activo.
4. Abrir **Billing and Cost Management �?' Budgets** y confirmar que exista el presupuesto.
5. Mantener la misma región seleccionada durante todo el laboratorio.

Git y Docker Hub se utilizarán desde GitHub Actions y desde las instancias, no como herramientas para aprovisionar recursos AWS.

## Fase 0: Control de costos y seguridad de la cuenta

Esta fase se realiza antes de crear la VPC.

### Acciones

#### 0.1 Seleccionar la región

1. Ingresar a [AWS Management Console](https://console.aws.amazon.com/).
2. En la barra superior, abrir el selector de región.
3. Elegir una región cercana, por ejemplo `US East (N. Virginia) us-east-1`, si está disponible para tu cuenta.
4. Mantener esa región seleccionada durante todo el laboratorio.

> Todos los recursos de red, EC2, RDS y ALB deben crearse en la misma región.

#### 0.2 Crear un presupuesto mensual

1. En la barra de búsqueda superior escribir **Billing**.
2. Abrir **Billing and Cost Management**.
3. En el menú izquierdo seleccionar **Budgets**.
4. Seleccionar **Create budget**.
5. Elegir **Use a template**.
6. Seleccionar la plantilla **Monthly cost budget**.
7. En **Budget name** escribir `cicd-lab-monthly-budget`.
8. En **Budgeted amount** introducir un límite pequeño para recibir alertas tempranas, por ejemplo `20 USD` mensuales.
9. En **Email recipients** escribir el correo que revisas habitualmente.
10. Seleccionar **Create budget**.

Para alertas más detalladas:

1. Volver a **Billing �?' Budgets**.
2. Abrir `cicd-lab-monthly-budget`.
3. Seleccionar **Edit**.
4. Agregar alertas en los porcentajes `50`, `80` y `100`.
5. Confirmar el correo de notificación.
6. Seleccionar **Save changes**.

#### 0.3 Activar MFA y revisar el usuario

1. Buscar **IAM** en la consola.
2. Abrir **IAM �?' Dashboard**.
3. En **Security recommendations**, revisar si el usuario raíz tiene MFA.
4. Si no está activo, seleccionar **Add MFA** y registrar una aplicación autenticadora.
5. En **IAM �?' Users**, abrir el usuario que usarás para el laboratorio.
6. Confirmar que tenga permisos suficientes para crear VPC, EC2, RDS, ELB, IAM, Route 53, CloudWatch y Budgets.

No se debe usar el usuario raíz para las operaciones diarias.

#### 0.4 Revisar la oferta gratuita

1. Buscar **Billing**.
2. Abrir **Free Tier**.
3. Revisar los límites vigentes para EC2, RDS, EBS, Elastic Load Balancing y NAT Gateway.
4. Abrir **Cost and Usage Reports** o **Cost Explorer** para consultar el consumo actual.

La oferta gratuita depende de la antigüedad y configuración de la cuenta. No se debe asumir que todos los servicios del diagrama son gratuitos.

#### 0.5 Preparar la información del administrador

Antes de crear EC2, abrir un sitio de consulta de IP pública desde el navegador y anotar la IP actual con formato CIDR:

```text
TU_IP_PUBLICA/32
```

Ese valor se utilizará para restringir SSH o EC2 Instance Connect. No usar `0.0.0.0/0` para SSH.

### Validación

- El presupuesto aparece activo en AWS Budgets.
- Se conoce la región objetivo.
- La IP pública del administrador está identificada.
- MFA está activo.
- Cost Explorer no muestra un consumo inesperado previo.

## Fase 1: Preparar una hoja de control

Como la creación será manual, se mantendrá una hoja de control fuera del código con:

- Nombre y región de cada recurso.
- ID de cada recurso creado.
- Subred y Availability Zone.
- Security Group asociado.
- Fecha de creación.
- Fecha prevista de eliminación.
- Dependencias con otros recursos.

No se deben guardar contraseñas ni claves privadas en esta hoja. La hoja sirve para no perder la trazabilidad de los recursos y facilitar la limpieza final.

## Fase 2: Crear la red VPC desde la consola

### Recursos

En la consola de AWS se abrirá **VPC �?' Your VPCs �?' Create VPC**. Para aprender cada componente, se seleccionará **VPC only** y los recursos se crearán manualmente.

#### 2.1 Crear la VPC

1. Buscar **VPC** en la barra superior.
2. Abrir **VPC �?' Your VPCs**.
3. Seleccionar **Create VPC**.
4. En **Resources to create**, seleccionar **VPC only**.
5. En **Name tag** escribir `cicd-lab-vpc`.
6. En **IPv4 CIDR**, escribir `10.0.0.0/16`.
7. En **IPv6 CIDR block**, seleccionar **No IPv6 CIDR block**.
8. En **Tenancy**, dejar **Default**.
9. Seleccionar **Create VPC**.
10. Abrir la VPC creada y anotar su **VPC ID** en la hoja de control.

#### 2.2 Crear las subredes

1. En el menú de VPC seleccionar **Subnets**.
2. Seleccionar **Create subnet**.
3. En **VPC ID**, seleccionar `cicd-lab-vpc`.
4. En la primera fila crear `cicd-public-a` con:
   - Availability Zone: la primera zona disponible, por ejemplo `us-east-1a`.
   - IPv4 subnet CIDR block: `10.0.1.0/24`.
5. Seleccionar **Add new subnet** una vez para agregar la segunda fila.
6. En la segunda fila crear `cicd-public-b` con:
   - Availability Zone: una segunda zona, por ejemplo `us-east-1b`.
   - IPv4 subnet CIDR block: `10.0.2.0/24`.
7. Seleccionar **Add new subnet** una vez para agregar la tercera fila.
8. En la tercera fila crear `cicd-private-app-a` con:
   - Availability Zone: la primera zona.
   - IPv4 subnet CIDR block: `10.0.11.0/24`.
9. Seleccionar **Add new subnet** una vez para agregar la cuarta fila.
10. En la cuarta fila crear `cicd-private-db-a` con:
   - Availability Zone: la primera zona.
   - IPv4 subnet CIDR block: `10.0.21.0/24`.
11. Seleccionar **Add new subnet** una vez para agregar la quinta fila.
12. En la quinta fila crear `cicd-private-db-b` con:
   - Availability Zone: la segunda zona.
   - IPv4 subnet CIDR block: `10.0.22.0/24`.
13. Confirmar que aparecen exactamente cinco filas completas.
14. Seleccionar **Create subnet**.

> No debes pulsar **Add new subnet** después de crear la quinta fila. Cada clic agrega una fila nueva; si aparece una sexta fila vacía, elimínala antes de seleccionar **Create subnet**.

Para las dos subredes públicas:

1. Seleccionar `cicd-public-a`.
2. Elegir **Actions �?' Edit subnet settings**.
3. Activar **Enable auto-assign public IPv4 address**.
4. Seleccionar **Save**.
5. Repetir los pasos para `cicd-public-b`.

No activar asignación automática de IP pública en las subredes privadas.

#### 2.3 Crear y asociar el Internet Gateway

1. En el menú de VPC seleccionar **Internet gateways**.
2. Seleccionar **Create internet gateway**.
3. En **Name tag** escribir `cicd-lab-igw`.
4. Seleccionar **Create internet gateway**.
5. Seleccionar el gateway creado.
6. Abrir **Actions �?' Attach to a VPC**.
7. Seleccionar `cicd-lab-vpc`.
8. Seleccionar **Attach internet gateway**.

#### 2.4 Crear la tabla de rutas públicas

1. En el menú de VPC seleccionar **Route tables**.
2. Seleccionar **Create route table**.
3. En **Name**, escribir `cicd-public-rt`.
4. En **VPC**, seleccionar `cicd-lab-vpc`.
5. Seleccionar **Create route table**.
6. Abrir la tabla creada.
7. Abrir la pestaña **Routes**.
8. Seleccionar **Edit routes**.
9. No modificar la fila existente `10.0.0.0/16 �?' local`; AWS la crea automáticamente.
10. Seleccionar **Add route** en la parte inferior de la pantalla.
11. En la nueva fila, en **Destination** escribir `0.0.0.0/0`.
12. En **Target**, elegir **Internet Gateway** y seleccionar `cicd-lab-igw`.
13. Seleccionar **Save changes**.
14. Abrir la pestaña **Subnet associations**.
15. Seleccionar **Edit subnet associations**.
16. Marcar `cicd-public-a` y `cicd-public-b`.
17. Seleccionar **Save associations**.

> Si solo aparecen `Instance`, `Network Interface`, `Gateway Load Balancer Endpoint` y `local`, estás editando la fila local existente. Pulsa **Add route** primero; la nueva fila mostrará los destinos de gateway disponibles.

#### 2.5 Crear el NAT Gateway

1. En el menú de VPC seleccionar **NAT gateways**.
2. Seleccionar **Create NAT gateway**.
3. En **Name**, escribir `cicd-lab-nat`.
4. En **Availability mode**, seleccionar **Zonal**.
5. En **VPC**, seleccionar `cicd-lab-vpc`.
6. Ahora aparecerá el campo **Subnet**; seleccionar `cicd-public-a`.
7. En **Connectivity type**, seleccionar **Public**.
8. En **Elastic IP allocation ID**, seleccionar una Elastic IP disponible.
9. Si no aparece ninguna Elastic IP disponible, seleccionar **Allocate Elastic IP** y confirmar la asignación.
10. Verificar que la Elastic IP seleccionada no esté asociada a otro recurso.
11. Seleccionar **Create NAT gateway**.
12. Esperar hasta que el estado sea **Available**.
13. Anotar el ID del NAT Gateway, el ID de asignación `eipalloc-...` y la dirección IPv4 pública.

La consola puede mostrar **Regional** como opción predeterminada. No seleccionar esa opción para esta primera versión: el modo Regional no solicita una subnet y cambia la forma en que se representa el NAT. Usaremos **Zonal** para ubicarlo explícitamente en `cicd-public-a`, como está definido en este plan.

El NAT Gateway cobra mientras está disponible. Si se detiene el laboratorio, debe eliminarse desde **VPC �?' NAT gateways**.

#### 2.6 Crear la tabla de rutas privadas de aplicación

1. Abrir **VPC �?' Route tables**.
2. Seleccionar **Create route table**.
3. En **Name**, escribir `cicd-private-app-rt`.
4. Seleccionar `cicd-lab-vpc`.
5. Seleccionar **Create route table**.
6. Abrir la tabla creada y entrar en **Routes �?' Edit routes**.
7. Seleccionar **Add route**.
8. En **Destination** escribir `0.0.0.0/0`.
9. En **Target**, elegir **NAT Gateway** y seleccionar `cicd-lab-nat`.
10. Seleccionar **Save changes**.
11. Abrir **Subnet associations �?' Edit subnet associations**.
12. Marcar `cicd-private-app-a`.
13. Seleccionar **Save associations**.

#### 2.7 Crear la tabla de rutas de base de datos

1. Seleccionar **Create route table**.
2. En **Name**, escribir `cicd-private-db-rt`.
3. Seleccionar `cicd-lab-vpc`.
4. Seleccionar **Create route table**.
5. Abrir **Subnet associations �?' Edit subnet associations**.
6. Marcar `cicd-private-db-a` y `cicd-private-db-b`.
7. Seleccionar **Save associations**.
8. No agregar una ruta `0.0.0.0/0` a esta tabla.

La ruta local de la VPC aparece automáticamente.

#### 2.8 Activar DNS de la VPC

1. Abrir **VPC �?' Your VPCs**.
2. Seleccionar `cicd-lab-vpc`.
3. Seleccionar **Actions �?' Edit VPC settings**.
4. Confirmar que **Enable DNS resolution** esté activo.
5. Confirmar que **Enable DNS hostnames** esté activo.
6. Seleccionar **Save**.

### Reglas de enrutamiento

Tabla pública:

```text
10.0.0.0/16 -> local
0.0.0.0/0   -> Internet Gateway
```

Tabla privada de aplicación:

```text
10.0.0.0/16 -> local
0.0.0.0/0   -> NAT Gateway
```

Tabla privada de RDS:

```text
10.0.0.0/16 -> local
```

### Validación

- Las subredes públicas tienen ruta al Internet Gateway.
- La subred privada de aplicación no tiene ruta directa al Internet Gateway.
- Las subredes de RDS no tienen exposición pública.
- El NAT Gateway está en una subred pública.

## Fase 3: Crear los Security Groups

Crea los grupos en este orden. Cada grupo tendra una regla de entrada exacta y una regla de salida exacta.

| Nombre visible | Uso |
|---|---|
| `alb-sg` | Application Load Balancer |
| `frontend-sg` | EC2 frontend |
| `backend-sg` | EC2 backend |
| `rds-sg` | RDS PostgreSQL |
| `runner-sg` | GitHub Actions Runner |

Para todos los grupos usa esta VPC:

```text
cicd-lab-vpc (vpc-0d9f5010052bd7577)
```

### Como crear cada grupo

1. Abrir **EC2 -> Network & Security -> Security Groups**.
2. Seleccionar **Create security group**.
3. Escribir el nombre visible indicado en la tabla.
4. Escribir una descripcion.
5. Seleccionar `cicd-lab-vpc` en **VPC**.
6. Agregar las reglas de entrada y salida indicadas abajo.
7. Seleccionar **Create security group**.
8. Copiar el **Security group ID** generado. El ID empieza con `sg-`.

Cuando una regla use otro grupo como origen, selecciona **Custom** y pega el ID completo del grupo. No escribas el nombre.

### `alb-sg`

Descripcion: `Public traffic to Application Load Balancer`

Inbound:

| Type | Port | Source |
|---|---:|---|
| HTTP | `80` | `0.0.0.0/0` |

Outbound:

| Type | Destination |
|---|---|
| All traffic | `0.0.0.0/0` |

No agregues HTTPS `443`.

### `frontend-sg`

Descripcion: `Frontend traffic from Application Load Balancer`

Inbound:

| Type | Port | Source |
|---|---:|---|
| HTTP | `80` | ID de `alb-sg` |

Outbound:

| Type | Destination |
|---|---|
| All traffic | `0.0.0.0/0` |

En **Source type**, elige **Custom** y pega el ID de `alb-sg`.

### `backend-sg`

Descripcion: `Backend API traffic from frontend`

Inbound:

| Type | Port | Source |
|---|---:|---|
| Custom TCP | `5000` | ID de `frontend-sg` |

Outbound:

| Type | Destination |
|---|---|
| All traffic | `0.0.0.0/0` |

En **Source type**, elige **Custom** y pega el ID de `frontend-sg`.

### `rds-sg`

Descripcion: `PostgreSQL traffic from backend`

Inbound:

| Type | Port | Source |
|---|---:|---|
| PostgreSQL | `5432` | ID de `backend-sg` |

Outbound:

| Type | Destination |
|---|---|
| All traffic | `0.0.0.0/0` |

En **Source type**, elige **Custom** y pega el ID de `backend-sg`.

### `runner-sg`

Descripcion: `GitHub Actions Runner access`

Inbound:

| Type | Port | Source |
|---|---:|---|
| SSH | `22` | **My IP** |

Outbound:

| Type | Destination |
|---|---|
| All traffic | `0.0.0.0/0` |

En **Source type**, selecciona **My IP**. No uses `0.0.0.0/0` para SSH.

### Verificacion antes de continuar

- `alb-sg`: HTTP `80` desde Internet; salida All traffic.
- `frontend-sg`: HTTP `80` desde `alb-sg`; salida All traffic.
- `backend-sg`: TCP `5000` desde `frontend-sg`; salida All traffic.
- `rds-sg`: PostgreSQL `5432` desde `backend-sg`; salida All traffic.
- `runner-sg`: SSH `22` desde **My IP**; salida All traffic.
- No existen reglas de entrada para `22`, `5000` o `5432` desde `0.0.0.0/0`.
- No crear otros Security Groups en esta fase.
## Fase 4: Crear la base de datos RDS

### Configuración inicial

#### 4.1 Crear el DB subnet group

1. Buscar **RDS** en la consola.
2. En el menú izquierdo seleccionar **Subnet groups**.
3. Seleccionar **Create DB subnet group**.
4. En **Name**, escribir `cicd-lab-db-subnet-group`.
5. En **Description**, escribir `Private subnets for PostgreSQL`.
6. En **VPC**, seleccionar `cicd-lab-vpc`.
7. En **Availability Zones**, seleccionar las dos zonas usadas por la VPC.
8. En **Subnets**, seleccionar `10.0.21.0/24` y `10.0.22.0/24`.
9. Seleccionar **Create**.

#### 4.2 Crear la instancia PostgreSQL

1. En RDS seleccionar **Databases**.
2. Seleccionar **Create database**.
3. Elegir **Standard create** para ver todas las opciones.
4. En **Engine options**, elegir **PostgreSQL**.
5. Seleccionar una versión disponible compatible con la aplicación.
6. En **Templates**, elegir **Free tier** si aparece habilitado para la cuenta.
7. En **DB instance identifier**, escribir `cicd-lab-postgres`.
8. En **Master username**, escribir `saludos_admin`.
9. En **Credentials management**, elegir administración propia de credenciales para este laboratorio.
10. Escribir una contraseña fuerte y guardarla fuera de Git.
11. En **DB instance size**, elegir la clase elegible para la oferta gratuita.
12. En **Storage**, mantener el mínimo permitido y desactivar el crecimiento automático si la cuenta lo permite.
13. En **Connectivity �?' Compute resource**, elegir **Don't connect to an EC2 compute resource**.
14. En **Virtual private cloud**, seleccionar `cicd-lab-vpc`.
15. En **DB subnet group**, seleccionar `cicd-lab-db-subnet-group`.
16. En **Public access**, seleccionar **No**.
17. En **VPC security group**, elegir **Choose existing**.
18. Quitar el Security Group predeterminado si aparece.
19. Seleccionar `rds-sg`.
20. En **Additional configuration �?' Initial database name**, escribir `saludos_db`.
21. En **Backup**, usar una retención mínima compatible con el laboratorio.
22. En **Encryption**, conservar la opción recomendada por AWS.
23. En **Deletion protection**, desactivarla para poder eliminar el laboratorio posteriormente.
24. Revisar el costo estimado.
25. Seleccionar **Create database**.

La instancia puede tardar varios minutos en pasar a **Available**. No continuar con la configuración del backend mientras permanezca en **Creating**.

La instancia resultante debe tener:

- DB subnet group usando `Private-Db-A` y `Private-Db-B`.
- `PubliclyAccessible = false`.
- `rds-sg`.
- Base de datos `saludos_db`.
- Usuario de aplicación dedicado.
- Almacenamiento mínimo compatible con la cuenta y el Free Tier.
- Backups automáticos conservadores para el laboratorio.

La contraseña se proporcionará mediante un mecanismo seguro. No se debe escribir como texto plano en el repositorio ni en la salida del pipeline.

#### 4.3 Copiar los datos de conexión

1. En **RDS �?' Databases**, abrir `cicd-lab-postgres`.
2. En la pestaña **Connectivity & security**, copiar el valor **Endpoint**.
3. Confirmar el puerto `5432`.
4. Guardar el endpoint en la hoja de control, nunca la contraseña.

El endpoint tendrá una forma similar a:

```text
cicd-lab-postgres.xxxxxxxxxxxx.us-east-1.rds.amazonaws.com
```

### Inicialización del esquema

El archivo `github/db/init.sql` se utilizará como referencia para crear las tablas iniciales, pero no se debe asumir que un Dockerfile de PostgreSQL ejecutará automáticamente ese script dentro de RDS.

La inicialización se hará mediante una de estas opciones:

1. Ejecutar una migración desde un job controlado.
2. Ejecutar un script desde la EC2 backend después de comprobar conectividad.
3. Utilizar una herramienta de migraciones de base de datos.

La opción elegida debe ser idempotente: repetirla no debe destruir datos existentes.

### Validación

- RDS está en estado `Available`.
- Su endpoint pertenece al dominio privado de AWS.
- No tiene dirección pública.
- El backend puede conectarse al puerto `5432`.
- Un cliente externo no puede conectarse directamente a RDS.

## Fase 5: Crear y configurar las instancias EC2

### 5.1 Crear una clave de acceso para EC2

1. Buscar **EC2** en la consola.
2. En el menú izquierdo seleccionar **Network & Security �?' Key pairs**.
3. Seleccionar **Create key pair**.
4. En **Name**, escribir `cicd-lab-key`.
5. En **Key pair type**, elegir **RSA**.
6. En **Private key file format**, elegir el formato compatible con tu sistema.
7. Seleccionar **Create key pair**.
8. Guardar el archivo descargado fuera del repositorio.

Aunque la administración se hará preferentemente con EC2 Instance Connect, la clave permite recuperar el acceso si la instancia no puede conectarse mediante el navegador.

### 5.2 Procedimiento común para lanzar una EC2

> Importante: la subnet de la interfaz de red principal no se puede cambiar después de lanzar una EC2. Verifica la subnet antes de seleccionar **Launch instance**. Si la instancia ya fue creada con la subnet incorrecta y todavía no contiene datos, termínala y créala nuevamente en la subnet correcta.

Para cada instancia:

1. Abrir **EC2 �?' Instances**.
2. Seleccionar **Launch instances**.
3. En **Name and tags �?' Name**, escribir el nombre de la instancia.
4. En **Application and OS Images**, elegir **Ubuntu Server 22.04 LTS** para mantener compatibilidad con los scripts del repositorio.
5. En **Instance type**, elegir el tipo elegible para la oferta gratuita de la cuenta.
6. En **Key pair**, seleccionar `cicd-lab-key`.
7. En **Network settings**, seleccionar **Edit**.
8. En **VPC**, seleccionar `cicd-lab-vpc`.
9. En **Subnet**, seleccionar la subred indicada para la instancia.
10. En **Auto-assign Public IP**, seleccionar el valor indicado para esa instancia.
11. En **Firewall**, seleccionar **Select existing security group**.
12. Marcar únicamente el Security Group indicado.
13. En **Configure storage**, conservar el volumen raíz mínimo necesario.
14. Abrir **Advanced details**.
15. En **IAM instance profile**, seleccionar el perfil correspondiente si se creó uno.
16. Para este laboratorio, dejar **User data** vacío. La configuración se realizará manualmente después mediante **EC2 Instance Connect**, para observar cada paso.
17. Revisar el resumen de costos y recursos.
18. Seleccionar **Launch instance**.
19. Esperar a que el estado sea **Running** y que **Status check** sea `2/2 checks passed`.
20. Anotar el Instance ID, la IP privada y, si existe, la IP pública.

No se debe seleccionar **Allow SSH traffic from anywhere**. Las reglas se aplicarán usando los Security Groups creados anteriormente.

Si todavía estás en la pantalla de lanzamiento, corregir la subnet así:

1. Abrir **Network settings → Edit**.
2. En **VPC**, seleccionar `cicd-lab-vpc`.
3. En **Subnet**, seleccionar la subnet correspondiente a la instancia.
4. Verificar **Auto-assign Public IP**.
5. En **Firewall**, seleccionar el Security Group correcto.
6. Continuar con **Launch instance**.

Valores de subnet por instancia:

| Instancia | Subnet | Auto-assign Public IP | Security Group |
|---|---|---|---|
| `cicd-lab-frontend` | `cicd-public-a` | `Enable` | `frontend-sg` |
| `cicd-lab-backend` | `cicd-private-app-a` | `Disable` | `backend-sg` |
| `cicd-lab-runner` | `cicd-public-b` | `Enable` | `runner-sg` |

### EC2 frontend

La instancia frontend se ubicará en una subred pública y ejecutará:

- Docker Engine.
- Contenedor frontend/Nginx.
- Configuración de Nginx para servir archivos estáticos.
- Proxy `/api/` hacia el backend privado.

Para crearla, usar:

- Name: `cicd-lab-frontend`.
- Subnet: `cicd-public-a`.
- Auto-assign Public IP: `Enable`.
- Security Group: `frontend-sg`.
- User data: instalación de Docker y preparación del despliegue frontend.

Para conectarte temporalmente desde **EC2 Instance Connect**:

1. Abrir **EC2 → Security Groups**.
2. Seleccionar `frontend-sg`.
3. Abrir **Inbound rules → Edit inbound rules**.
4. Seleccionar **Add rule**.
5. En **Type**, elegir **SSH**.
6. En **Source type**, elegir **Custom**.
7. En **Source**, escribir el rango que muestra la advertencia de EC2 Instance Connect para tu región. Para esta sesión, la consola muestra:

```text
18.206.107.24/29
```

8. Confirmar que el puerto sea `22`.
9. Seleccionar **Save rules**.
10. Volver a **EC2 → Instances → cicd-lab-frontend → Connect**.
11. En **EC2 Instance Connect**, seleccionar el usuario `ubuntu`.
12. Seleccionar el botón de actualizar si aparece la advertencia de SSH.
13. Seleccionar **Connect**.

Esta regla SSH para el rango de EC2 Instance Connect es solo para la configuración inicial. Cuando termines de instalar y probar Docker, vuelve a `frontend-sg → Inbound rules`, elimina la regla SSH `22` y conserva únicamente HTTP `80` desde `alb-sg`.

### Preparación inicial de Docker en el frontend

GitHub Actions será responsable del despliegue de las versiones de la aplicación. Sin embargo, la EC2 frontend necesita tener Docker instalado una primera vez. El runner no instala Docker automáticamente en otra EC2.

Ejecuta en la terminal de **EC2 Instance Connect** de `cicd-lab-frontend`:

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
```

La instalación anterior prepara la EC2, pero no despliega la aplicación. Para probar únicamente la red antes del pipeline, puedes ejecutar temporalmente:

```bash
sudo docker run -d --name frontend -p 80:80 nginx:alpine
curl -I http://localhost
```

La respuesta debe ser `HTTP/1.1 200 OK`. Este contenedor es una prueba opcional, no el despliegue final. Después:

1. Abrir **EC2 → Target Groups**.
2. Seleccionar `cicd-lab-frontend-tg`.
3. Abrir **Targets**.
4. Seleccionar **Refresh**.
5. Esperar hasta que `cicd-lab-frontend` aparezca como **healthy**.

Si ejecutaste la prueba, el pipeline deberá detener y reemplazar ese contenedor por la imagen frontend del proyecto. El flujo final será:

```text
GitHub Actions runner
  -> Docker Hub
  -> conexión a EC2 frontend
  -> docker pull
  -> docker stop/rm del contenedor anterior
  -> docker run de la nueva imagen
```

La prueba manual de Nginx se puede omitir si prefieres validar la aplicación únicamente mediante CI/CD.

La configuración no debe usar:

```nginx
proxy_pass http://backend:5000/;
```

salvo que frontend y backend compartan la misma red Docker en la misma instancia.

Debe utilizarse un nombre privado resoluble, por ejemplo:

```text
proxy_pass http://backend.internal.example:5000/;
```

### EC2 backend

La instancia backend se ubicará en una subred privada y ejecutará:

- Docker Engine.
- Contenedor Flask.
- Variables de conexión hacia RDS.
- Endpoint `/health`.
- Endpoint `/saludo`.

El backend no tendrá una IP pública ni acceso entrante desde Internet.

Para crearla, usar:

- Name: `cicd-lab-backend`.
- Subnet: `cicd-private-app-a`.
- Auto-assign Public IP: `Disable`.
- Security Group: `backend-sg`.
- User data: instalación de Docker y preparación del despliegue backend.

Para esta primera etapa no crees un EC2 Instance Connect Endpoint ni `eic-endpoint-sg`. Tampoco agregues una regla SSH `22` a `backend-sg`.

El backend permanece privado. Cuando lleguemos a la configuracion manual del backend, crearemos el acceso desde la consola como una etapa separada y te indicare exactamente que recurso crear y cuando eliminarlo. No necesitas resolver ese acceso ahora.
### EC2 runner

La instancia runner ejecutará:

- Docker Engine.
- GitHub Actions Runner.
- Herramientas necesarias para CI/CD.
- EC2 Instance Connect para administración desde el navegador.

Para crearla, usar:

- Name: `cicd-lab-runner`.
- Subnet: `cicd-public-b` o una subred pública dedicada.
- Auto-assign Public IP: `Enable`.
- Security Group: `runner-sg`.
- User data: instalación de Docker y preparación del GitHub Actions Runner.

El runner se registrará a nivel de repositorio usando un token temporal. El token no se guardará en la hoja de control ni en el repositorio.

### Configuración manual mediante EC2 Instance Connect

Para este laboratorio dejaremos **User data** vacío durante el lanzamiento. Esto no significa que la instancia quede sin configurar: después de crearla, entraremos desde el navegador mediante **EC2 Instance Connect** y ejecutaremos cada paso manualmente.

La configuración posterior será:

1. Actualizar Ubuntu.
2. Instalar Docker.
3. Habilitar y arrancar Docker.
4. Crear directorios de despliegue.
5. Descargar o preparar las imágenes.
6. Configurar las variables necesarias.
7. Iniciar el servicio correspondiente.
8. Probar la conectividad.

User data queda como una alternativa para automatizar estas tareas en futuras instancias. Si se utiliza más adelante, cada instancia debe tener un bloque diferente para frontend, backend y runner; no se debe reutilizar el mismo script.

Para revisar la configuración desde el navegador:

1. Abrir **EC2 �?' Instances**.
2. Seleccionar la instancia.
3. Seleccionar **Connect**.
4. Abrir **EC2 Instance Connect**.
5. Conectarse usando el botón **Connect**.
6. Revisar los logs de inicialización desde la terminal de la instancia.

Si la conexión no funciona, revisar primero que la instancia tenga el perfil IAM requerido, que el Security Group permita el acceso de administración y que el estado sea `Running`.

### Validación

- Las tres instancias aparecen en estado `Running`.
- Frontend puede descargar imágenes.
- Backend resuelve y alcanza el endpoint privado de RDS.
- Runner aparece como `Idle` en GitHub.
- No existen puertos innecesariamente abiertos.

## Fase 6: Resolver la comunicación frontend-backend

Como frontend y backend estarán en instancias diferentes, se creará un nombre privado para el backend.

### Opción inicial recomendada

#### 6.1 Crear la private hosted zone

1. Buscar **Route 53** en la consola.
2. En el menú izquierdo seleccionar **Hosted zones**.
3. Seleccionar **Create hosted zone**.
4. En **Domain name**, escribir `internal.example`.
5. En **Type**, elegir **Private hosted zone**.
6. En **Region**, seleccionar la región del laboratorio.
7. En **VPCs to associate**, seleccionar `cicd-lab-vpc`.
8. Seleccionar **Create hosted zone**.

#### 6.2 Crear el registro privado del backend

1. Abrir la zona `internal.example`.
2. Seleccionar **Create record**.
3. En **Record name**, escribir `backend`.
4. En **Record type**, seleccionar `A`.
5. Abrir otra pestaña y entrar en **EC2 → Instances → Instances**.
6. Seleccionar `cicd-lab-backend`.
7. En la pestaña **Networking**, copiar el valor **Private IPv4 addresses**.
8. Volver a Route 53 y pegar esa dirección en **Value**.
9. Dejar el TTL predeterminado.
10. Seleccionar **Create records**.

La dirección correcta debe pertenecer al rango privado de la VPC, por ejemplo `10.0.11.x`. No uses **Public IPv4 address**, **Elastic IP**, el endpoint de RDS ni el ID de la instancia.

El registro completo será:

```text
backend.internal.example
```

```text
backend.internal.example -> IP privada de cicd-lab-backend
```

Luego Nginx utilizará:

```nginx
proxy_pass http://backend.internal.example:5000/;
```

### Consideración sobre la IP privada

Si la EC2 backend se reemplaza, su IP privada puede cambiar. Para un laboratorio inicial se puede:

- Asociar una Elastic Network Interface.
- Crear el registro de Route 53 desde la consola.
- Usar un servicio de descubrimiento como AWS Cloud Map en una evolución posterior.

El mecanismo elegido debe actualizar el registro cuando cambie la instancia.

### Validación

Desde **EC2 �?' Instances �?' Connect �?' EC2 Instance Connect** en la EC2 frontend:

```bash
curl http://backend.internal.example:5000/health
```

Debe devolver una respuesta exitosa. Si falla, se revisarán DNS privado, rutas y `backend-sg`.

## Fase 7: Crear el Application Load Balancer

### Recursos

En la consola de AWS se abrirá **EC2 �?' Load Balancers �?' Create Application Load Balancer** y se configurará:

1. Application Load Balancer público.
2. Listener HTTP en el puerto `80`.
3. Target group para el frontend.
4. Registro del frontend como target.
5. Health check.

#### 7.1 Crear el target group

1. En **EC2 �?' Target Groups**, seleccionar **Create target group**.
2. En **Choose a target type**, elegir **Instances**.
3. En **Target group name**, escribir `cicd-lab-frontend-tg`.
4. En **Protocol**, elegir `HTTP`.
5. En **Port**, escribir `80`.
6. En **VPC**, seleccionar `cicd-lab-vpc`.
7. En **Health checks**, mantener `HTTP` y ruta `/`.
8. Seleccionar **Next**.
9. Marcar `cicd-lab-frontend`.
10. Seleccionar **Include as pending below**.
11. Seleccionar **Create target group**.

#### 7.2 Crear el Application Load Balancer

1. En **EC2 �?' Load Balancers**, seleccionar **Create Load Balancer**.
2. Elegir **Application Load Balancer**.
3. En **Load balancer name**, escribir `cicd-lab-alb`.
4. En **Scheme**, elegir **Internet-facing**.
5. En **IP address type**, elegir `IPv4`.
6. En **VPC**, seleccionar `cicd-lab-vpc`.
7. En **Availability Zones and subnets**, marcar `us-east-1a`.
8. Después de marcar la zona, en el selector de subnet que aparece debajo elegir `cicd-public-a`.
9. Marcar `us-east-1b`.
10. Después de marcar la zona, en el selector de subnet que aparece debajo elegir `cicd-public-b`.
11. Confirmar que hay exactamente dos zonas seleccionadas y que ambas subnets pertenecen a `cicd-lab-vpc`.
12. En **Security groups**, seleccionar `alb-sg`.
13. En **Listeners and routing**, dejar listener `HTTP : 80`.
14. En **Default action**, seleccionar `cicd-lab-frontend-tg`.
15. Revisar la configuración.
16. Seleccionar **Create load balancer**.

La consola actual muestra primero solo las casillas de Availability Zones. Los campos de subnet aparecen después de seleccionar cada casilla; no debes buscar una lista global llamada **Mappings**.

#### 7.3 Registrar y verificar el target correcto

Este Target Group es para el frontend. No registres `cicd-lab-backend` en este grupo.

1. Abrir **EC2 -> Target Groups**.
2. Seleccionar `cicd-lab-frontend-tg`.
3. Abrir la pestaña **Targets**.
4. Si aparece `cicd-lab-backend`, marcarlo y seleccionar **Deregister**.
5. Seleccionar **Register targets**.
6. Marcar la instancia `cicd-lab-frontend`.
7. En **Ports**, dejar `80`.
8. Seleccionar **Include as pending below**.
9. Seleccionar **Register pending targets**.
10. Esperar hasta que el estado de `cicd-lab-frontend` sea **healthy**.

Si el target anterior `cicd-lab-backend` aparece como **Draining**, no hagas nada: AWS está terminando su desregistro. Espera a que desaparezca de la lista.

Si `cicd-lab-frontend` aparece como **Unhealthy** en este punto, es normal si la instancia todavía no tiene Docker/Nginx o el frontend ejecutándose. No lo retires del Target Group; primero configura la instancia frontend y después selecciona el botón de actualizar en la pestaña **Targets**.

La configuracion del Target Group debe ser:

```text
Target type: Instances
Target: cicd-lab-frontend
Protocol: HTTP
Port: 80
Health check path: /
```

Si `cicd-lab-frontend` aparece como **unhealthy**:

1. Confirmar que la instancia este en estado `Running`.
2. Confirmar que el contenedor frontend o Nginx este escuchando en el puerto `80`.
3. Confirmar que `frontend-sg` permita HTTP `80` desde el ID de `alb-sg`.
4. Confirmar que la instancia frontend este en la VPC `cicd-lab-vpc`.
5. Revisar que el Target Group use el puerto `80` y la ruta `/`.
## Fase 8: Configurar Docker Hub y GitHub Actions

### Secrets de GitHub

Se crearán como mínimo:

```text
DOCKERHUB_USERNAME
DOCKERHUB_TOKEN
AWS_REGION
EC2_DEPLOY_HOST o identificador de instancia
SSH_PRIVATE_KEY, si se usa SSH
```

Si se utiliza AWS Systems Manager, se evitará almacenar `SSH_PRIVATE_KEY`.

### CI

El workflow de CI deberá:

1. Hacer checkout en el job que lo necesite.
2. Ejecutar pruebas del backend.
3. Construir frontend y backend.
4. Etiquetar las imágenes con el commit o una versión.
5. Publicar las imágenes en Docker Hub.

La imagen de PostgreSQL se mantendrá para desarrollo y pruebas. RDS será la base de datos del ambiente AWS.

### CD

El workflow de CD deberá:

1. Ejecutarse únicamente después de un CI exitoso.
2. Conectarse al runner o a la instancia de despliegue.
3. Descargar la versión definida de las imágenes.
4. Detener y reemplazar los contenedores correspondientes.
5. Configurar las variables de RDS.
6. Comprobar `/health`.
7. Comprobar el frontend a través del ALB.
8. Fallar explícitamente si alguna comprobación no es exitosa.

Se recomienda desplegar por tag inmutable, no depender únicamente de `latest`.

## Orden de ejecución

El orden completo será:

```text
0. Costos y seguridad de cuenta
1. Costos, seguridad y región
2. Hoja de control de recursos
3. VPC y subredes
4. Internet Gateway, NAT y rutas
5. Security Groups
6. RDS
7. EC2 backend
8. EC2 frontend
9. EC2 runner
10. DNS privado del backend
11. ALB y target group
12. Secrets y GitHub Actions
13. CI
14. CD
15. Pruebas funcionales
```

No se debe crear el ALB antes de que el frontend pueda arrancar correctamente. Tampoco se debe iniciar el CD antes de comprobar manualmente la conectividad backend-RDS.

## Pruebas de aceptación

### Red

- [ ] Las subredes públicas tienen salida por Internet Gateway.
- [ ] El backend no tiene exposición pública.
- [ ] RDS no tiene exposición pública.
- [ ] La salida de la subred privada funciona por NAT cuando es necesaria.

### Aplicación

- [ ] El target group del ALB está saludable.
- [ ] El frontend responde desde el DNS del ALB.
- [ ] `/health` devuelve estado exitoso.
- [ ] `/saludo` devuelve un mensaje desde RDS.
- [ ] El backend puede reconectarse después de reiniciarse.

### CI/CD

- [ ] El runner aparece disponible en GitHub.
- [ ] CI ejecuta las pruebas.
- [ ] CI publica las imágenes esperadas.
- [ ] CD descarga la versión correcta.
- [ ] CD verifica el despliegue.
- [ ] Un error de health check hace fallar el workflow.

### Seguridad

- [ ] No hay secretos en Git.
- [ ] SSH está restringido a la IP del administrador.
- [ ] RDS acepta únicamente tráfico desde `backend-sg`.
- [ ] El backend acepta únicamente tráfico desde `frontend-sg`.
- [ ] Se utiliza una cuenta IAM o rol, no credenciales del usuario raíz.

## Plan de destrucción y pausa

Para no consumir créditos cuando termine una sesión:

1. Detener o terminar las EC2.
2. Eliminar el ALB.
3. Eliminar el NAT Gateway.
4. Liberar las Elastic IP no utilizadas.
5. Detener o eliminar RDS según la necesidad de conservar datos.
6. Revisar volúmenes EBS y snapshots.
7. Consultar Cost Explorer.

La eliminación se hará desde la consola, respetando las dependencias. Antes de borrar RDS se debe confirmar que no existan datos que deban conservarse.

Orden recomendado desde la consola:

1. Detener los workflows de GitHub Actions.
2. Desregistrar el runner del repositorio.
3. Terminar las instancias EC2.
4. Eliminar el target group y el listener del ALB.
5. Eliminar el ALB.
6. Eliminar la instancia RDS o conservar un snapshot si se necesitan los datos.
7. Eliminar el DB subnet group si ya no tiene dependencias.
8. Eliminar el NAT Gateway.
9. Liberar la Elastic IP del NAT Gateway.
10. Eliminar Route 53 private hosted zones y sus registros.
11. Eliminar las route tables no principales.
12. Desasociar y eliminar el Internet Gateway.
13. Eliminar las subredes.
14. Eliminar la VPC.

La consola no permitirá eliminar un recurso mientras otro dependa de él. En ese caso se debe revisar el mensaje de dependencia y retirar primero el recurso indicado, sin forzar eliminaciones destructivas.

## Criterio de finalización

La primera iteración se considerará completa cuando:

1. Los recursos se creen desde la consola sin errores.
2. El ALB muestre un target saludable.
3. El frontend sea accesible mediante el DNS del ALB.
4. El frontend pueda llamar al backend privado.
5. El backend pueda consultar RDS.
6. GitHub Actions pueda ejecutar CI en el runner.
7. CD pueda actualizar el frontend y backend.
8. Las pruebas de aceptación pasen.
9. El presupuesto y las alertas de costos estén activos.
