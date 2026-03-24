# Lab 1 — Firewall & Active Directory

![Status](https://img.shields.io/badge/status-completed-brightgreen)
![Platform](https://img.shields.io/badge/platform-VirtualBox-blue)
![OS](https://img.shields.io/badge/OS-pfSense%20%7C%20Windows%20Server%202022%20%7C%20Windows%2010-lightgrey)

## Objetivo

Simular um ambiente corporativo real com firewall de borda, segmentação de rede e domínio Windows Active Directory — tudo rodando em máquinas virtuais no VirtualBox. O objetivo foi entender na prática como funciona a infraestrutura de TI de uma empresa, antes de partir para os labs de ataque e detecção.

---

## O que você aprende

- Como um firewall de borda funciona na prática (pfSense)
- Segmentação de rede com interfaces WAN e LAN
- Configuração de DHCP e DNS internos
- Active Directory Domain Services (AD DS)
- Criação de usuários, grupos e políticas de grupo (GPO)
- Como ingressar uma estação de trabalho em um domínio Windows
- Monitoramento de endpoints com Sysmon (config SwiftOnSecurity)

---

## Topologia de Rede

```
                    Internet
                       │
               ┌───────┴───────┐
               │   pfSense     │  WAN: 10.0.2.15 (NAT)
               │  192.168.1.1  │  LAN: 192.168.1.1/24
               └───────┬───────┘
                       │
          ─────────────┴─────────────
          │                         │
┌─────────┴──────────┐   ┌──────────┴─────────┐
│   Windows Server   │   │    Windows 10       │
│       DC01         │   │     Cliente         │
│   192.168.1.10     │   │   192.168.1.102     │
│   AD DS + DNS      │   │  Domínio: lab.local │
└────────────────────┘   └────────────────────┘
```

---

## Tabela de VMs

| VM | Sistema Operacional | RAM | CPU | Disco | Função |
|----|-------------------|-----|-----|-------|--------|
| pfSense | pfSense CE 2.8.1 | 512 MB | 1 | 10 GB | Firewall / Gateway |
| Windows Server | Windows Server 2022 Standard Evaluation | 4 GB | 2 | 60 GB | AD DS + DNS + DHCP |
| Windows 10 | Windows 10 64-bit | 3 GB | 2 | 50 GB | Estação do domínio |

---

## Pré-requisitos

**Hardware:**
- 16 GB de RAM (mínimo — não rode todas as VMs ao mesmo tempo)
- SSD recomendado (impacto enorme no tempo de boot das VMs)
- Processador com suporte a virtualização (Intel VT-x ou AMD-V habilitado na BIOS)

**Software:**
- [VirtualBox 7.x](https://www.virtualbox.org/wiki/Downloads) + Extension Pack
- [pfSense CE](https://www.netgate.com/downloads) — ISO gratuito
- [Windows Server 2022 Evaluation](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022) — trial de 180 dias
- [Windows 10](https://www.microsoft.com/pt-br/software-download/windows10) — via ferramenta de criação de mídia

---

## Configuração

### pfSense

1. Criar VM com 2 adaptadores de rede: **Adaptador 1 → NAT** (WAN) e **Adaptador 2 → Rede Interna "LAN"** (LAN)
2. Na instalação, selecionar **Install CE** (versão gratuita)
3. Atribuir interfaces: `em0 → WAN` e `em1 → LAN (192.168.1.1/24)`
4. Acessar a WebGUI pelo navegador de outra VM: `https://192.168.1.1` (admin / pfsense)
5. Configurar DHCP Server: range `192.168.1.100` a `192.168.1.200`

### Windows Server — Active Directory

1. Criar VM com adaptador em **Rede Interna "LAN"**
2. Configurar IP fixo: `192.168.1.10` | Gateway: `192.168.1.1` | DNS: `127.0.0.1`
3. Renomear o servidor para `DC01`
4. No Server Manager: **Add Roles → Active Directory Domain Services → Install**
5. Promover a Domain Controller: **Add a new forest → lab.local**
6. Criar usuários de teste em **Active Directory Users and Computers**
7. Instalar Sysmon com config SwiftOnSecurity (ver seção abaixo)

### Windows 10 — Estação do Domínio

1. Criar VM com adaptador em **Rede Interna "LAN"**
2. Configurar DNS manualmente para `192.168.1.10` (obrigatório para encontrar o domínio)
3. Ingressar no domínio: **Configurações avançadas do sistema → Nome do Computador → Alterar → Domínio: lab.local**
4. Reiniciar e fazer login com usuário de domínio
5. Instalar Sysmon com config SwiftOnSecurity (ver seção abaixo)

### Sysmon (Windows Server e Windows 10)

```powershell
# Baixar Sysmon
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "C:\Sysmon.zip"

# Extrair
Expand-Archive C:\Sysmon.zip -DestinationPath C:\Sysmon

# Baixar config SwiftOnSecurity
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "C:\Sysmon\config.xml"

# Instalar
C:\Sysmon\Sysmon64.exe -accepteula -i C:\Sysmon\config.xml

# Verificar
Get-Service Sysmon64
```

---

## Screenshots

| # | Descrição |
|---|-----------|
| [01](./screenshots/01-pfsense-wan-lan.jpg) | pfSense — interfaces WAN e LAN configuradas |
| [02](./screenshots/02-ad-usuarios.jpg) | Active Directory — usuários do domínio lab.local |
| [03](./screenshots/03-win10-login-dominio.jpg) | Windows 10 — tela de login com usuário de domínio |
| [04](./screenshots/04-sysmon-running-server.jpg) | Sysmon rodando no Windows Server |
| [05](./screenshots/05-sysmon-running-win10.jpg) | Sysmon rodando no Windows 10 |

---

## O que deu problema (e o que resolveu)

**VM bootando pelo ISO após instalação do pfSense**
Depois de instalar o pfSense e reiniciar, a VM voltou para o instalador. O problema era que o ISO ainda estava montado na unidade virtual. Solução: remover o ISO nas configurações de armazenamento do VirtualBox antes de reiniciar.

**pfSense com apenas uma interface de rede**
Na criação da VM, só foi adicionado um adaptador de rede. O instalador do pfSense só mostrava `em0` e não tinha como configurar WAN e LAN. Solução: adicionar o segundo adaptador (Rede Interna) nas configurações da VM antes de instalar.

**VMs não se comunicando apesar de estarem na mesma rede interna**
Windows Server e Windows 10 conseguiam pingar o pfSense mas não se pingavam entre si. O problema estava no firewall do Windows bloqueando o tráfego ICMP no perfil Public — perfil usado enquanto a máquina ainda não está em domínio. Solução: desativar o firewall temporariamente para diagnóstico, confirmar a comunicação e reativar após ingressar no domínio.

**Windows 10 não encontrava o domínio lab.local**
O erro "Não foi possível contatar um Controlador de Domínio" aparecia ao tentar ingressar no domínio. O DNS do Windows 10 estava apontando para o pfSense (192.168.1.1) em vez do Windows Server (192.168.1.10). O pfSense não conhece o domínio lab.local, então a resolução falhava. Solução: configurar o DNS manualmente para 192.168.1.10.

**Sysmon não instalando por falta do config.xml**
O comando de instalação foi executado antes do download do arquivo de configuração terminar. O Sysmon retornou erro de arquivo não encontrado e ficou parcialmente instalado. Solução: verificar se o arquivo existe com `dir C:\Sysmon\`, desinstalar com `Sysmon64.exe -u force`, reiniciar a VM e reinstalar.

---

## Referências

- [pfSense CE Downloads](https://www.netgate.com/downloads)
- [Windows Server 2022 Evaluation](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022)
- [Sysmon — Sysinternals](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [SwiftOnSecurity Sysmon Config](https://github.com/SwiftOnSecurity/sysmon-config)
