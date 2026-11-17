# Oracle APEX SMTP Relay

Envío de correo desde Oracle APEX mediante HTTPS y tu propio SMTP. Incluye servicio PHP, ejemplos Oracle genéricos, cola transaccional y job automático.

## Descargar e implementar

1. **[Descargar el paquete v1.0.0](oracle-apex-smtp-relay-v1.0.0.zip)** — en la página del archivo, pulsa el botón de descarga.
2. Extrae el paquete y sigue la **[guía de implementación paso a paso](GUIA_IMPLEMENTACION.md)**.
3. Configura y ejecuta personalmente los scripts Oracle en tu entorno. Primero diagnóstico, después un correo de prueba y finalmente el job.

El botón `Code → Download ZIP` de GitHub descarga este repositorio de distribución: dentro encontrarás el paquete que también debes extraer.

## Documentación

- [Instalación, configuración, pruebas e integración](GUIA_IMPLEMENTACION.md)
- [Contrato API y estados](API.md)
- [Solución de problemas](TROUBLESHOOTING.md)
- [Seguridad](SECURITY.md)
- [Cambios de la versión](CHANGELOG.md)
- [Licencias de dependencias](THIRD_PARTY.md)

## Qué contiene el paquete

34 archivos de código y documentación: servicio HTTPS en PHP, configuración privada interactiva, dependencia PHPMailer fijada mediante Composer, ejemplos SQL independientes, pruebas automatizadas PHP/HTTP/concurrencia y workflow de ejemplo para GitHub Actions.

No contiene ERP, exports de aplicaciones, claves, contraseñas, configuración real ni conexiones de base de datos. Las dependencias se instalan con Composer siguiendo la guía; no se incluyen archivos de ejecución ni `vendor/`.

Las pruebas locales del servicio aprobaron 22 casos unitarios, 10 controles HTTP y una prueba concurrente con SMTP simulado. No enviaron correos reales. Los ejemplos SQL generalizados requieren compilación y validación personal en Oracle. Este repositorio de distribución no ejecuta GitHub Actions; el workflow está incluido dentro del paquete.

## Integridad

SHA-256 de `oracle-apex-smtp-relay-v1.0.0.zip`:

```text
EAD72A9FAB7EB8ECA0F8F7E707A7014DFCF8AD07957A960B2A833D7802B5F615
```

## Acceso y licencia

El repositorio es público para consulta y descarga. Descargarlo no concede permisos de escritura sobre este repositorio.

Código propio bajo [licencia MIT](LICENSE), que permite usar, copiar y modificar las copias descargadas, conservando los avisos. PHPMailer tiene su propia licencia, indicada en [THIRD_PARTY.md](THIRD_PARTY.md).
