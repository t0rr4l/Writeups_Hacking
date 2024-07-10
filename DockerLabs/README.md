# DockerLabs Writeups

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
- [Eclipse](Eclipse.md):En este laboratorio de hacking, escaneamos el sistema con nmap y encontramos servicios en los puertos 80 y 8983. Identificamos una vulnerabilidad en Solr y la explotamos con Metasploit. Luego, utilizamos el binario dosbox para escalar privilegios y obtener acceso root ejecutando `sudo su`.
- [Stranger](Stranger.md):En este laboratorio de hacking, realizamos un escaneo con nmap y encontramos servicios corriendo en los puertos 21, 22 y 80. Descubrimos credenciales de usuario utilizando fuerza bruta en el servicio FTP con Hydra. Luego, desencriptamos un archivo usando OpenSSL para obtener la contraseña del usuario mwheeler. Con esta contraseña, accedimos al sistema vía SSH. Finalmente, escalamos privilegios a root manipulando un script de backup ejecutado por root, lo que nos permitió obtener una shell con privilegios de root.

## Contacto

Si tienes alguna pregunta, sugerencia o simplemente quieres conectar, no dudes en contactarme:

- **Email:** [pablo.torralvoo@gmail.com](mailto:tu-email@example.com)
- **LinkedIn:** [Tu Perfil de LinkedIn](https://www.linkedin.com/in/tu-perfil)
