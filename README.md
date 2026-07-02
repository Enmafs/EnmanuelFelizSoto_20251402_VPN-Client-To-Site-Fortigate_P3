# 🔒 Lab 11 — VPN Client-to-Site (Dial-Up) FortiGate

> **Práctica 3 — Semana 6 | Seguridad en Redes**
> Instituto Tecnológico de Las Américas (ITLA)
> Estudiante: Enmanuel Feliz Soto | Matrícula: 2025-1402
> Docente: Jonathan Esteban Rondón Corniel | Sección: 2-1C

---

## 📋 Descripción

Implementación de una VPN **Client-to-Site (Dial-Up)** en FortiGate usando **IKEv1 en modo Aggressive** con XAuth y Mode-Config para asignar IPs dinámicas a los clientes. Compatible con **FortiClient (Windows)** y **vpnc (Linux)**. El lab también incluye las políticas Site-to-Site del Lab 10 como base, agregando la política de acceso remoto.

---

## 🗺️ Topología de Red

<img width="1492" height="577" alt="image" src="https://github.com/user-attachments/assets/4dd14708-fbde-4b50-bde1-581d77e808b0" />


---

## 📊 Tabla de Direccionamiento

| Dispositivo      | Interfaz / Pool | Dirección IP            | Descripción                       |
|------------------|-----------------|-------------------------|-----------------------------------|
| FGT-A            | port1 (WAN)     | 20.25.1.2 /30           | Enlace hacia ISP                  |
| FGT-A            | port2 (LAN)     | 14.2.10.254 /24         | Red interna                       |
| ISP Router       | e0/0 (WAN)      | 20.25.1.1 /30           | Gateway FGT-A                     |
| ISP Router       | e0/2 (LAN)      | 192.168.100.1 /25       | Gateway FGT-A                     |
| Clientes VPN     | Asignada        | 172.16.1.10 – .100 /24  | Pool Mode-Config (Dial-Up)        |

---

## 🔐 Parámetros IKE / IPSec — Dial-Up

### Phase 1 (IKEv1 Aggressive + XAuth)

| Parámetro           | Valor                              |
|---------------------|------------------------------------|
| IKE Version         | **1**                              |
| Tipo                | **Dynamic (Dial-Up)**              |
| Modo                | **Aggressive**                     |
| Autenticación       | Pre-Shared Key + XAuth             |
| Clave PSK           | `ClaveDialup123!`                  |
| Propuesta Cifrado   | DES / MD5                          |
| DH Groups           | 14, 5, 2 (compat. Linux legacy)    |
| DPD                 | On-Idle                            |
| XAuth Type          | Auto                               |
| User Group          | VPN_Users                          |
| Mode-Config         | Enable                             |
| IP Pool Start       | 172.16.1.10                        |
| IP Pool End         | 172.16.1.100                       |
| Netmask pool        | 255.255.255.0                      |

### Phase 2 (Quick Mode)

| Parámetro    | Valor              |
|--------------|--------------------|
| Phase1 Name  | VPN-DialUp         |
| Propuesta    | DES / MD5          |
| PFS          | **Disable** ⚠️     |

> ⚠️ PFS deshabilitado para evitar el error `ISAKMP_N_NO_PROPOSAL_CHOSEN` con clientes Linux (vpnc).

---

## 👤 Usuarios y Grupos

| Usuario   | Tipo           | Contraseña           | Grupo     |
|-----------|----------------|----------------------|-----------|
| vpnuser   | Password local | `UsuarioSeguro123!`  | VPN_Users |

---

## 🖥️ Evidencias de Configuración

### Figura 1 — FGT-A: Phase 1 Dial-Up (VPN-DialUp)
![Phase 1 Dial-Up](screenshots/lab11_dialup_phase1.png)

### Figura 2 — Usuarios, Grupos y Políticas de Firewall
![Users y Policy](screenshots/lab11_users_policy.png)

---

## ⚙️ Procedimiento de Configuración

### 1. Crear Usuario y Grupo Local

```fortios
config user local
    edit "vpnuser"
        set type password
        set passwd "UsuarioSeguro123!"
    next
end

config user group
    edit "VPN_Users"
        set member "vpnuser"
    next
end
```

### 2. VPN IPSec Phase 1 — Dial-Up

```fortios
config vpn ipsec phase1-interface
    edit "VPN-DialUp"
        set type dynamic
        set interface "port1"
        set ike-version 1
        set peertype any
        set net-device disable
        set mode aggressive
        set proposal des-md5
        set dhgrp 14 5 2
        set dpd on-idle
        set xauthtype auto
        set authusrgrp "VPN_Users"
        set mode-cfg enable
        set ipv4-start-ip 172.16.1.10
        set ipv4-end-ip 172.16.1.100
        set ipv4-netmask 255.255.255.0
        set psksecret "ClaveDialup123!"
    next
end
```

### 3. VPN IPSec Phase 2

```fortios
config vpn ipsec phase2-interface
    edit "VPN-DialUp-P2"
        set phase1name "VPN-DialUp"
        set proposal des-md5
        set pfs disable
    next
end
```

### 4. Política de Firewall — Acceso Cliente → LAN

```fortios
config firewall policy
    edit 3
        set name "DialUp_to_LAN"
        set srcintf "VPN-DialUp"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
end
```

---

## 🖥️ Configuración del Cliente (FortiClient - Windows)

| Campo            | Valor                  |
|------------------|------------------------|
| Connection Name  | VPN-Lab11              |
| Remote Gateway   | 20.25.1.2              |
| Authentication   | Pre-Shared Key         |
| PSK              | `ClaveDialup123!`      |
| Username         | vpnuser                |
| Password         | `UsuarioSeguro123!`    |

## 🐧 Configuración del Cliente (vpnc - Linux)

```bash
# /etc/vpnc/vpn-lab11.conf
IKE Authmode psk
IPSec gateway 20.25.1.2
IPSec ID vpnuser
IPSec secret ClaveDialup123!
Xauth username vpnuser
Xauth password UsuarioSeguro123!

# Conectar
sudo vpnc vpn-lab11.conf
```

---

## ✅ Verificación

```fortios
# Listar usuarios conectados al túnel Dial-Up
get vpn ipsec tunnel summary

# Ver IPs asignadas por Mode-Config
diagnose vpn ike gateway list

# Verificar Phase 2
diagnose vpn tunnel list

# Ping desde cliente hacia LAN interna (después de conectar)
ping 14.2.10.254
```

---

## 📝 Notas Técnicas

- **DH Groups 14, 5, 2:** agregados para soportar clientes Linux con `vpnc` que no negocian DH 14 por defecto.
- **Aggressive Mode:** requerido para Dial-Up con XAuth en IKEv1; permite al FortiGate identificar el grupo de usuarios antes de completar la autenticación.
- **PFS Disable:** necesario para evitar fallos de negociación con clientes que no soportan PFS en Phase 2.
- **Mode-Config:** permite que el FortiGate asigne automáticamente una IP del pool al cliente sin configuración manual en el endpoint.

---

## 📁 Recursos

| Recurso | Enlace |
|---------|--------|
| Repositorio | [EnmanuelFelizSoto_20251402_VPN-Clientt-To-Site-Fortigate](https://github.com/Enmafs/EnmanuelFelizSoto_20251402_VPN-Clientt-To-Site-Fortigate) |
| Repositorio padre (NetSec) | [Enmafs/NetSec](https://github.com/Enmafs/NetSec) |
| Video demostrativo | 🎬 [Aquí](https://youtu.be/RCI8Qt6YqPU) |

---

> ⚠️ *Laboratorio realizado en entorno controlado (PNetLab). Fines exclusivamente académicos.*
