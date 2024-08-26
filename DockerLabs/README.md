# DockerLabs Writeups
![imagen](https://github.com/user-attachments/assets/d9358beb-d99b-4b71-aa72-b278eb79ed3e)


En este repositorio iré subiendo los WriteUps de las máquinas de la plataforma DockerLabs. En DockerLabs, puedes descargar entornos controlados en Docker que se despliegan automáticamente con un script en bash. Estos entornos son ideales para practicar hacking ético y pentesting.

## ¿Qué es DockerLabs?

DockerLabs es una plataforma que proporciona entornos de laboratorio basados en Docker, diseñados específicamente para aprender y practicar habilidades de seguridad informática. Estos entornos están preconfigurados con una variedad de aplicaciones y configuraciones que contienen vulnerabilidades conocidas para que los usuarios puedan experimentar y aprender sobre ellas de manera segura.

- **[DockerLabs](https://dockerlabs.es/#/)**: Sitio web oficial de DockerLabs donde puedes encontrar más información sobre la plataforma y acceder a los entornos de laboratorio.

## Estructura del Repositorio

- **Máquinas.md/**: Cada archivo estará etiquetado con el nombre de la máquina correspondiente y contendrá la solución paso a paso.
- **README.md**: Este archivo que estás leyendo ahora mismo, proporciona una descripción general del repositorio y cómo utilizarlo.

## Contenido

Aquí encontrarás los writeups de las máquinas que he completado en DockerLabs. Cada writeup incluye una descripción detallada de la metodología y las soluciones aplicadas.

### Máquinas

- [Fooding](Fooding.md): En este laboratorio de hacking, escaneamos el sistema con nmap y encontramos múltiples servicios abiertos en los puertos 80, 443, 1883, 5672, 8161, 41031, 61613, 61614 y 61616. Identificamos una vulnerabilidad en ActiveMQ 5.15.15 y la explotamos utilizando un exploit específico para esta versión. Luego, obtuvimos acceso root ejecutando una reverse shell debido a que el servicio explotado corría con privilegios de root.
- [Eclipse](Eclipse.md): En este laboratorio de hacking, escaneamos el sistema con nmap y encontramos servicios en los puertos 80 y 8983. Identificamos una vulnerabilidad en Solr y la explotamos con Metasploit. Luego, utilizamos el binario dosbox para escalar privilegios y obtener acceso root ejecutando `sudo su`.
- [Stranger](Stranger.md): En este laboratorio de hacking, realizamos un escaneo con nmap y encontramos servicios corriendo en los puertos 21, 22 y 80. Descubrimos credenciales de usuario utilizando fuerza bruta en el servicio FTP con Hydra. Luego, desencriptamos un archivo usando OpenSSL para obtener la contraseña del usuario mwheeler. Con esta contraseña, accedimos al sistema vía SSH. Finalmente, escalamos privilegios a root manipulando un script de backup ejecutado por root, lo que nos permitió obtener una shell con privilegios de root.
- [Dark](Dark.md): Esta máquina Linux está diseñada para poner a prueba habilidades en reconocimiento, enumeración, explotación de vulnerabilidades y pivoting. Los usuarios deberán realizar fuerza bruta de credenciales SSH, explotar parámetros vulnerables en una aplicación web y llevar a cabo una escalada de privilegios. La máquina tiene dos direcciones IP: 10.10.10.2 y 20.20.20.3, lo que introduce elementos de pivoting y tunneling en el proceso de explotación.
- [Pinguinazo](Pinguinazo.md): Este laboratorio de hacking se centra en la explotación de vulnerabilidades como HTML Injection, Server-Side Template Injection (SSTI) y una escalada de privilegios mediante `sudo java`. Inicialmente, se descubre una vulnerabilidad de inyección HTML que permite la ejecución de código malicioso en la página web. Luego, se aprovecha una SSTI para ejecutar comandos en el servidor. Finalmente, se logra la escalada de privilegios utilizando `sudo java` para ejecutar un archivo JAR malicioso que establece una reverse shell como root.
- [Winterfell](Winterfell.md): En este laboratorio de hacking, realizamos un escaneo con nmap y descubrimos servicios corriendo en los puertos 22, 80, 139 y 445. Utilizamos fuerza bruta en SMB para obtener credenciales y acceder a recursos compartidos. Escalamos privilegios a diferentes usuarios manipulando scripts con permisos sudo y expuestos en el sistema, hasta obtener una shell con privilegios root.
- [Showtime](Showtime.md): En este laboratorio de hacking, comenzamos con un escaneo de puertos que reveló servicios en los puertos 22 y 80. A través de una inyección SQL en la página de login, accedimos a la aplicación web con privilegios administrativos. Luego, ejecutamos comandos del sistema a través de un formulario vulnerable en la web, obteniendo una shell. Posteriormente, encontramos una lista de contraseñas en un archivo oculto, permitiéndonos realizar un ataque de fuerza bruta y escalar privilegios al usuario joe. Utilizando sudo, ejecutamos un comando que nos permitió cambiar al usuario luciano y finalmente, modificamos un script con permisos de root para obtener acceso root.
- [Escolares](Escolares.md): En este laboratorio de hacking, realizamos un escaneo con nmap y encontramos servicios corriendo en los puertos 22 y 80. Identificamos un sitio WordPress vulnerable y explotamos una vulnerabilidad de fuerza bruta en la página de login utilizando un diccionario personalizado, obteniendo acceso como administrador. Subimos un archivo PHP malicioso para ejecutar comandos en el servidor y obtuvimos una reverse shell. Luego, escalamos privilegios a usuario luisillo usando credenciales encontradas en un archivo, y finalmente a root mediante el uso del binario `awk` con permisos sudo.
- [Psycho](Psycho.md): En la máquina Psycho, realizamos un escaneo inicial que reveló los puertos 22 y 80 abiertos. En el servidor web, identificamos un archivo `index.php` vulnerable a LFI (Local File Inclusion), que nos permitió leer archivos sensibles, incluyendo la clave privada SSH del usuario `vaxei`. Con esta clave, obtuvimos acceso a la máquina. Luego, escalamos privilegios al usuario `luisillo` utilizando permisos sudo para ejecutar `perl`. Finalmente, explotamos un script de Python vulnerable a un **Python Library Hijacking**, logrando acceso root y completando el laboratorio.

## Contacto

Si tienes alguna pregunta, sugerencia o simplemente quieres conectar, no dudes en contactarme:

- **Email:** [pablo.torralvoo@gmail.com](mailto:tu-email@example.com)
- **LinkedIn:** [Tu Perfil de LinkedIn](https://www.linkedin.com/in/tu-perfil)
