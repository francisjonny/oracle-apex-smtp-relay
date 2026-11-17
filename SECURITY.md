# Seguridad

No publiques claves, contraseñas SMTP, `config.php`, archivos SQLite, enlaces de
recuperación, correos de usuarios ni exports de aplicaciones en issues o PR.

El único archivo PHP público es `hosting/public/index.php`. Instala todo lo
demás fuera del directorio web. `setup.php` y `config.php` nunca deben ser
accesibles mediante HTTP. Usa permisos 0700 para el directorio privado y 0600
para configuración y estado; el usuario PHP debe ser su propietario.

La clave API se almacena como SHA256 en el relay. La contraseña SMTP queda en
un archivo privado del hosting; la clave API de cliente queda cifrada en Web
Credentials de APEX. No existe cifrado adicional del archivo privado SMTP:
protege también copias de seguridad, usuarios del hosting y acceso al servidor.

El servicio valida TLS, exige autenticación, fija el remitente, limita tamaño y
frecuencia y no acepta URLs SMTP, remitentes, adjuntos o cabeceras del cliente.
No agregues `verify_peer=false`, `allow_self_signed=true` ni claves en la URL.

Un timeout SMTP puede ocurrir después de aceptar el mensaje. `UNCERTAIN` exige
reconciliación manual: no borres el estado para forzar un reenvío. No se promete
exactly-once: la idempotencia evita repetir el intento de un mismo ID, pero SMTP
no permite confirmar entrega al destinatario ni todas las condiciones de fallo.

Para reportar un problema, usa el reporte privado de vulnerabilidades de GitHub
si está habilitado en el repositorio. Si no aparece, abre un issue sin secretos
ni detalles de explotación y solicita un canal privado. No se publica una
dirección de contacto personal en este proyecto.
