# TryHackMe Writeups ![imagen](https://github.com/user-attachments/assets/6c6161c9-257a-4181-aee5-11f9239913a0)


En este repositorio iré subiendo los WriteUps de las máquinas de la plataforma TryHackMe. En TryHackMe, puedes acceder a entornos controlados que se despliegan automáticamente, ideales para practicar hacking ético y pentesting.

## ¿Qué es TryHackMe?

TryHackMe es una plataforma que proporciona entornos de laboratorio diseñados específicamente para aprender y practicar habilidades de seguridad informática. Estos entornos están preconfigurados con una variedad de aplicaciones y configuraciones que contienen vulnerabilidades conocidas, permitiendo a los usuarios experimentar y aprender sobre ellas de manera segura.

- **[TryHackMe](https://tryhackme.com/)**: Sitio web oficial de TryHackMe donde puedes encontrar más información sobre la plataforma y acceder a los entornos de laboratorio.

## Estructura del Repositorio

- **Máquinas.md/**: Cada archivo estará etiquetado con el nombre de la máquina correspondiente y contendrá la solución paso a paso.
- **README.md**: Este archivo que estás leyendo ahora mismo, proporciona una descripción general del repositorio y cómo utilizarlo.

## Contenido

Aquí encontrarás los writeups de las máquinas que he completado en TryHackMe. Cada writeup incluye una descripción detallada de la metodología y las soluciones aplicadas.

### Máquinas

- [**GamingServer**](GamingServer.md): es una máquina Linux de dificultad fácil en la plataforma TryHackMe. La intrusión inicial se consigue explotando una clave privada RSA encontrada en el directorio web oculto, que, tras ser crackeada, permite acceder al sistema como el usuario `john`. Posteriormente, la escalada de privilegios se realiza aprovechando la pertenencia del usuario al grupo `lxd`, lo que permite crear un contenedor privilegiado y montar el sistema de archivos del host, otorgando acceso root. La máquina requiere un enfoque de enumeración cuidadosa y técnicas de cracking de claves para lograr la intrusión y escalada.
- [Máquina2](Máquina2.md): Descripción del writeup de la segunda máquina. Incluye detalles sobre la metodología y los pasos seguidos para explotar las vulnerabilidades encontradas.

## Contacto

Si tienes alguna pregunta, sugerencia o simplemente quieres conectar, no dudes en contactarme:

- **Email:** [tu-email@example.com](mailto:tu-email@example.com)
- **LinkedIn:** [Tu Perfil de LinkedIn](https://www.linkedin.com/in/tu-perfil)
