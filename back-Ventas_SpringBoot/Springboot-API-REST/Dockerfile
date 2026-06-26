# Etapa 1: Construcción (Build)
FROM maven:3.9-eclipse-temurin-17-alpine AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
# Compilamos el proyecto omitiendo los tests para que sea más rápido
RUN mvn clean package -DskipTests

# Etapa 2: Producción (Run)
FROM eclipse-temurin:17-jre-alpine
# Cumplimiento de rúbrica: Crear usuario no root por seguridad
RUN addgroup -S cordilleragroup && adduser -S cordillerauser -G cordilleragroup
WORKDIR /app
# Copiamos solo el .jar generado en la etapa 1 (limpieza de capas)
COPY --from=builder /app/target/*.jar app.jar
# Cambiamos al usuario sin privilegios
USER cordillerauser
# Exponemos el puerto
EXPOSE 8080 
# Comando de inicio
CMD ["java", "-jar", "app.jar"]