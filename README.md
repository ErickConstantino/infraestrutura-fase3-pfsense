# 🛡️ Fase 3: Governança, Storage e Segurança com pfSense e Samba4

Nesta terceira fase do nosso laboratório de Infraestrutura Corporativa, vamos pegar o Domínio criado na Fase 2 e protegê-lo de verdade. Se você é iniciante, não se preocupe: vou explicar o "porquê" de cada comando antes de executarmos.

## 📋 Pré-requisitos
Antes de começar, você precisa ter concluído:
- [Fase 1: Configuração do Debian Base](https://github.com/ErickConstantino/infraestrutura-corporativa-linux)
- [Fase 2: Instalação do Samba4 AD DC](https://github.com/ErickConstantino/samba4-ad-debian)

## 📚 Índice
1. **Entendendo a Topologia:** O que é uma DMZ e por que precisamos do pfSense?
2. **Instalação e Configuração do pfSense:**
   - Adicionando as placas de rede virtuais (LAN e WAN).
   - Liberando as portas do Active Directory no Firewall.
3. **Servidor de Arquivos (File Server) no Linux:**
   - Criando pastas para Diretoria, RH e Financeiro.
   - O que são permissões POSIX ACL e como configurá-las.
4. **Políticas de Grupo (GPO) no Windows:**
   - Mapeando discos de rede automaticamente.
   - Bloqueando o Painel de Controle dos usuários.
5. **Troubleshooting:** Como resolvi os problemas de bloqueio de rede e permissões fantasmas.
## 🚀 Passo a Passo da Implementação

### 1. Entendendo a Topologia e Configurando o pfSense

Na Fase 2, nosso servidor Samba e os clientes Windows estavam na mesma rede. Agora, vamos separá-los para aumentar a segurança. O servidor ficará na **DMZ** (Zona Desmilitarizada) e os clientes na **LAN**. O pfSense fará a ponte e o bloqueio entre eles.

#### Passo 1.1: Configurando as Interfaces no Hypervisor
Antes de ligar as máquinas, garanta que no seu virtualizador (VirtualBox/Proxmox) as placas estejam isoladas:
1. **pfSense:** Terá 3 placas (WAN para internet, LAN para clientes, DMZ para servidores).
2. **Debian (Samba4):** Placa de rede conectada APENAS na rede DMZ.
3. **Windows 10:** Placa de rede conectada APENAS na rede LAN.

#### Passo 1.2: Criando Aliases no pfSense (O Segredo da Organização)
Para não criarmos dezenas de regras soltas, vamos agrupar os dados do nosso Domínio. Acesse a interface web do pfSense e vá em `Firewall > Aliases`.

**A) Criando o Alias de IP:**
1. Clique em **Add**.
2. **Name:** `AD_SERVER_IP`
3. **Type:** Host(s)
4. **IP or FQDN:** Digite o IP estático do seu servidor Debian.
5. Salve.

**B) Criando o Alias de Portas:**
1. Clique em **Add** novamente.
2. **Name:** `AD_SERVICES_PORTS`
3. **Type:** Port(s)
4. Adicione as seguintes portas vitais para o domínio funcionar:
   - `53` (DNS - TCP/UDP)
   - `88` (Kerberos - TCP/UDP)
   - `389` (LDAP - TCP/UDP)
   - `445` (SMB/CIFS - TCP)
5. Salve e clique em **Apply Changes**.

#### Passo 1.3: Criando as Regras de Liberação (Firewall Rules)
Por padrão, a rede LAN não fala com a DMZ. Precisamos abrir uma exceção apenas para os serviços do Active Directory.
1. Vá em `Firewall > Rules > LAN`.
2. Clique no botão de adicionar (`Add` com a seta para cima).
3. **Action:** Pass
4. **Protocol:** TCP/UDP
5. **Source:** LAN net
6. **Destination:** Single host or alias -> digite `AD_SERVER_IP`
7. **Destination Port Range:** Em *from* e *to*, digite `AD_SERVICES_PORTS`.
8. **Description:** "Permitir tráfego da LAN para serviços Core do Samba4 DMZ".
9. Salve e clique em **Apply Changes**.

*Pronto! Agora a sua rede local já consegue autenticar no Domínio de forma segura através do firewall.*
