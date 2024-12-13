# HackTheBox Writeups ![imagen](https://github.com/user-attachments/assets/f2108645-7cb5-4699-8d63-e0b8854cf853)


En este repositorio iré subiendo los WriteUps de las máquinas de la plataforma HackTheBox. HackTheBox ofrece entornos controlados ideales para practicar hacking ético y pentesting.

## ¿Qué es HackTheBox?

HackTheBox es una plataforma que proporciona entornos de laboratorio diseñados específicamente para aprender y practicar habilidades de seguridad informática. Estos entornos están preconfigurados con una variedad de aplicaciones y configuraciones que contienen vulnerabilidades conocidas, permitiendo a los usuarios experimentar y aprender sobre ellas de manera segura.

- **[HackTheBox](https://www.hackthebox.com/)**: Sitio web oficial de HackTheBox donde puedes encontrar más información sobre la plataforma y acceder a los entornos de laboratorio.

## Estructura del Repositorio

- **Máquinas.md/**: Cada archivo estará etiquetado con el nombre de la máquina correspondiente y contendrá la solución paso a paso.
- **README.md**: Este archivo que estás leyendo ahora mismo, proporciona una descripción general del repositorio y cómo utilizarlo.

## Contenido

Aquí encontrarás los writeups de las máquinas que he completado en HackTheBox. Cada writeup incluye una descripción detallada de la metodología y las soluciones aplicadas.

### Máquinas

- [Cicada](cicada.md): Cicada es una máquina enfocada en la explotación de vulnerabilidades en entornos Windows, con un énfasis especial en la enumeración y explotación de SMB (Server Message Block). Durante el proceso, se emplean técnicas de reconocimiento de usuarios y comparticiones SMB mediante herramientas como netexec y crackmapexec.
- [Máquina2](Forest.md): Se trata de una máquina Windows en la que realizaremos una enumeración RPC para recopilar información sobre los usuarios del dominio, lo que nos permitirá llevar a cabo el ataque AS-RepRoast y, finalmente, descifrar la contraseña de un usuario comprometido. Esto nos otorgará acceso al sistema a través de WinRM. Para la escalada de privilegios, llevaremos a cabo una exploración exhaustiva del dominio utilizando herramientas como BloodHound y SharpHound para identificar una posible vía de compromiso que nos permita elevar nuestros privilegios hasta convertirnos en Administrador, utilizando técnicas como WriteDacl y DCSync.
## Contacto

Si tienes alguna pregunta, sugerencia o simplemente quieres conectar, no dudes en contactarme:

- **Email:** [tu-email@example.com](mailto:tu-email@example.com)
- **LinkedIn:** [Tu Perfil de LinkedIn](https://www.linkedin.com/in/tu-perfil)
