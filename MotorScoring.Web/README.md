# MotorScoring.Web.Java

Frontend independiente desarrollado con **.NET 8 + ASP.NET Core MVC + Razor + JavaScript**
para consumir el Motor de Scoring Crediticio implementado en **Java 21 + Spring Boot + Arquitectura Onion**.

## Compatibilidad con el backend Java

Este frontend utiliza exactamente las rutas del proyecto Java:

```http
POST /api/solicitudes-credito
POST /api/solicitudes-credito/{id}/evaluar
```

No utiliza `/api/v1`.

Los identificadores recibidos desde Java se modelan como `long`:

- `idSolicitud`
- `idSolicitante`
- `idEvaluacion`

Las fechas del backend Java (`LocalDateTime`) se modelan como `DateTime`.

## Flujo de dos pasos

1. El usuario completa el formulario.
2. `Registrar solicitud` consume `POST /api/solicitudes-credito`.
3. Si responde `201 Created`, se conserva internamente el `idSolicitud` numérico y
   se mantiene visible el mismo `identificadorExterno` utilizado.
4. Los campos quedan bloqueados y se habilita `Evaluar solicitud`.
5. `Evaluar solicitud` consume
   `POST /api/solicitudes-credito/{idSolicitud}/evaluar`.
6. Si la evaluación falla, NO se vuelve a registrar la solicitud.

## Pantallas

- Registrar solicitud de crédito.
- Resultado de evaluación.

La interfaz no muestra los identificadores técnicos.

## Configurar URL del API Java

Editar `appsettings.json`:

```json
{
  "MotorScoringApi": {
    "BaseUrl": "http://localhost:8080"
  }
}
```

Ajustar el puerto al puerto real de Spring Boot.

## Ejecutar

```powershell
dotnet restore
dotnet build
dotnet run
```

## Diferencias frente a la versión para .NET Hexagonal

| Aspecto | Backend Java |
|---|---|
| Registro | `/api/solicitudes-credito` |
| Evaluación | `/api/solicitudes-credito/{id}/evaluar` |
| Identificadores | `Long` / `long` |
| Fechas | `LocalDateTime` / `DateTime` |
| ResultadoFactor | No requiere `resultadoExcluyente` en el contrato HTTP actual |
