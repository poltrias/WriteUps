# 🖥️ Writeup - Cocido Andaluz 

**Plataforma:** The Hackers Labs  
**Sistema Operativo:** Windows  

# INSTALACIÓN

Descargamos el archivo `zip` que contiene la `.ova` de la máquina Cocido Andaluz, lo extraemos y la importamos en VirtualBox.

Configuramos la interfaz de red de la máquina Cocido Andaluz y la iniciamos junto a nuestra máquina atacante.

# HOST DISCOVERY

En este punto, aún desconocemos la dirección `IP` asignada a Cocido Andaluz, por lo que procedemos a descubrirla:

```bash
netdiscover -i eth1 -r 10.0.0.0/16
```
Info:
```
Currently scanning: 10.0.0.0/16   |   Screen View: Unique Hosts               
                                                                               
 4 Captured ARP Req/Rep packets, from 4 hosts.   Total size: 240               
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname      
 -----------------------------------------------------------------------------
 10.0.4.1        52:54:00:12:35:00      1      60  Unknown vendor              
 10.0.4.2        52:54:00:12:35:00      1      60  Unknown vendor              
 10.0.4.3        08:00:27:67:2e:3e      1      60  PCS Systemtechnik GmbH      
 10.0.4.38       08:00:27:9a:e7:06      1      60  PCS Systemtechnik GmbH
 ```

Identificamos con total seguridad que la `IP` de la víctima es `10.0.4.38`.

# ESCANEO DE PUERTOS    

A continuación, realizamos un escaneo general para identificar qué puertos están abiertos, seguido de un escaneo más exhaustivo para enumerar las versiones y servicios que corren en ellos.

```bash
nmap -n -Pn -sS -sV -p- --open --min-rate 5000 10.0.4.38
``` 

```bash
nmap -n -Pn -sCV -p22,80 --min-rate 5000 10.0.4.38
```
Info:
```

```