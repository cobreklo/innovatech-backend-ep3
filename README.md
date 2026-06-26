# 🔧 Innovatech Chile — Backend (APIs Despachos & Ventas)

## Descripción General

Este repositorio contiene el código fuente y la configuración de contenedorización del **Backend** del sistema Innovatech Chile (Grupo Cordillera). Incluye dos microservicios REST desarrollados en **Spring Boot (Java 17)**: la API de Despachos y la API de Ventas. Ambos servicios se despliegan automáticamente en una instancia EC2 privada en AWS mediante un pipeline CI/CD con GitHub Actions y Amazon ECR, con persistencia de datos garantizada por volúmenes Docker.

---

## Arquitectura del Sistema

```
[EC2 Frontend – Subred Pública]  ──HTTP──►  [EC2 Backend – Subred Privada]
                                               │
                                    ┌──────────┴──────────┐
                                    │                     │
                             [api-despachos]        [api-ventas]
                             Spring Boot :8080      Spring Boot :8081
                                    │                     │
                                    └──────────┬──────────┘
                                               │
                                        [db_cordillera]
                                         MySQL 8.0 :3306
                                               │
                                     [Volume: db_data]
                                   (Named Volume – Persistente)
```

---

## Contenedorización

### Dockerfile (Multi-Stage Build)

Cada microservicio (Despachos y Ventas) posee su propio Dockerfile con **multi-stage build**:

**Etapa 1 — Build:** Utiliza `maven:3.9-eclipse-temurin-17-alpine` para compilar el proyecto con `mvn clean package -DskipTests`, generando el archivo `.jar`.

**Etapa 2 — Producción:** Utiliza `eclipse-temurin:17-jre-alpine` (imagen JRE mínima). Se crea un usuario de sistema sin privilegios (`cordillerauser`) y se cambia al mismo con la instrucción `USER`, cumpliendo el principio de mínimo privilegio. Solo el `.jar` compilado se copia a la imagen final.

**Beneficios del diseño:**
- Imagen final liviana: no incluye Maven ni JDK completo
- Capas limpias: separación total entre build y runtime
- Sin ejecución como root → seguridad reforzada

### docker-compose.yml

El archivo `docker-compose.yml` orquesta tres servicios interconectados:

| Servicio | Imagen | Puerto | Dependencias |
|---|---|---|---|
| `db` | mysql:8.0 | 3306:3306 | — |
| `api-ventas` | ECR/back-ventas:latest | 8080:8080 | `db` |
| `api-despachos` | ECR/back-despachos:latest | 8081:8080 | `db` |

### Persistencia de Datos — Named Volume

Se implementa un **Named Volume** (`db_data`) en lugar de un Bind Mount por las siguientes razones técnicas:

- **Portabilidad:** El named volume es gestionado íntegramente por Docker, no depende de rutas del host
- **Seguridad:** Docker controla los permisos del volumen, evitando accesos no autorizados desde el sistema de archivos del host
- **Resiliencia:** Al reiniciar o reemplazar el contenedor `db`, los datos de MySQL en `/var/lib/mysql` se conservan íntegramente

---

## Cómo Ejecutar Localmente

### Prerrequisitos

- Docker Desktop instalado
- Docker Compose instalado
- Git
- Credenciales AWS configuradas (para pull desde ECR)

### Pasos

```bash
# 1. Clonar el repositorio
git clone https://github.com/1Fnx/innovatech-backend.git
cd innovatech-backend

# 2. Autenticarse en Amazon ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  960261614846.dkr.ecr.us-east-1.amazonaws.com

# 3. Construir y levantar los servicios
docker compose up --build -d

# 4. Verificar estado de contenedores
docker ps

# 5. Probar endpoints
# API Despachos: http://localhost:8080/api/...
# API Ventas:    http://localhost:8081/api/...
```

---

## Pipeline CI/CD (GitHub Actions)

El pipeline se activa automáticamente al realizar un `push` sobre la rama **`deploy`**.

### Flujo Completo: Build → Push → Deploy

```
Push en rama deploy
        │
        ▼
[1] Checkout del código
        │
        ▼
[2] Configurar credenciales AWS (GitHub Secrets)
        │
        ▼
[3] Login a Amazon ECR
        │
        ▼
[4] Build y Push back-ventas → ECR
        │
        ▼
[5] Build y Push back-despachos → ECR
        │
        ▼
[6] SSH a EC2 Backend
        │
        ▼
[7] Generar docker-compose.yml en EC2
        │
        ▼
[8] docker compose pull + docker compose up -d
        │
        ▼
✅ Backend actualizado en producción (datos persistidos)
```

### GitHub Secrets Requeridos

| Secret | Descripción |
|---|---|
| `AWS_ACCESS_KEY_ID` | Clave de acceso AWS Academy |
| `AWS_SECRET_ACCESS_KEY` | Clave secreta AWS |
| `AWS_SESSION_TOKEN` | Token de sesión (Lab) |
| `EC2_HOST` | IP privada de la instancia Backend |
| `EC2_SSH_KEY` | Llave privada SSH para acceso a EC2 |

---

## Estructura del Repositorio

```
innovatech-backend/
├── .github/
│   └── workflows/
│       └── deploy.yml               # Pipeline GitHub Actions
├── back-Despachos_SpringBoot/
│   └── Springboot-API-REST-DESPACHO/
│       ├── src/
│       ├── Dockerfile               # Multi-stage build
│       └── pom.xml
├── back-Ventas_SpringBoot/
│   └── Springboot-API-REST/
│       ├── src/
│       ├── Dockerfile               # Multi-stage build
│       └── pom.xml
├── docker-compose.yml               # Orquestación completa (db + 2 APIs)
└── README.md
```

---

## Registros ECR (Amazon Elastic Container Registry)

| Repositorio | URI |
|---|---|
| `back-despachos` | `960261614846.dkr.ecr.us-east-1.amazonaws.com/back-despachos` |
| `back-ventas` | `960261614846.dkr.ecr.us-east-1.amazonaws.com/back-ventas` |

---

## Principios DevOps Aplicados

- **Contenedorización:** Multi-stage Dockerfile con usuario no root (`cordillerauser`)
- **Persistencia:** Named Volume `db_data` garantiza continuidad operativa tras reinicios
- **Automatización CI/CD:** Pipeline construye y despliega ambas APIs en un solo workflow
- **Gestión de Secretos:** Credenciales AWS y SSH almacenadas como GitHub Secrets
- **Registro de Imágenes:** Amazon ECR (privado y seguro, integrado con IAM de AWS)
- **Control de Versiones:** Historial de commits descriptivos con mensajes semánticos

---

## Equipo de Desarrollo

**Grupo Cordillera — Innovatech Chile**  
Asignatura: ISY1101 — Introducción a Herramientas DevOps  
Evaluación Parcial N°2 | 2026
