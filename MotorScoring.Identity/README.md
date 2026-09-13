# MotorScoring.Identity - Arquitectura Hexagonal

Migración del servicio Identity a arquitectura Hexagonal sin cambiar los contratos HTTP, rutas, puerto 8082 ni la integración con el `docker-compose` existente.

## Módulos

- `MotorScoring.Identity.Domain`: modelos y constantes del dominio de identidad.
- `MotorScoring.Identity.Application`: casos de uso y puertos de entrada/salida.
- `MotorScoring.Identity.Adapters.Inbound.Api`: controllers HTTP con las mismas rutas y contratos existentes.
- `MotorScoring.Identity.Adapters.Outbound.Identity`: ASP.NET Core Identity, EF Core, SQL Server, JWT, refresh tokens y proveedores Google/GitHub.
- `MotorScoring.Identity.Api`: host/composition root.

## Compatibilidad

Se conservan las rutas existentes bajo `/api/auth` y `/api/users`, los modelos JSON de entrada/salida, el puerto `8082`, el nombre final `MotorScoring.Identity.dll`, el `Dockerfile` en la raíz de `MotorScoring.Identity` y el contexto de build usado por Docker Compose.
