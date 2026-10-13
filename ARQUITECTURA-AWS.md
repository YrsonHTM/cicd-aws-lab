# Arquitectura AWS para la aplicación de saludos

Este documento describe la arquitectura AWS propuesta para desplegar la aplicación del repositorio. El diseño sigue el diagrama de referencia: una VPC con subredes públicas y privadas, un Application Load Balancer (ALB), instancias EC2 para los contenedores, una base de datos PostgreSQL en Amazon RDS y un runner de GitHub Actions.

La arquitectura está pensada para aprender redes, seguridad, contenedores y CI/CD en AWS. Para mantener el costo controlado, algunos componentes se simplifican para el laboratorio. Esas simplificaciones están identificadas explícitamente.

## Vista rápida

El flujo principal es:

```text
Usuario
  |
  v
Application Load Balancer (ALB)
  |
  v
Frontend en EC2 pública
  |
  v
Backend en EC2 privada
  |
  v
Amazon RDS PostgreSQL en subred privada
```

El flujo de despliegue es:

```text
GitHub
  |
  v
GitHub Actions
  |
  v
GitHub self-hosted runner en EC2
  |
  v
Docker Hub
  |
  v
EC2 de aplicación descarga las nuevas imágenes
```

## Componentes de AWS

| Componente | Responsabilidad | Ubicación |
|---|---|---|
| AWS Region | Región donde viven todos los recursos | Una región elegida para el laboratorio |
| VPC | Aislamiento lógico de la red | `10.0.0.0/16` recomendado |
| Availability Zones | Separación física para mejorar disponibilidad | Dos zonas para la capa pública |
| Internet Gateway | Entrada y salida de Internet para recursos públicos | Asociado a la VPC |
| Subredes públicas | ALB y, en el laboratorio, frontend/runner | Ruta hacia Internet Gateway |
| Subredes privadas | Backend y RDS | Sin ruta directa al Internet Gateway |
| Route Tables | Control del siguiente salto del tráfico | Una tabla pública y tablas privadas |
| NAT Gateway | Salida controlada de recursos privados hacia Internet | En una subred pública |
| Application Load Balancer | Punto de entrada HTTP/HTTPS y distribución del tráfico | Subredes públicas |
| EC2 frontend | Sirve la interfaz web mediante Docker/Nginx | Subred pública |
| EC2 runner | Ejecuta jobs de GitHub Actions y comandos Docker | Subred pública en el diseño de laboratorio |
| EC2 backend | Ejecuta la API Flask mediante Docker | Subred privada |
| Amazon RDS PostgreSQL | Persistencia administrada de la aplicación | Subredes privadas |
| Security Groups | Firewall con estado por recurso | Asociados a ALB, EC2 y RDS |
| Docker Hub | Registro de las imágenes publicadas por CI | Servicio externo |
| GitHub Actions | Automatización de CI/CD | Servicio externo |

## Diseño de red

### VPC

Se recomienda crear una VPC con el bloque:

```text
10.0.0.0/16
```

El bloque `/16` deja espacio para crecer sin rediseñar la red. Una distribución inicial puede ser:

| Subred | CIDR | Tipo | Zona sugerida | Recursos |
|---|---|---|---|---|
| Public-A | `10.0.1.0/24` | Pública | AZ-a | ALB, frontend o runner |
| Public-B | `10.0.2.0/24` | Pública | AZ-b | ALB y capacidad futura |
| Private-App-A | `10.0.11.0/24` | Privada | AZ-a | Backend |
| Private-Db-A | `10.0.21.0/24` | Privada | AZ-a | RDS |
| Private-Db-B | `10.0.22.0/24` | Privada | AZ-b | Segunda subred del DB subnet group |

RDS requiere normalmente un DB subnet group con subredes privadas en al menos dos Availability Zones, aunque durante el laboratorio se utilice una sola instancia de base de datos.

### Tabla de rutas pública

La tabla pública se asocia a `Public-A` y `Public-B`:

| Destino | Siguiente salto |
|---|---|
| `10.0.0.0/16` | Local |
| `0.0.0.0/0` | Internet Gateway |

Los recursos con una dirección IPv4 pública o Elastic IP pueden recibir conexiones desde Internet si su Security Group también lo permite.

### Tabla de rutas privada

La tabla privada de aplicación se asocia a `Private-App-A`:

| Destino | Siguiente salto |
|---|---|
| `10.0.0.0/16` | Local |
| `0.0.0.0/0` | NAT Gateway |

La salida por NAT permite, por ejemplo, instalar paquetes o descargar imágenes. El NAT no permite iniciar conexiones entrantes desde Internet hacia el backend.

La tabla de RDS debe ser todavía más restrictiva. RDS no necesita una ruta a Internet para aceptar conexiones desde el backend:

| Destino | Siguiente salto |
|---|---|
| `10.0.0.0/16` | Local |

## Flujo de tráfico de usuarios

1. El usuario solicita la aplicación usando el DNS del ALB.
2. El Internet Gateway entrega el tráfico al ALB en una subred pública.
3. El ALB escucha en el puerto `80` durante el laboratorio. Para un ambiente real debe utilizar `443` con un certificado de AWS Certificate Manager.
4. El ALB reenvía la solicitud al frontend en el puerto `80`.
5. Nginx sirve los archivos estáticos del frontend.
6. Las solicitudes a `/api/` son reenviadas por Nginx al backend en el puerto `5000`.
7. El backend consulta RDS PostgreSQL en el puerto `5432`.
8. La respuesta regresa por la misma cadena hasta el usuario.

### Consideración importante sobre Nginx

El archivo actual de Nginx utiliza:

```nginx
proxy_pass http://backend:5000/;
```

El nombre `backend` funciona cuando frontend y backend comparten una red Docker con ese nombre de servicio. En esta arquitectura viven en instancias EC2 diferentes, por lo que se debe reemplazar por un nombre resoluble dentro de la VPC, por ejemplo:

```text
backend.internal.example:5000
```

Las opciones para resolverlo son:

- Un registro privado de Route 53 apuntando al backend.
- AWS Cloud Map.
- La IP privada del backend, solo como solución temporal de laboratorio.
- Colocar frontend y backend en la misma instancia EC2, si se desea conservar el nombre Docker `backend`.

La opción recomendada para aprender correctamente la arquitectura mostrada es usar un nombre privado de Route 53 y mantener el puerto `5000` inaccesible desde Internet.

## Flujo de despliegue CI/CD

### Integración continua

GitHub Actions ejecuta las etapas:

```text
Checkout
  -> Tests
  -> Build de frontend, backend y database
  -> Tag de imágenes
  -> Push a Docker Hub
```

Las imágenes esperadas son:

```text
brendisas/frontend:<tag>
brendisas/backend:<tag>
brendisas/database:<tag>
```

La base de datos de producción no debería reemplazarse con un contenedor PostgreSQL cada vez que se despliega. En AWS, la imagen `database` puede conservarse para pruebas locales o entornos efímeros; el ambiente AWS debe usar RDS como persistencia principal.

### Despliegue continuo

Después de un CI exitoso, el workflow de CD debe:

1. Autenticarse contra la EC2 objetivo mediante SSH o AWS Systems Manager.
2. Descargar las nuevas imágenes del frontend y backend desde Docker Hub.
3. Detener únicamente los contenedores que serán actualizados.
4. Iniciar los contenedores con las variables de entorno de AWS.
5. Comprobar el endpoint `/health`.
6. Verificar que el frontend responda a través del ALB.
7. Reportar el error y detener el pipeline si alguna comprobación falla.

Para una primera versión del laboratorio se puede usar SSH. Como evolución recomendada, AWS Systems Manager evita abrir el puerto `22` y elimina la necesidad de administrar una clave privada en GitHub.

## Security Groups

Los Security Groups deben permitir únicamente el tráfico requerido. No se debe abrir toda la VPC con reglas amplias como `0.0.0.0/0` para todos los puertos.

### `sg-alb`

| Dirección | Protocolo/puerto | Origen o destino | Motivo |
|---|---|---|---|
| Entrada | TCP `80` | `0.0.0.0/0` | Acceso HTTP durante el laboratorio |
| Entrada | TCP `443` | `0.0.0.0/0` | HTTPS cuando se configure TLS |
| Salida | TCP `80` | `sg-frontend` | Envío al frontend |
| Salida | TCP `443` | `sg-frontend` | Envío HTTPS si se configura internamente |

### `sg-frontend`

| Dirección | Protocolo/puerto | Origen o destino | Motivo |
|---|---|---|---|
| Entrada | TCP `80` | `sg-alb` | Solo el ALB debe acceder al frontend |
| Salida | TCP `5000` | `sg-backend` | Solicitudes API mediante Nginx |
| Salida | TCP `443` | `0.0.0.0/0` | Actualizaciones o descarga de imágenes, si aplica |

### `sg-backend`

| Dirección | Protocolo/puerto | Origen o destino | Motivo |
|---|---|---|---|
| Entrada | TCP `5000` | `sg-frontend` | Tráfico de la API |
| Salida | TCP `5432` | `sg-rds` | Conexión a PostgreSQL |
| Salida | TCP `443` | NAT Gateway | Descargas necesarias |

### `sg-rds`

| Dirección | Protocolo/puerto | Origen o destino | Motivo |
|---|---|---|---|
| Entrada | TCP `5432` | `sg-backend` | Único acceso de aplicación a PostgreSQL |
| Salida | Predeterminada o restringida | Según política | Respuesta de la base de datos |

### `sg-runner`

Si el runner se mantiene en una EC2 separada:

| Dirección | Protocolo/puerto | Origen o destino | Motivo |
|---|---|---|---|
| Entrada | TCP `22` | IP pública del administrador | Administración temporal |
| Salida | TCP `443` | `0.0.0.0/0` | GitHub, Docker Hub y AWS |

El runner no debe aceptar conexiones entrantes desde Internet para ejecutar los workflows. La conexión hacia GitHub es saliente.

## Disponibilidad y límites del diseño

El diagrama utiliza dos Availability Zones para la capa pública y una instancia por componente. Eso mejora la organización de la red, pero no convierte automáticamente la solución en altamente disponible.

### Estado del laboratorio

- Un ALB distribuido en dos zonas.
- Una instancia frontend activa.
- Una instancia backend activa.
- Una instancia RDS Single-AZ.
- Un runner self-hosted.
- Un NAT Gateway compartido.

### Mejoras para producción

- Frontend en un Auto Scaling Group con al menos dos instancias.
- Backend en un Auto Scaling Group dentro de subredes privadas en dos zonas.
- RDS Multi-AZ.
- NAT Gateway por Availability Zone para evitar un único punto de salida.
- HTTPS obligatorio mediante ACM.
- Secrets Manager o Systems Manager Parameter Store para credenciales.
- ECR en lugar de Docker Hub si se desea mantener el registro dentro de AWS.
- CloudWatch Logs, métricas y alarmas.

Estas mejoras incrementan el costo y no son necesarias para la primera práctica.

## Decisión sobre el runner

El diagrama muestra el GitHub Actions runner junto al frontend en una EC2 pública. Esto es aceptable como simplificación didáctica, pero no es la separación recomendada para producción:

- El runner ejecuta código de workflows y necesita permisos elevados para Docker.
- Una vulnerabilidad en un workflow podría afectar la instancia de aplicación.
- El ciclo de vida del runner es diferente al de la aplicación.

La distribución recomendada es:

```text
EC2 pública o privada aislada: runner
EC2 pública: frontend
EC2 privada: backend
RDS privada: base de datos
```

Si el presupuesto obliga a reducir instancias, el runner puede compartir temporalmente la EC2 del frontend, siempre que se documente como una decisión exclusiva del laboratorio.

## Persistencia y configuración

RDS debe almacenar los datos de la aplicación. El backend debe obtener la conexión mediante variables seguras:

```text
DB_HOST=<endpoint privado de RDS>
DB_PORT=5432
DB_NAME=saludos_db
DB_USER=<usuario>
DB_PASSWORD=<secreto>
```

No se deben guardar contraseñas, claves SSH ni tokens en el repositorio. Para la primera iteración pueden configurarse como GitHub Secrets y variables protegidas de despliegue. La opción AWS recomendada es Secrets Manager o Parameter Store.

## Observabilidad y operación

Como mínimo se deben verificar:

- Estado del target group del ALB.
- Logs de Nginx.
- Logs del contenedor backend.
- Estado de la instancia EC2.
- Conectividad backend-RDS.
- Métricas de CPU, memoria y almacenamiento.
- Estado de la instancia RDS.
- Resultado del endpoint `/health`.

Una comprobación de despliegue mínima es:

```bash
curl http://<dns-del-alb>/health
curl http://<dns-del-alb>/api/health
```

El endpoint exacto depende de cómo se configure el `location /api/` de Nginx y de las rutas expuestas por Flask.

## Costos y controles del laboratorio

El recurso que más fácilmente puede generar un costo inesperado en esta arquitectura es el NAT Gateway, porque cobra por hora y por volumen procesado. Para controlar los créditos:

- Crear un presupuesto mensual de AWS.
- Configurar alertas al 50 %, 80 % y 100 %.
- Detener las instancias EC2 cuando no se esté practicando.
- Eliminar NAT Gateway cuando el laboratorio esté detenido.
- Eliminar Elastic IP no utilizadas.
- Eliminar snapshots, volúmenes EBS y grupos de RDS que ya no se necesiten.
- No crear un Load Balancer adicional por ambiente.
- Evitar Multi-AZ, Auto Scaling y RDS de producción durante la primera iteración.
- Revisar Cost Explorer al terminar cada sesión.

El ALB, NAT Gateway y RDS no son completamente gratuitos en todos los escenarios. Antes de crear recursos se debe revisar la oferta Free Tier vigente de la cuenta y de la región elegida.

## Checklist de implementación

- [ ] Elegir región AWS y confirmar disponibilidad de los tipos de instancia.
- [ ] Crear la VPC `10.0.0.0/16`.
- [ ] Crear dos subredes públicas en distintas Availability Zones.
- [ ] Crear subredes privadas para aplicación y RDS.
- [ ] Crear y asociar Internet Gateway.
- [ ] Crear route table pública.
- [ ] Crear NAT Gateway y route table privada.
- [ ] Crear `sg-alb`, `sg-frontend`, `sg-backend`, `sg-rds` y `sg-runner`.
- [ ] Crear DB subnet group para RDS.
- [ ] Crear RDS PostgreSQL sin exposición pública.
- [ ] Crear EC2 frontend y configurar Docker.
- [ ] Crear EC2 backend en una subred privada.
- [ ] Resolver el nombre privado del backend para Nginx.
- [ ] Crear ALB, listener, target group y health check.
- [ ] Configurar el runner de GitHub Actions.
- [ ] Configurar secretos de GitHub sin incluirlos en el código.
- [ ] Probar CI y después CD.
- [ ] Probar `/health`, `/saludo` y el acceso al frontend desde el DNS del ALB.
- [ ] Crear presupuesto y alarmas de costos antes de dejar recursos activos.

## Referencias

- [Amazon VPC documentation](https://docs.aws.amazon.com/vpc/)
- [VPC with public and private subnets](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Scenario2.html)
- [Internet gateways](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)
- [NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
- [Application Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)
- [Amazon EC2 security groups](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security-groups.html)
- [Amazon RDS subnet groups](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_VPC.WorkingWithRDSInstanceinaVPC.html)
- [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)

