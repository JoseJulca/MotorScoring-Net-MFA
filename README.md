# Motor de Scoring Crediticio

## 1. Descripción

Solución del **Motor de Scoring Crediticio** compuesta por una
aplicación Web, un API funcional de scoring y un servicio centralizado
de identidad.

La solución permite:

-   Registrar solicitudes de crédito.
-   Evaluar solicitudes mediante el Motor de Scoring.
-   Consultar solicitudes.
-   Autenticarse mediante usuario y contraseña.
-   Autenticarse mediante Google.
-   Autenticarse mediante GitHub.
-   Utilizar MFA/TOTP de forma opcional por usuario.
-   Autorizar operaciones mediante roles, claims y permisos.
-   Emitir JWT y Refresh Tokens desde el servicio de Identity.

La autenticación y autorización no se implementan en el Front. El
proyecto **MotorScoring.Identity** centraliza esta responsabilidad.

------------------------------------------------------------------------

## 2. Arquitectura general

``` text
                         ┌──────────────┐
                         │   Usuario    │
                         └──────┬───────┘
                                │
                                ▼
                    ┌──────────────────────┐
                    │  MotorScoring.Web    │
                    │        :8081         │
                    │ ASP.NET Core MVC 8   │
                    └──────────┬───────────┘
                               │
                               ▼
                 ┌────────────────────────────┐
                 │ MotorScoring.Identity      │
                 │          :8082             │
                 │ Arquitectura Hexagonal     │
                 │ ASP.NET Core Identity      │
                 └────────────┬───────────────┘
                              │
               ┌──────────────┼──────────────┐
               │              │              │
               ▼              ▼              ▼
         Usuario/Clave      Google         GitHub
               │              │              │
               └──────────────┼──────────────┘
                              │
                       AspNetUser local
                              │
                    ┌─────────┴─────────┐
                    │                   │
               Roles / Claims        MFA TOTP
                    │                   │
                    └─────────┬─────────┘
                              │
                      JWT / Refresh Token
                              │
                              ▼
                    ┌──────────────────────┐
                    │ MotorScoring.Web     │
                    └──────────┬───────────┘
                               │ Bearer JWT
                               ▼
                    ┌──────────────────────┐
                    │ MotorScoring.Api     │
                    │        :8080         │
                    │ Arquitectura         │
                    │ Hexagonal            │
                    └──────────┬───────────┘
                               │
                      Roles / Permisos
                               │
                 ┌─────────────┴─────────────┐
                 ▼                           ▼
        Registrar Solicitud          Evaluar Solicitud
```

------------------------------------------------------------------------

## 3. Proyectos

### MotorScoring.Web

Aplicación ASP.NET Core MVC que contiene la interfaz de usuario.

Responsabilidades principales:

-   Pantalla de Login.
-   Inicio de autenticación local.
-   Inicio de autenticación Google/GitHub.
-   Configuración de MFA.
-   Visualización del QR para MFA.
-   Captura del código TOTP.
-   Conservación de la sesión Web.
-   Consumo de MotorScoring.Api utilizando el JWT emitido por Identity.

El Web **no valida directamente contraseñas, roles ni permisos**.

### MotorScoring.Identity

Servicio de identidad implementado con **ASP.NET Core Identity** y
organizado con arquitectura Hexagonal.

Responsabilidades:

-   Usuarios locales.
-   Password Hash.
-   Login local.
-   Google OAuth.
-   GitHub OAuth.
-   Vinculación de logins externos.
-   Roles.
-   Claims.
-   Permisos.
-   MFA/TOTP.
-   Recovery Codes.
-   JWT.
-   Refresh Tokens.
-   Bloqueo de cuentas según la configuración de Identity.

### MotorScoring.Api

API funcional del Motor de Scoring.

Responsabilidades:

-   Registro de solicitudes.
-   Evaluación de scoring.
-   Consulta de solicitudes.
-   Validación del JWT.
-   Autorización mediante permisos.

El API no autentica contraseñas ni implementa Google/GitHub.

------------------------------------------------------------------------

## 4. Puertos

  Componente                Puerto
  ----------------------- --------
  MotorScoring.Api            8080
  MotorScoring.Web            8081
  MotorScoring.Identity       8082
  SQL Server                  1433

Accesos locales:

``` text
Web:              http://localhost:8081
API Swagger:      http://localhost:8080/swagger
Identity Swagger: http://localhost:8082/swagger
```

------------------------------------------------------------------------

## 5. Base de datos

SQL Server se ejecuta dentro de Docker.

Se utilizan dos bases:

``` text
MotorScoring
MotorScoringIdentity
```

`MotorScoringIdentity` contiene las tablas de ASP.NET Core Identity,
entre ellas:

``` text
AspNetUsers
AspNetRoles
AspNetUserRoles
AspNetUserClaims
AspNetRoleClaims
AspNetUserLogins
AspNetUserTokens
RefreshTokens
```

Los datos son persistidos mediante el volumen configurado en Docker
Compose.

> `docker compose down -v` elimina los volúmenes. Esto elimina las bases
> persistidas y, por tanto, usuarios, vinculaciones externas, roles
> asignados posteriormente y configuración MFA.

------------------------------------------------------------------------

## 6. Usuario administrador inicial

El usuario administrador se crea mediante el Seeder de Identity.

Configuración utilizada actualmente:

``` yaml
Seed__AdminEmail: "admin@motorscoring.local"
Seed__AdminPassword: "Admin1234*"
```

Usuario:

``` text
admin@motorscoring.local
```

El usuario inicial tiene el rol:

``` text
Administrador
```

------------------------------------------------------------------------

## 7. Roles

Actualmente existen:

``` text
Administrador
Analista
```

Los permisos funcionales configurados son:

``` text
Scoring.Solicitud.Crear
Scoring.Evaluacion.Ejecutar
```

Estos permisos están almacenados como claims asociados a los roles.

La autenticación y la autorización son procesos diferentes. Un usuario
puede autenticarse correctamente mediante Google o GitHub y aun así
recibir `403 Forbidden` si no tiene un rol con los permisos requeridos.

------------------------------------------------------------------------

## 8. Login local

Flujo:

``` text
Web
 ↓
Correo + contraseña
 ↓
Identity
 ↓
ASP.NET Core Identity
 ↓
AspNetUsers
 ↓
Validación PasswordHash
 ↓
Roles / Claims
 ↓
MFA si TwoFactorEnabled = true
 ↓
JWT + Refresh Token
 ↓
Web
```

El usuario local fue validado correctamente con:

-   Login.
-   MFA.
-   Registro de solicitud.
-   Evaluación de solicitud.

------------------------------------------------------------------------

## 9. Login con Google

Google actúa como proveedor externo de autenticación.

Flujo:

``` text
Web
 ↓
Identity
 ↓
Google
 ↓
Callback Identity
 ↓
Identificación / vinculación AspNetUser
 ↓
Roles / Claims
 ↓
MFA si está habilitado
 ↓
JWT
 ↓
Web
```

La autenticación con Google fue validada correctamente.

Un usuario nuevo creado mediante Google no recibe automáticamente
permisos funcionales si no tiene un rol asignado.

------------------------------------------------------------------------

## 10. Login con GitHub

GitHub funciona como segundo proveedor externo.

Flujo:

``` text
Web
 ↓
Identity
 ↓
GitHub
 ↓
Callback Identity
 ↓
Identificación / vinculación AspNetUser
 ↓
Roles / Claims
 ↓
MFA si está habilitado
 ↓
JWT
 ↓
Web
```

La autenticación con GitHub fue validada correctamente.

------------------------------------------------------------------------

## 11. Google y GitHub con el mismo usuario

Cuando Google y GitHub corresponden al mismo usuario reconocido por
Identity, ambos logins externos quedan asociados al mismo `AspNetUser`.

Conceptualmente:

``` text
AspNetUsers
└── usuario@gmail.com
      │
      ├── AspNetUserLogins
      │     ├── Google
      │     └── GitHub
      │
      ├── Roles
      ├── Claims
      └── TwoFactorEnabled
```

Esto permite compartir:

-   Roles.
-   Permisos.
-   MFA.
-   Recovery Codes.
-   Estado de la cuenta.

Este comportamiento fue validado al autenticarse con GitHub utilizando
el mismo usuario previamente utilizado con Google.

------------------------------------------------------------------------

## 12. MFA

MFA es **a demanda por usuario**.

No es obligatorio globalmente.

``` text
TwoFactorEnabled = false
        ↓
No se solicita MFA

TwoFactorEnabled = true
        ↓
Se solicita MFA
```

Esto aplica independientemente del mecanismo de autenticación utilizado.

Por ejemplo:

``` text
Usuario/Password → mismo AspNetUser → MFA
Google           → mismo AspNetUser → MFA
GitHub           → mismo AspNetUser → MFA
```

Si Google y GitHub están vinculados al mismo `AspNetUser`, activar MFA
una sola vez afecta a ambos métodos de acceso.

Este comportamiento fue validado: después de activar MFA utilizando
Google, el posterior login mediante GitHub solicitó el código MFA.

------------------------------------------------------------------------

## 13. Configuración MFA mediante QR

La pantalla Web de configuración utiliza el `AuthenticatorUri` generado
por Identity para construir un QR.

La URI TOTP **no se muestra al usuario**.

Flujo:

``` text
Identity
 ↓
Secret MFA
 ↓
AuthenticatorUri
 ↓
MotorScoring.Web
 ↓
Generación QR
 ↓
Authenticator
 ↓
Código TOTP
 ↓
Identity valida código
 ↓
TwoFactorEnabled = true
```

La pantalla presenta:

1.  QR para escanear.
2.  Campo para ingresar el código de seis dígitos.
3.  Opción para mostrar la clave secreta cuando no se pueda escanear el
    QR.

La clave secreta permanece oculta inicialmente y solo funciona como
alternativa de configuración manual.

------------------------------------------------------------------------

## 14. Recovery Codes

Al habilitar MFA, ASP.NET Core Identity puede generar códigos de
recuperación.

Estos permiten recuperar acceso cuando no se dispone del Authenticator.

Los códigos deben tratarse como información sensible y cada código de
recuperación debe utilizarse una sola vez.

------------------------------------------------------------------------

## 15. JWT y autorización

Identity emite el JWT después de completar correctamente el proceso de
autenticación y, cuando corresponde, MFA.

El Web utiliza ese token para consumir MotorScoring.Api:

``` text
Authorization: Bearer <JWT>
```

MotorScoring.Api valida el token y posteriormente los permisos
requeridos por la operación.

Ejemplo:

``` text
POST solicitud
 ↓
JWT válido
 ↓
Scoring.Solicitud.Crear
 ↓
Permitido / 403
```

Un `401 Unauthorized` corresponde a un problema de autenticación/token.

Un `403 Forbidden` significa que el usuario está autenticado, pero no
posee el permiso requerido.

------------------------------------------------------------------------

## 16. Configuración Google

Identity requiere las credenciales OAuth de Google.

Ejemplo de variables de entorno:

``` yaml
Authentication__Google__ClientId: "CLIENT_ID"
Authentication__Google__ClientSecret: "CLIENT_SECRET"
```

Callback utilizado en desarrollo:

``` text
http://localhost:8082/signin-google
```

Las credenciales reales no deben almacenarse en el repositorio.

------------------------------------------------------------------------

## 17. Configuración GitHub

Identity requiere las credenciales OAuth de GitHub.

Ejemplo:

``` yaml
Authentication__GitHub__ClientId: "CLIENT_ID"
Authentication__GitHub__ClientSecret: "CLIENT_SECRET"
```

Callback utilizado en desarrollo:

``` text
http://localhost:8082/signin-github
```

Las credenciales reales no deben almacenarse en el repositorio.

------------------------------------------------------------------------

## 18. Levantar la solución con Docker

Desde la carpeta que contiene `docker-compose.yml`:

``` powershell
docker compose up --build
```

Para consultar el estado:

``` powershell
docker compose ps
```

Los servicios esperados son:

``` text
motor-scoring-database
motor-scoring-api
motor-scoring-identity
motor-scoring-web
```

Para detener:

``` powershell
docker compose down
```

### No utilizar `-v` si se desea conservar información

``` powershell
docker compose down -v
```

también elimina los volúmenes.

En desarrollo esto implica perder, entre otros:

-   Usuarios.
-   Configuración MFA.
-   Authenticator Keys.
-   Recovery Codes.
-   Vinculaciones Google/GitHub.
-   Asignaciones de roles realizadas después de inicializar la BD.
-   Datos persistidos del Motor de Scoring.

------------------------------------------------------------------------

## 19. Reconstruir un solo servicio

Si solo cambia Identity:

``` powershell
docker compose build --no-cache motor-scoring-identity
docker compose up -d motor-scoring-identity
```

Si solo cambia el Web:

``` powershell
docker compose build --no-cache motor-scoring-web
docker compose up -d motor-scoring-web
```

No es necesario eliminar el volumen para cambios de código.

------------------------------------------------------------------------

## 20. Validaciones realizadas

El estado actual fue probado con:

``` text
Login usuario local                         OK
Login Google                                OK
Login GitHub                                OK
MFA usuario local                           OK
MFA Google                                  OK
MFA GitHub                                  OK
Google/GitHub sobre mismo usuario           OK
JWT hacia MotorScoring.Api                  OK
Autorización mediante permisos              OK
Registrar solicitud                         OK
Evaluar solicitud                           OK
```

También se comprobó que un usuario autenticado mediante Google sin rol
recibe correctamente `403 Forbidden` al intentar ejecutar una operación
protegida.

------------------------------------------------------------------------

## 21. Validación de usuarios externos

Para revisar los usuarios:

``` sql
USE MotorScoringIdentity;
GO

SELECT
    Id,
    Email,
    DisplayName,
    TwoFactorEnabled
FROM AspNetUsers;
```

Para comprobar Google/GitHub:

``` sql
SELECT
    U.Email,
    U.DisplayName,
    L.LoginProvider,
    L.ProviderKey
FROM AspNetUserLogins L
INNER JOIN AspNetUsers U
    ON U.Id = L.UserId
ORDER BY U.Email, L.LoginProvider;
```

Para comprobar roles:

``` sql
SELECT
    U.Email,
    U.DisplayName,
    R.Name AS Rol
FROM AspNetUsers U
LEFT JOIN AspNetUserRoles UR
    ON U.Id = UR.UserId
LEFT JOIN AspNetRoles R
    ON R.Id = UR.RoleId
ORDER BY U.Email;
```

------------------------------------------------------------------------

## 22. Asignar posteriormente el rol Analista a un usuario

Esta sección se deja al final para utilizarla durante una demostración o
validación.

El objetivo es demostrar que **autenticación no implica autorización**.

Un usuario puede:

``` text
Google/GitHub
     ↓
Autenticarse correctamente
     ↓
Sin rol
     ↓
API devuelve 403
```

Luego se le asigna `Analista`:

``` text
Usuario autenticado
     ↓
Rol Analista
     ↓
Claims del rol
     ↓
Scoring.Solicitud.Crear
Scoring.Evaluacion.Ejecutar
```

### Opción SQL

Reemplazar el correo por el usuario que se desea autorizar:

``` sql
USE MotorScoringIdentity;
GO

DECLARE @Email NVARCHAR(256) = 'usuario@gmail.com';

INSERT INTO AspNetUserRoles (UserId, RoleId)
SELECT U.Id, R.Id
FROM AspNetUsers U
CROSS JOIN AspNetRoles R
WHERE U.Email = @Email
  AND R.Name = 'Analista'
  AND NOT EXISTS
  (
      SELECT 1
      FROM AspNetUserRoles UR
      WHERE UR.UserId = U.Id
        AND UR.RoleId = R.Id
  );
GO
```

Validar la asignación:

``` sql
SELECT
    U.Email,
    R.Name AS Rol
FROM AspNetUsers U
LEFT JOIN AspNetUserRoles UR
    ON U.Id = UR.UserId
LEFT JOIN AspNetRoles R
    ON R.Id = UR.RoleId
WHERE U.Email = 'usuario@gmail.com';
GO
```

El resultado esperado es:

``` text
usuario@gmail.com    Analista
```

### Ejecutarlo desde Docker

Si no se dispone de un cliente SQL gráfico:

``` powershell
docker exec -it motor-scoring-database /opt/mssql-tools18/bin/sqlcmd `
-S localhost `
-U sa `
-P "Your_password123!" `
-C `
-d MotorScoringIdentity `
-Q "INSERT INTO AspNetUserRoles (UserId, RoleId) SELECT U.Id, R.Id FROM AspNetUsers U CROSS JOIN AspNetRoles R WHERE U.Email='usuario@gmail.com' AND R.Name='Analista' AND NOT EXISTS (SELECT 1 FROM AspNetUserRoles UR WHERE UR.UserId=U.Id AND UR.RoleId=R.Id)"
```

Después de asignar el rol, el usuario debe **cerrar sesión y
autenticarse nuevamente** para obtener un nuevo JWT con los
claims/permisos correspondientes.

La demostración completa puede realizarse así:

``` text
1. Login Google/GitHub
2. Intentar Registrar
3. 403 Forbidden
4. Asignar rol Analista
5. Cerrar sesión
6. Login nuevamente
7. Registrar
8. Operación permitida
9. Evaluar
10. Operación permitida
```

Esto permite demostrar claramente la separación entre **autenticación**
y **autorización**.
