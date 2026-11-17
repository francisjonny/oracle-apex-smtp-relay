# Diagnóstico paso a paso

No repitas un envío hasta saber si el SMTP lo aceptó. Usa primero diagnóstico,
estado e historial. No pegues cuerpos, claves, tokens ni archivos de configuración
en issues, chats o capturas.

## Separar las capas

1. **Hosting web:** URL HTTPS accesible, certificado válido y PHP web compatible.
2. **Servicio:** configuración privada, dependencias y permisos de SQLite.
3. **Autenticación:** clave API igual en hosting y Web Credential de APEX.
4. **Oracle:** ACL HTTPS, workspace, configuración, compilación del paquete.
5. **SMTP:** STARTTLS, usuario, contraseña y remitente permitidos.
6. **Entrega:** límites del proveedor, reputación, SPF/DKIM/DMARC, spam y rebotes.

`03_health.sql` llega hasta la capa 4. El correo de prueba comprueba la 5.
La recepción manual o los registros de entrega comprueban la 6.

## ORA-29278 / SMTP 421 en una conexión directa

Este código no demuestra que el SMTP haya recibido la conexión. UTL_SMTP también
convierte errores de red en 421. Una ACL correcta puede coexistir con una ruta
de salida restringida por Autonomous. Probar el SMTP desde una PC no prueba
la misma ruta desde Oracle.
[UTL_SMTP](https://docs.oracle.com/en/database/oracle/oracle-database/26/arpls/UTL_SMTP.html).

Este proyecto usa HTTPS para el tramo Oracle-hosting. El tramo hosting-SMTP sigue
requiriendo conectividad y autorización del proveedor. No cambies a una IP,
amplíes ACL a `*` ni desactives TLS como primer intento de solución.

## HTTP 500: consola y web usan PHP distintos

Un caso frecuente: Terminal muestra PHP 8.x, pero `.htaccess` de la web fija
PHP 7.4. La sintaxis del servicio no funciona con PHP 7.4.

- Revisa versión web con herramientas del proveedor, no solo `php -v`.
- Inspecciona únicamente el handler de `.htaccess`; no publiques el archivo completo.
- Revisa el error log de la carpeta del servicio, evitando mostrar secretos.
- Configura PHP 8.1+ solo para el servicio. El ejemplo opcional usa ea-php84
  y debe coincidir con un handler realmente instalado y compatible.
- PHP-FPM y PHP Selector tienen reglas diferentes. Si no admite versión por carpeta,
  utiliza un subdominio dedicado o pide al proveedor una configuración equivalente.

El bootstrap actualizado devuelve `503 php_version_requires_8_1` ante una versión
demasiado antigua. Errores anteriores al bootstrap aún pueden aparecer como 500.
[Versión por directorio en cPanel](https://support.cpanel.net/hc/en-us/articles/24112641934103-How-to-set-a-different-PHP-version-per-directory).

## HTTP 401

Sin clave es el comportamiento esperado. Con la credencial de APEX:

- Static ID: APEX_MAIL_RELAY, exactamente igual al configurado en AMR_CONFIG.
- Tipo: HTTP Header, no Basic Authentication.
- Credential Name: X-APEX-Mail-Key.
- El secreto debe coincidir con el que se ingresó en setup, no con la contraseña SMTP.
- Valid for URLs debe incluir el endpoint real y usar HTTPS.
- Comprueba que un proxy o WAF no quite la cabecera; no imprimas su valor.

Si una clave pudo exponerse, pausa el servicio y el job, reemplaza su hash en la
configuración privada usando un procedimiento seguro y actualiza Web Credentials
personalmente. No incluyas el secreto en un comando con historial o en Git.

## HTTP 503

| Respuesta | Qué revisar |
|---|---|
| not_configured | Ruta privada o config.php ausente. |
| php_version_requires_8_1 | PHP web incompatible. |
| dependencies | PHP, PDO SQLite, OpenSSL o autoload de Composer. |
| disabled | Falta autorizar el envío mediante control.php enable. |
| smtp_not_configured | Falta contraseña SMTP o remitente válido. |
| unavailable | Permisos/almacenamiento o error interno; revisar logs privados. |

El usuario PHP debe poder atravesar el directorio privado y escribir SQLite.
No uses chmod 777. En hosting compartido, confirma con el proveedor qué usuario
ejecuta PHP antes de cambiar propietarios o permisos.

## ORA-24247, certificados o DNS

- ORA-24247: revisa hostname, puerto 443 y principales del ACL. La identidad ADMIN
  del workspace no equivale a ADMIN de la base.
- ORA-29024 / ORA-24263: revisar cadena de certificados y nombre DNS exacto;
  no desactivar verificación. El certificado SMTP y el certificado web son pruebas distintas.
- Timeout: revisar DNS, salida de red, WAF/firewall y logs del hosting a la hora del intento.
  No concluir por un timeout que la contraseña SMTP está mal: todavía puede no usarse.

## El job existe pero no envía

Ejecuta `07_status_readonly.sql` y revisa:

1. AMR_CONFIG.enabled debe ser Y y el servicio PHP estar habilitado.
2. Job ENABLED = TRUE; revisa la última ejecución y `error#` si falla.
3. Debe haber mensajes NEW/PROCESSING con menos de 20 intentos.
4. Las filas deben estar confirmadas. El worker autónomo no ve filas sin COMMIT
   de la sesión padre.
5. El workspace debe tener la Web Credential. El job establece su contexto antes
   de usar APEX_WEB_SERVICE; no depende de una sesión de navegador abierta.
6. Una cola vacía puede generar SUCCEEDED: eso prueba ejecución, no entrega.

Si falta CREATE JOB, el DBA debe conceder únicamente ese permiso al esquema
correspondiente. No ejecutes el job como ADMIN para evitar configurar el esquema.

## SENT pero no llegó

SENT significa aceptación del SMTP, no entrega garantizada. Revisa entrada, spam,
bandeja del remitente para rebotes y monitor de entrega del hosting. Confirma
remitente permitido, SPF, DKIM, DMARC y cuotas con el proveedor. No reemplaces
registros SPF ni MX existentes sin revisar la configuración del dominio.

## UNCERTAIN, timeout o proceso interrumpido

El relay registra PROCESSING antes del SMTP. Si el servidor se cae o se pierde
la respuesta, otro intento consulta el mismo ID. No vuelve a enviar ese ID.
Pasados tres minutos, PROCESSING se presenta como UNCERTAIN al consultar estado.

Pide al proveedor revisar el Message-ID del [contrato API](API.md). Si no puedes
resolverlo, toma una decisión de negocio explícita sobre un mensaje nuevo y sus
posibles duplicados; para enlaces de acceso, invalida los tokens anteriores.
No borres SQLite, no cambies IDs del payload ni conviertas UNCERTAIN a NEW a ciegas.

## Restaurar o retirar

1. Deshabilita el job y el transporte con `08_pause.sql`.
2. Ejecuta `php ~/apex-mail-relay-private/control.php disable`.
3. Conserva Oracle y SQLite para reconciliación.
4. Respáldalos de forma protegida y coherente antes de migrar.
5. Retira la ruta web y la credencial solo cuando no queden operaciones por resolver.

No se proporciona un DROP automático. Nunca borres objetos o estados para
"volver a intentar" sin identificar qué mensajes pudieron salir.
