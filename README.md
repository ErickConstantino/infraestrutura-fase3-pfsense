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

---

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
8. **Description:** Permitir tráfego da LAN para serviços Core do Samba4 DMZ.
9. Salve e clique em **Apply Changes**.

*Pronto! Agora a sua rede local já consegue autenticar no Domínio de forma segura através do firewall.*

---

### 2. Servidor de Arquivos (File Server) no Linux e Permissões POSIX ACL

Com a rede se comunicando de forma segura, vamos criar nosso Servidor de Arquivos departamental. O objetivo aqui é aplicar o "Princípio do Privilégio Mínimo": o RH não pode ver os arquivos da Diretoria, e vice-versa.

#### Passo 2.1: Criando a Estrutura de Pastas
Acesse o terminal do seu servidor Debian (Samba4) e crie a estrutura raiz e as pastas dos departamentos:
```bash
mkdir -p /srv/samba/arquivos_corporativos
cd /srv/samba/arquivos_corporativos
mkdir Diretoria RH Financeiro
```

#### Passo 2.2: O Segredo da Herança (vfs objects)
Para que o Windows entenda perfeitamente as permissões do Linux, precisamos ativar os atributos estendidos (POSIX ACLs) no Samba. Edite o arquivo principal:
```bash
nano /etc/samba/smb.conf
```
Dentro da configuração do seu compartilhamento (ex: `[Arquivos]`), adicione:
```ini
vfs objects = acl_xattr
map acl inherit = yes
store dos attributes = yes
```
Salve o arquivo e reinicie o serviço: 
```bash
systemctl restart samba-ad-dc
```

#### Passo 2.3: Aplicando as Permissões via Console
Vamos usar o `setfacl` para atrelar os grupos de segurança do Active Directory diretamente às pastas no Debian:
```bash
# Exemplo para a pasta da Diretoria:
chmod 770 /srv/samba/arquivos_corporativos/Diretoria
setfacl -m g:"GG_Diretoria":rwx /srv/samba/arquivos_corporativos/Diretoria
```

---

### 3. Políticas de Grupo (GPO) no Windows

Agora que a fundação e as pastas estão prontas, vamos usar o RSAT no Windows 10 para criar políticas que automatizem as tarefas dos usuários.

#### Passo 3.1: Mapeamento Automático de Discos
1. Abra o **Gerenciamento de Política de Grupo**.
2. Crie uma GPO chamada `GPO_Mapeamento_Discos` e vincule à unidade organizacional dos usuários.
3. Edite a GPO: `Configurações do Usuário > Preferências > Configurações do Windows > Mapas de Unidade`.
4. Clique com o botão direito `Novo > Unidade Mapeada`.
5. Em **Local**, digite o caminho da rede (ex: `\\dc01\Arquivos\RH`).
6. Escolha uma letra (ex: `R:`) e marque a opção **Item de Nível de Destino** para aplicar essa unidade apenas se o usuário pertencer ao grupo "GG_RH".

#### Passo 3.2: Bloqueio do Painel de Controle
1. Crie uma GPO chamada `GPO_Seguranca_Desktop`.
2. Edite a GPO: `Configurações do Usuário > Políticas > Modelos Administrativos > Painel de Controle`.
3. Dê um duplo clique em **Proibir acesso ao Painel de Controle e às configurações do PC**.
4. Marque como **Habilitado** e aplique.

---

### 🐛 Troubleshooting Documentado: Problemas Reais que Enfrentamos

A teoria é linda, mas na prática as coisas quebram. Durante essa homologação, enfrentamos problemas críticos que você também pode encontrar:

#### 1. O "Default-Deny" do pfSense (Falha de Logon)
*   **Sintoma:** Após ligar o pfSense, as máquinas Windows na LAN não conseguiam autenticar ou validar o ticket Kerberos.
*   **Solução:** A DMZ bloqueia tudo por padrão. A solução foi criar as regras explícitas de aprovação (Pass) no pfSense para as portas TCP/UDP 53, 88, 389 e 445 da LAN para o IP do Servidor AD.

#### 2. O Crash do Windows Explorer (Aba Segurança)
*   **Sintoma:** Ao tentar acessar a pasta `\\dc01` pelo Windows, clicar com o botão direito na pasta de um departamento e ir na aba "Segurança", o Windows simplesmente surtava. Ele fechava a janela de propriedades e a pasta sozinhos, voltando direto para a Área de Trabalho sem dar nenhuma mensagem de erro.
*   **Solução:** Esse "crash" bizarro acontece porque o Windows tenta ler a lista de controle de acesso (ACL) do Linux e não consegue interpretar os metadados corretamente, causando uma falha no processo `explorer.exe`. Isso foi resolvido garantindo que o parâmetro `vfs objects = acl_xattr` estava perfeitamente configurado no `smb.conf` (conforme Passo 2.2) e reiniciando o serviço. O Samba passou a traduzir as permissões perfeitamente para o formato NT do Windows!
