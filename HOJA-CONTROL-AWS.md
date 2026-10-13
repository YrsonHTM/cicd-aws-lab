# Hoja de control del laboratorio AWS

Esta hoja registra los recursos creados manualmente desde la AWS Management Console. Se debe actualizar despuÃ©s de crear cada recurso y antes de pasar a la siguiente fase.

> No registrar aquÃ­ contraseÃ±as, tokens, claves privadas ni secretos.

## Datos generales

| Campo | Valor |
|---|---|
| Proyecto | Laboratorio CI/CD AWS |
| Cuenta AWS | Pendiente de registrar de forma segura |
| RegiÃ³n | Pendiente |
| Fecha de inicio | 2026-10-12 |
| Responsable | Pendiente |
| Presupuesto mensual | Pendiente |
| Nombre del presupuesto | `cicd-lab-monthly-budget` |
| Estado general | En progreso |

## VPC

| Recurso | Nombre | ID | CIDR | Estado |
|---|---|---|---|---|
| VPC | `cicd-lab-vpc` | `vpc-0d9f5010052bd7577` | `10.0.0.0/16` | Creada |

### ValidaciÃ³n de la VPC

- [ ] La VPC aparece en la regiÃ³n correcta.
- [ ] El nombre es `cicd-lab-vpc`.
- [ ] El CIDR es `10.0.0.0/16`.
- [ ] **DNS resolution** estÃ¡ habilitado.
- [ ] **DNS hostnames** estÃ¡ habilitado.

## Availability Zones

| Zona | Uso | Confirmada |
|---|---|---|
| AZ-a | Subredes pÃºblicas y privadas | Pendiente |
| AZ-b | Segunda subred pÃºblica y subred RDS | Pendiente |

Anotar aquÃ­ los nombres reales de las zonas seleccionadas, por ejemplo:

```text
AZ-a: ____________________
AZ-b: ____________________
```

## Subredes

| Nombre | Tipo | CIDR | Availability Zone | ID | Estado |
|---|---|---|---|---|---|
| `cicd-public-a` | PÃºblica | `10.0.1.0/24` | Pendiente | Pendiente | Pendiente |
| `cicd-public-b` | PÃºblica | `10.0.2.0/24` | Pendiente | Pendiente | Pendiente |
| `cicd-private-app-a` | Privada | `10.0.11.0/24` | Pendiente | Pendiente | Pendiente |
| `cicd-private-db-a` | Privada | `10.0.21.0/24` | Pendiente | Pendiente | Pendiente |
| `cicd-private-db-b` | Privada | `10.0.22.0/24` | Pendiente | Pendiente | Pendiente |

## Internet Gateway y NAT Gateway

| Recurso | Nombre | ID | Subred | Elastic IP | Estado |
|---|---|---|---|---|---|
| Internet Gateway | `cicd-lab-igw` | Pendiente | VPC `vpc-0d9f5010052bd7577` | No aplica | Pendiente |
| NAT Gateway | `cicd-lab-nat` | `nat-0ae0e200855643ba7` | `cicd-public-a` | Pendiente | Creado |

> El NAT Gateway genera costos mientras permanece disponible. Debe eliminarse cuando el laboratorio estÃ© detenido.

## Route Tables

| Nombre | ID | Subredes asociadas | Ruta principal | Estado |
|---|---|---|---|---|
| `cicd-public-rt` | Pendiente | `cicd-public-a`, `cicd-public-b` | `0.0.0.0/0 â†’ Internet Gateway` | Pendiente |
| `cicd-private-app-rt` | Pendiente | `cicd-private-app-a` | `0.0.0.0/0 â†’ NAT Gateway` | Pendiente |
| `cicd-private-db-rt` | Pendiente | `cicd-private-db-a`, `cicd-private-db-b` | Sin salida a Internet | Pendiente |

## Security Groups

| Nombre | ID | Asociado a | Estado |
|---|---|---|---|
| `alb-sg` | Pendiente | Application Load Balancer | Pendiente |
| `frontend-sg` | Pendiente | EC2 frontend | Pendiente |
| `backend-sg` | Pendiente | EC2 backend | Pendiente |
| `rds-sg` | Pendiente | RDS PostgreSQL | Pendiente |
| `runner-sg` | Pendiente | EC2 GitHub Runner | Pendiente |
| `admin-temporary-sg` | Pendiente | AdministraciÃ³n temporal | Pendiente |
| `eic-endpoint-sg` | Pendiente | EC2 Instance Connect Endpoint | Pendiente |
| `cicd-sg` | `sg-081b9488c14918643` | Security Group adicional | Creado |

## RDS PostgreSQL

| Campo | Valor |
|---|---|
| Identificador | `cicd-lab-postgres` |
| Engine | PostgreSQL |
| DB subnet group | Pendiente |
| Endpoint | `cicd-lab-postgres.cyx26me4q2u8.us-east-1.rds.amazonaws.com` |
| Puerto | `5432` |
| Base de datos | `saludos_db` |
| Usuario | Pendiente |
| Security Group | Pendiente |
| PÃºblicamente accesible | No |
| Estado | Pendiente |

> La contraseÃ±a no se registra en este documento.

## Instancias EC2

| Nombre | FunciÃ³n | Instance ID | Subred | IP privada | IP pÃºblica | Estado |
|---|---|---|---|---|---|---|
| `cicd-lab-frontend` | Frontend/Nginx | `i-06d286c1c44af8ebe` | `cicd-public-a` (`subnet-0962e3b0b2b537f1d`) | `10.0.1.216` | `3.227.247.164` | Running |
| `cicd-lab-backend` | API Flask | Pendiente | `cicd-private-app-a` | Pendiente | No debe tener | Pendiente |
| `cicd-lab-runner` | GitHub Actions Runner | Pendiente | `cicd-public-b` | Pendiente | Pendiente | Pendiente |

## Route 53 privado

| Campo | Valor |
|---|---|
| Hosted zone | `internal.example` |
| Hosted zone ID | Pendiente |
| VPC asociada | `vpc-0d9f5010052bd7577` |
| Registro | `backend.internal.example` |
| IP privada del backend | Pendiente |
| Estado | Pendiente |

## Application Load Balancer

| Campo | Valor |
|---|---|
| Nombre | `cicd-lab-alb` |
| DNS name | Pendiente |
| ARN | Pendiente |
| Scheme | Internet-facing |
| Subredes | `cicd-public-a`, `cicd-public-b` |
| Listener | HTTP `80` |
| Security Group | `alb-sg` |
| Estado | Pendiente |

## Target Group

| Campo | Valor |
|---|---|
| Nombre | `cicd-lab-frontend-tg` |
| Protocolo | HTTP |
| Puerto | `80` |
| Health check path | `/` |
| Target | `cicd-lab-frontend` |
| Estado del target | Pendiente |

## GitHub Actions y Docker Hub

| Elemento | Valor | Estado |
|---|---|---|
| Repositorio GitHub | Pendiente | Pendiente |
| Runner registrado | Pendiente | Pendiente |
| Docker Hub username | Pendiente | Pendiente |
| Imagen frontend | Pendiente | Pendiente |
| Imagen backend | Pendiente | Pendiente |
| Imagen database | Pendiente | Pendiente |

Los tokens y secretos se deben guardar Ãºnicamente en GitHub Secrets o en el gestor de secretos correspondiente.

## Checklist de avance

### Red

- [x] VPC creada.
- [ ] Subredes creadas.
- [ ] Internet Gateway asociado.
- [ ] NAT Gateway creado.
- [ ] Route tables configuradas.
- [ ] DNS de la VPC habilitado.

### Seguridad

- [ ] Security Groups creados.
- [ ] SSH restringido a la IP del administrador.
- [ ] RDS sin acceso pÃºblico.
- [ ] Backend sin acceso pÃºblico.

### AplicaciÃ³n

- [ ] RDS disponible.
- [ ] EC2 frontend disponible.
- [ ] EC2 backend disponible.
- [ ] EC2 runner disponible.
- [ ] Backend conectado a RDS.
- [ ] Frontend conectado al backend.
- [ ] ALB creado.
- [ ] Target group saludable.

### Costos

- [ ] Presupuesto creado.
- [ ] Alertas al 50 %, 80 % y 100 %.
- [ ] NAT Gateway eliminado al terminar la sesiÃ³n.
- [ ] EC2 detenidas o terminadas cuando no se usan.
- [ ] Cost Explorer revisado.

## Historial de cambios

| Fecha | Cambio | Responsable |
|---|---|---|
| 2026-10-12 | VPC `vpc-0d9f5010052bd7577` registrada | Pendiente |
| 2026-10-12 | NAT Gateway `nat-0ae0e200855643ba7` registrado | Pendiente |
| 2026-10-12 | Endpoint de RDS `cicd-lab-postgres.cyx26me4q2u8.us-east-1.rds.amazonaws.com` registrado | Pendiente |
| 2026-10-12 | EC2 frontend `i-06d286c1c44af8ebe` registrada | Pendiente |
