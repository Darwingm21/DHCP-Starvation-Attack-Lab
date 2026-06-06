# DHCP-Starvation-Attack-Lab# Ataque DHCP Starvation

![Python](https://img.shields.io/badge/Python-3.x-blue)
![Platform](https://img.shields.io/badge/Platform-Kali%20Linux-red)
![Environment](https://img.shields.io/badge/Environment-GNS3%20%7C%20IOSvL2-orange)
![Attack](https://img.shields.io/badge/Attack-DHCP%20Starvation-purple)
![Mitigation](https://img.shields.io/badge/Mitigation-DHCP%20Snooping%20%7C%20Port%20Security-darkgreen)
![Use](https://img.shields.io/badge/Use-Controlled%20Lab-yellow)

## Información del proyecto

| Dato                  | Información                                          |
| --------------------- | ---------------------------------------------------- |
| Autor                 | Darwing                                              |
| Matrícula             | 2024-2690                                            |
| Docente               | Jonathan Rondon                                      |
| Repositorio           | https://github.com/TuUsuario/DHCP-Starvation-Attack  |
| Video demostrativo    | https://youtu.be/j-SWntE8cis            |
| Documentación técnica | docs/documentacion-tecnica-profesional.pdf           |
| Red de laboratorio    | 20.24.26.0/24                                        |

---

## Aviso de uso responsable

Este proyecto fue desarrollado únicamente con fines educativos, académicos y de laboratorio controlado. Las pruebas deben ejecutarse solamente en entornos propios o autorizados como GNS3, EVE-NG, PNETLab o laboratorios internos. No debe utilizarse en redes públicas, empresariales o de terceros sin autorización explícita.

---

## Objetivo del laboratorio

Demostrar un ataque **DHCP Starvation**, donde Kali Linux envía múltiples solicitudes DHCP usando direcciones MAC falsas y aleatorias hasta agotar completamente el pool de direcciones del servidor DHCP legítimo (R1). Como resultado, un cliente legítimo (PC1) queda sin posibilidad de obtener una dirección IP.

Después de demostrar el impacto, se aplican contramedidas basadas en **DHCP Snooping**, **limitación de tasa** y **Port Security** para bloquear el ataque.

---

## Archivos del repositorio

| Archivo                                        | Descripción                                                           |
| ---------------------------------------------- | --------------------------------------------------------------------- |
| `dhcp_starvation.py`                           | Script principal para ejecutar el ataque DHCP Starvation.             |
| `mitigacion-dhcp-starvation.md`                | Documento con las contramedidas aplicadas.                            |
| `README.md`                                    | Guía principal del laboratorio.                                       |
| `docs/documentacion-tecnica-profesional.pdf`   | Documentación técnica profesional detallada del laboratorio.          |
| `images/`                                      | Capturas de pantalla de evidencia del laboratorio.                    |

---

## Topología del laboratorio

La red de laboratorio utiliza el segmento `20.24.26.0/24` con R1 como servidor DHCP legítimo, SW-1 como switch de capa 2, Kali como atacante y PC1 como cliente legítimo.

```
         R1 (20.24.26.91)
         f0/0
          |
         Gi0/2
        [SW-1]
       /       \
   Gi0/0      Gi0/1
   Kali        PC1
(20.24.26.90)  (víctima)
```

| Dispositivo | Rol                     | Interfaz | Dirección IP    | Descripción                        |
| ----------- | ----------------------- | -------- | --------------- | ---------------------------------- |
| R1          | Gateway / DHCP legítimo | F0/0     | 20.24.26.91/24  | Servidor DHCP real de la red       |
| SW-1        | Switch capa 2           | Gi0/0~2  | N/A             | Switch objetivo de la contramedida |
| Kali Linux  | Atacante                | eth0     | 20.24.26.90/24  | Ejecuta el flood de DHCP Discover  |
| PC1         | Víctima / Cliente       | e0       | DHCP            | Cliente legítimo que pierde acceso |

---

## Requisitos previos

- GNS3 o entorno de virtualización equivalente.
- Router Cisco con pool DHCP configurado.
- Switch Cisco IOSvL2.
- Kali Linux con Python 3 y Scapy instalado.
- Permisos de superusuario (`sudo`).
- Conectividad de capa 2 entre Kali y el switch.

```bash
pip install scapy
```

---

## Configuración del pool DHCP en R1

Antes del ataque, R1 debe tener el servicio DHCP activo:

```
service dhcp
interface fastEthernet0/0
 ip address 20.24.26.91 255.255.255.0
 no shutdown

ip dhcp excluded-address 20.24.26.1 20.24.26.45
ip dhcp pool LAN_20242690
 network 20.24.26.0 255.255.255.0
 default-router 20.24.26.91
 dns-server 20.24.26.91
 lease 0 1
```

Verificar el pool antes del ataque:

```
R1# show ip dhcp pool
R1# show ip dhcp binding
```

---

## Parámetros del script

| Parámetro          | Descripción                                            | Ejemplo    |
| ------------------ | ------------------------------------------------------ | ---------- |
| `-i` / `--iface`   | Interfaz de red usada para enviar los DHCP Discover    | `-i eth0`  |
| `-c` / `--count`   | Cantidad de paquetes a enviar (0 = infinito)           | `-c 200`   |
| `-d` / `--delay`   | Tiempo entre paquetes en segundos                      | `-d 0.05`  |
| `-v` / `--verbose` | Muestra cada paquete enviado en pantalla               | `-v`       |

---

## Ejecución del ataque

```bash
sudo python3 dhcp_starvation.py -i eth0 -c 200 -v
```

Durante la ejecución el script genera paquetes DHCP Discover con MACs aleatorias:

```
╔══════════════════════════════════════════════╗
║         DHCP Starvation Attack               ║
║  Interfaz : eth0                            ║
║  Paquetes : 200                             ║
╚══════════════════════════════════════════════╝
[*] Agotando pool DHCP... (Ctrl+C para detener)

[     1] DISCOVER → MAC=02:a1:b2:c3:d4:e5
[     2] DISCOVER → MAC=02:f3:e2:11:cc:45
...
[+] Total paquetes enviados: 200
```

Para detener: `Ctrl + C`

---

## Verificación del impacto

Durante el ataque verificar en R1:

```
R1# show ip dhcp binding
R1# show ip dhcp pool
```

Se observará el pool con decenas de asignaciones a MACs falsas. Luego intentar que PC1 obtenga IP:

```
PC1> ip dhcp
```

PC1 no recibirá respuesta porque el pool está agotado.

---

## Contramedida aplicada

La defensa se aplica en SW-1. Se habilita DHCP Snooping, se marca como confiable el puerto hacia R1, y se limita la tasa y aplica Port Security en el puerto del atacante.

```
enable
configure terminal

ip dhcp snooping
ip dhcp snooping vlan 1
no ip dhcp snooping information option

interface gigabitEthernet0/2
 description HACIA_R1_DHCP_LEGITIMO
 ip dhcp snooping trust
 exit

interface gigabitEthernet0/0
 description HACIA_KALI_ATACANTE
 no ip dhcp snooping trust
 ip dhcp snooping limit rate 5
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation shutdown
 switchport port-security mac-address sticky
 exit

interface gigabitEthernet0/1
 description HACIA_PC1_CLIENTE
 no ip dhcp snooping trust
 ip dhcp snooping limit rate 5
 switchport mode access
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 exit

end
write memory
```

---

## Limpieza del router después del ataque

```
R1# clear ip dhcp binding *
R1# clear ip dhcp conflict *
R1# clear arp-cache
```

---

## Verificación posterior a la mitigación

Relanzar el ataque — el script ya no debe recibir ofertas DHCP para las MACs falsas. Luego PC1 debe poder obtener IP normalmente:

```
PC1> clear ip
PC1> ip dhcp
```

Verificar en el switch:

```
SW-1# show ip dhcp snooping
SW-1# show ip dhcp snooping binding
SW-1# show port-security interface gigabitEthernet0/0
SW-1# show interfaces status
```

Verificar en R1:

```
R1# show ip dhcp binding
R1# show ip dhcp pool
```

---

## Comandos de verificación completos

**En el switch:**
```
show ip dhcp snooping
show ip dhcp snooping binding
show port-security interface gigabitEthernet0/0
show interfaces status
show logging
```

**En el router:**
```
show ip dhcp binding
show ip dhcp pool
show ip dhcp conflict
```

**En PC1:**
```
PC1> ip dhcp
PC1> show ip
```

---

## Conclusión

El ataque DHCP Starvation afecta directamente la disponibilidad del servicio DHCP al consumir el pool de direcciones mediante solicitudes generadas con MACs falsas. Cuando el pool se agota, clientes legítimos como PC1 no pueden integrarse a la red.

La mitigación aplicada es efectiva: **DHCP Snooping** diferencia puertos confiables y no confiables, **rate limit** restringe los mensajes DHCP por segundo desde el puerto atacante, y **Port Security** impide que una sola interfaz registre múltiples MACs falsas. En conjunto, estas medidas evitan que el pool sea agotado y permiten que los clientes legítimos reciban DHCP normalmente.

---

## Autor

**Darwing**  
Matrícula: **2024-2690**  
Repositorio: https://github.com/TuUsuario/DHCP-Starvation-Attack
