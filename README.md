# Proyecto Innovatech Chile - Evaluación Parcial N°3

**Autores:** Claudio Alejandro Salcedo Arenas
**Asignatura:** Introducción a Herramientas DevOps (ISY1101) - Duoc UC  

## 1. Introducción y Arquitectura del Sistema
Este repositorio contiene la implementación productiva de los servicios de Innovatech Chile. La arquitectura se basa en un patrón de microservicios contenerizados y orquestados en la nube utilizando AWS Elastic Container Service (ECS).

La infraestructura completa se compone de:
* **Frontend:** Aplicación web desarrollada en React y compilada con Vite, servida mediante Nginx en el puerto 8080.
* **Backend:** Microservicios desarrollados en Spring Boot (Ventas y Despachos) en el puerto 8080.
* **Base de Datos:** Instancia gestionada MySQL 8.0 en Amazon RDS (`cordillera_db`), alojada en una subred privada.
* **Orquestación:** Clúster AWS ECS ejecutándose bajo el proveedor de capacidad Fargate (Serverless).
* **Redes:** Application Load Balancer (ALB) público que enruta el tráfico HTTP (puerto 80) hacia el contenedor del Frontend.

## 2. Configuración de AWS y Seguridad
Para cumplir con los requerimientos de seguridad y las restricciones del entorno Learner Lab:
* **Roles IAM:** Se asignó el `LabRole` preexistente para la ejecución de tareas, permitiendo a ECS descargar imágenes desde ECR y escribir logs en CloudWatch.
* **Security Groups:** El grupo de seguridad de la instancia RDS fue modificado para permitir tráfico TCP en el puerto 3306 únicamente desde dentro de la VPC.
* **Gestión de Variables:** Las credenciales de la base de datos se inyectan dinámicamente como variables de entorno en las Task Definitions de ECS, sin exponerse en el código fuente.

## 3. Pipeline CI/CD con GitHub Actions
Se implementó un flujo de integración y despliegue continuo (CI/CD) completamente automatizado que se dispara ante cualquier evento `push` en la rama `main`.

**Etapas del pipeline:**
1.  **Checkout:** Extracción del código fuente.
2.  **Configuración de AWS:** Autenticación segura utilizando GitHub Secrets (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`).
3.  **Amazon ECR:** Autenticación en el registro privado de contenedores.
4.  **Build & Push:** Construcción de la imagen Docker etiquetada con el SHA del commit y `latest`. (Para el Frontend, se inyecta la URL de la API mediante `--build-arg VITE_API_URL`).
5.  **Deploy ECS:** Descarga de la Task Definition actual, actualización con la nueva URI de la imagen y despliegue forzado en ECS.

## 4. Autoscaling y Monitoreo
Se configuraron políticas de ECS Service Auto Scaling utilizando Target Tracking para garantizar la resiliencia:
* **Métrica objetivo:** Uso de CPU (`ECSServiceAverageCPUUtilization`).
* **Umbral:** 50%.
* **Capacidad:** Mínimo 1 tarea, máximo 3 tareas.
* **Justificación:** El umbral del 50% otorga al sistema un margen de reacción adecuado frente a incrementos de tráfico, considerando que Fargate requiere tiempo para provisionar contenedores antes de degradar el servicio.

## 5. Troubleshooting y Resolución de Problemas
Durante la implementación se mitigaron los siguientes incidentes:
1.  **Error en creación de clúster (InvalidRequest):** Por limitaciones del Learner Lab, se provisionó el clúster `innovatech-cluster` mediante AWS CLI.
2.  **Crash Loop en despliegue de Backend:** Se ajustó la configuración para utilizar la variable dinámica `server.port=${PORT:8080}` y se optimizaron los tiempos de gracia del Health Check.
3.  **Pérdida de conectividad SPA (404):** Se configuró el `nginx.conf` del Frontend incorporando la directiva `try_files $uri $uri/ /index.html` para el enrutamiento correcto de React.
