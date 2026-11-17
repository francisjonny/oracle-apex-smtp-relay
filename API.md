# API HTTPS y estados

Base de ejemplo: `https://tu-dominio/apex-mail-relay/index.php`.
Todos los endpoints requieren HTTPS y la cabecera **X-APEX-Mail-Key**.
No se admite la clave en query string ni un remitente enviado por el cliente.
Las respuestas no incluyen contraseña SMTP, destinatario, cuerpo o enlaces.

## Diagnóstico

`GET ?action=health`

```json
{"status":"ready","enabled":false}
```

Comprueba autenticación, versión/extensiones, autoload y acceso al estado SQLite.
No conecta al SMTP, no prueba su contraseña ni envía correo. `enabled=false`
permite preparar la instalación antes de autorizar envíos.

## Estado de un ID

`GET ?action=status&id=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa`

```json
{"id":"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa","state":"SENT"}
```

Un ID inexistente devuelve **404**, sin crear un registro.
Un proceso que quedó `PROCESSING` durante más de tres minutos se presenta como
`UNCERTAIN`; no vuelve a enviarse automáticamente.

## Enviar

`POST ?action=send`, `Content-Type: application/json`.

```json
{
  "id": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
  "to": "destinatario@example.com",
  "subject": "Notificación",
  "body": "Texto UTF-8 con acentos: áéíóú y ñ.",
  "expires": 2000000000
}
```

El timestamp es ilustrativo: debes generar uno futuro con vigencia de hasta
1800 segundos al recibirlo el relay. El paquete Oracle lo calcula al encolar.
Los relojes Oracle y hosting deben estar sincronizados. El ID es un hexadecimal
de 32 caracteres generado una sola vez para ese mensaje, nunca derivado de un
correo, documento personal o dato secreto.

Campos exactos: `id`, `to`, `subject`, `body`, `expires`. El orden de claves JSON
es indiferente. Campos adicionales, múltiples destinatarios, cabeceras inyectadas,
JSON inválido o un mensaje vencido se rechazan antes de enviar.

El servicio registra `PROCESSING` **antes** de intentar SMTP. Después registra
`SENT` o `UNCERTAIN`. Un nuevo POST con el mismo ID y contenido devuelve el
estado existente; no repite SMTP. El mismo ID con otro contenido devuelve **409**.
Consulta estado antes de reenviar después de un timeout.

## Códigos HTTP

| Código | Significado / acción |
|---|---|
| 200 | Solicitud procesada; examina `state`, no supongas entrega por el 200. |
| 400 | JSON, campos, ID, destinatario o cabeceras inválidos; también HTTPS ausente. |
| 401 | Clave ausente, inválida o configuración de autenticación incompleta. |
| 404 | ID no registrado en la consulta de estado. |
| 405 | Acción o método no permitido. |
| 409 | ID existente con otro contenido; no cambies el ID para eludirlo. |
| 410 | Mensaje vencido o timestamp fuera del límite admitido. |
| 413 | Payload mayor al límite configurado. |
| 415 | Content-Type distinto de application/json. |
| 429 | Límite de frecuencia; Retry-After: 60. Mantén el ID. |
| 503 | Servicio deshabilitado, dependencias, configuración o almacenamiento no disponibles. |

Un error de servidor web previo a PHP puede producir otro formato o un 500.
El cliente debe tratar respuestas desconocidas como no confirmadas y consultar
el estado del mismo ID. No guardar respuestas remotas completas en logs.

## Estados Oracle

| Estado | Interpretación |
|---|---|
| NEW | Espera un primer intento o un reintento de transporte/configuración. |
| PROCESSING | El relay comenzó el envío; el worker consultará su estado. |
| SENT | El SMTP aceptó el mensaje; falta confirmar recepción en el buzón. |
| UNCERTAIN | La aceptación SMTP no se pudo confirmar con seguridad; revisión manual. |
| FAILED | Rechazo permanente del payload/contrato. No se reintenta automáticamente. |
| EXPIRED | El relay no tenía ese ID y el plazo de primer envío terminó. |

Los estados terminales SENT, FAILED y EXPIRED borran el payload de la cola Oracle.
UNCERTAIN conserva el payload para investigación; contiene datos sensibles y debe
tener acceso y retención controlados. No se imprime en el informe de estado.

## Límites, reintentos y concurrencia

- El límite horario y diario se evalúa bajo una transacción SQLite antes del envío.
- Los duplicados se resuelven antes de consumir otra cuota.
- Dos peticiones simultáneas para un ID producen un único intento SMTP.
- Una desconexión o un reinicio después de registrar PROCESSING no dispara otro envío.
- Tras 20 intentos de consulta/transporte, el ejemplo Oracle deja el registro para
  revisión. No resetees el contador sin consultar el relay y revisar la caducidad.
- No elimines SQLite ni restaures una copia antigua sin reconciliar solicitudes.
- Si el destinatario pide otro enlace, tu aplicación debe invalidar el token anterior
  y generar una notificación nueva siguiendo sus propias reglas de seguridad.

El Message-ID SMTP tiene formato `<apex-mail-relay-ID@dominio-del-remitente>`.
Puede usarse para pedir al proveedor una revisión sin compartir claves o cuerpos.
