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
