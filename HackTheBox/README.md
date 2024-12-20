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
- [Forest](Forest.md): Se trata de una máquina Windows en la que realizaremos una enumeración RPC para recopilar información sobre los usuarios del dominio, lo que nos permitirá llevar a cabo el ataque AS-RepRoast y, finalmente, descifrar la contraseña de un usuario comprometido. Esto nos otorgará acceso al sistema a través de WinRM. Para la escalada de privilegios, llevaremos a cabo una exploración exhaustiva del dominio utilizando herramientas como BloodHound y SharpHound para identificar una posible vía de compromiso que nos permita elevar nuestros privilegios hasta convertirnos en Administrador, utilizando técnicas como WriteDacl y DCSync.
- [Active](Active.md): Se trata de una máquina con sistema operativo **Windows** que nos permitirá practicar la técnica de **enumeración SMB**. Durante este proceso, encontraremos un archivo `Groups.xml` que contiene una contraseña cifrada con **AES-256**. Esta contraseña se ha establecido mediante **GPP (Group Policy Preferences)**, para descifrar la contraseña, usaremos la herramienta `gpp-decrypt`, que nos proporcionará las credenciales de un usuario del dominio. Con estas credenciales, podremos realizar un ataque **Kerberoasting**, que consiste en explotar una vulnerabilidad del protocolo Kerberos. Este ataque nos permitirá descubrir que la cuenta del **Administrador** tiene configurado un **SPN (Service Principal Name)**, lo que significa que podemos solicitar un **TGS (Ticket Granting Service)** para esa cuenta. Para ello, usaremos la utilidad `GetUserSPNs.py`, que nos devolverá el hash del TGS. A continuación, podremos descifrar el hash mediante un ataque de fuerza bruta offline y obtener la contraseña del Administrador. Finalmente, con la herramienta `psexec.py`, nos autenticaremos en la máquina víctima y conseguiremos el acceso completo.
- [Sauna](Sauna.md): Sauna está diseñada para poner a prueba habilidades de enumeración y explotación en un entorno de Active Directory. El desafío principal consiste en obtener acceso al sistema y escalar privilegios utilizando técnicas de ataque como ASREPRoast, DCSync, y Pass-the-Hash.
- [Administrator](Administrator.md): Esta máquina con sistema operativo Windows permite practicar diversas técnicas de enumeración y escalada de privilegios en un entorno de Active Directory. El proceso comienza con un escaneo de puertos que revela servicios como WinRM y FTP. La descripción de la máquina proporciona credenciales iniciales para el usuario "Olivia" con la contraseña "ichliebedich". Utilizando estas credenciales, se accede al sistema a través de WinRM. Posteriormente, se emplea BloodHound para analizar las relaciones y permisos dentro del dominio, identificando que el usuario "Olivia" tiene privilegios para modificar atributos de otras cuentas. Aprovechando estos permisos, se realiza un ataque DCSync mediante la herramienta impacket-secretsdump para extraer hashes de contraseñas, incluyendo la del administrador. Finalmente, con el hash obtenido, se lleva a cabo un ataque Pass-The-Hash para obtener acceso completo al sistema.

Espero que esta descripción sea de ayuda.

## Contacto

Si tienes alguna pregunta, sugerencia o simplemente quieres conectar, no dudes en contactarme:

- **Email:** [tu-email@example.com](mailto:tu-email@example.com)
- **LinkedIn:** [Tu Perfil de LinkedIn](https://www.linkedin.com/in/tu-perfil)
