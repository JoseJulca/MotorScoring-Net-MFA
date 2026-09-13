# Cambio visual MFA - QR

Cambios realizados únicamente en `MotorScoring.Web`:

- Generación server-side del QR TOTP con QRCoder.
- La URI `otpauth://` ya no se muestra en la interfaz.
- La clave secreta queda oculta por defecto y solo se muestra como alternativa manual.
- Se conserva el endpoint y flujo existente de activación MFA.
- No se modifican contratos de Identity, API de scoring ni docker-compose.
