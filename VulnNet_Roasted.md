# TryHackMe: VulnNet - Roasted

**Dificultad:** Media 
**Sistema Operativo:** Windows (Active Directory)
**Herramientas usadas:** Nmap, CrackMapExec, Impacket (GetNPUsers, Secretsdump), John the Ripper, Evil-WinRM, SMBClient.

## 1. Reconocimiento y Enumeración
Se realizó un escaneo inicial con Nmap para identificar los servicios expuestos. Se descubrió un Controlador de Dominio (DC) de Windows Server 2019 con el dominio `vulnnet-rst.local`.
```bash
nmap -sV -sC <IP>
