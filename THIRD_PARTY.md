# Dependencias

| Dependencia | Versión | Licencia | Fuente |
|---|---|---|---|
| PHPMailer | 7.1.1, fijada mediante Composer | LGPL-2.1 | https://github.com/PHPMailer/PHPMailer |

El código propio usa MIT. Esa licencia no cambia la de PHPMailer. Composer
instala la dependencia en `hosting/private/vendor/`, ignorado por Git.
Conserva sus avisos y licencia cuando distribuyas el directorio vendor.

Las pruebas emplean Node.js y PHP, que no se distribuyen con el proyecto.
SQLite se utiliza mediante la extensión PDO SQLite de PHP.
