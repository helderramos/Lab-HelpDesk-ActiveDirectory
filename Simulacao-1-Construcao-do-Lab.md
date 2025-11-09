# Relatório de Laboratório: Implantação de Active Directory e Simulação de Help Desk  


## 1. Resumo do Projeto

Este projeto foi a minha primeira imersão prática num ambiente corporativo simulado. O objetivo foi construir um laboratório de Active Directory (AD) do zero para praticar e documentar as três tarefas fundamentais de um Analista de Suporte Nível 1 (Help Desk).

O laboratório consiste num Servidor ("Chefe") e num Cliente ("Funcionário") numa rede isolada, onde executei simulações de gestão de identidade, acesso e segurança.

## 2. Componentes e Topologia

| Componente | Software | Hostname | Papel | Endereço IP | DNS Server |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Servidor** | Windows Server 2022 | `SRV-DC01` | DC / DNS Server | `10.0.0.10` | `127.0.0.1` |
| **Cliente** | Windows 11 Enterprise | `WKS-01` | Estação de Trabalho | `10.0.0.20` | `10.0.0.10` |
| **Virtualizador** | VirtualBox 7.x | N/A | Host | N/A | N/A |
| **Rede** | VirtualBox `Rede Interna` | `intnet` | Switch Virtual Isolado | N/A | N/A |

---

## 3. Fase 1: A Construção e os Problemas

A primeira fase foi construir o laboratório. Configurei as duas VMs na `Rede Interna` (`intnet`) e defini os IPs estáticos. O Cliente (`WKS-01`) foi configurado para usar o Servidor (`SRV-DC01`) como seu único **Servidor DNS** (`10.0.0.10`).

### Desafio 1: O Ping Falhou

Ao tentar testar a conectividade com `ping 10.0.0.10`, recebi `Request timed out`.

<p align="center">
<img width="556" height="195" alt="unnamed" src="https://github.com/user-attachments/assets/7a9cf9bc-76ec-46a5-9c11-33f350fb2d1b" />
</p>

**Como eu resolvi (Troubleshooting):**
1.  **Verifiquei os IPs:** Usei `ipconfig` em ambas as máquinas. Os IPs estavam corretos (`.10` e `.20`).
2.  **Verifiquei a Rede:** Conferi no VirtualBox se as duas VMs estavam na rede `intnet`. Estavam.
3.  **O Problema:** O **Firewall do Windows Server**.
4.  **O Teste:** Desativei o firewall no `SRV-DC01` e o `ping` funcionou imediatamente.

Após isto, promovi o `SRV-DC01` a Controlador de Domínio (usando `Add Roles and Features`) e criei o meu domínio: **`MeuLab.local`**.

### Desafio 2: Ingressar no Domínio

Ao tentar fazer o `WKS-01` entrar no domínio, ele falhou de novo: `The specified domain... could not be contacted.`
<p align="center">
<img width="393" height="177" alt="unnamed (1)" src="https://github.com/user-attachments/assets/fb01fb33-4db7-4234-9e60-fcedb3a43e9b" />
</p>

**Como eu resolvi:**
Era o mesmo problema! O Firewall do Servidor estava a bloquear os serviços que o AD usa (DNS, LDAP, etc.). Ao promover o servidor, um novo perfil de firewall ("Domain") foi ativado.
1.  Desativei os 3 perfis de firewall no `SRV-DC01` (Domain, Private, Public).
2.  Na `WKS-01`, tentei ingressar de novo, mas usei o utilizador correto: **`MEULAB\Administrator`** (e não o utilizador local).
3.  Funcionou! A máquina reiniciou e "entrou" na empresa.


---

## 4. Fase 2: Simulação de Help Desk (Tarefas 1, 2 e 3)

Com o laboratório funcional, executei um cenário do mundo real: integrar novos funcionários e aplicar políticas de segurança.

### Simulação 1: Gestão de Identidade (Reset de Senha)

* **Tarefa:** Pratiquei a tarefa mais comum do Help Desk: resetar a senha de um utilizador.
* **Ação:**
    1.  Abri a ferramenta `Active Directory Users and Computers` (`dsa.msc`).
    2.  Encontrei o meu utilizador de teste (`helder.ramos`).
    3.  Cliquei com o botão direito -> `Reset Password...`.
    4.  Defini uma senha temporária e, o mais importante, marquei a caixa **`User must change password at next logon`** (O utilizador deve alterar a senha no próximo logon).
* **Resultado:** Ao fazer login na `WKS-01` com a senha temporária, o sistema obrigou-me imediatamente a criar uma nova senha secreta. Tarefa concluída com sucesso.

### Simulação 2: Gestão de Acesso (Pastas e Grupos - RBAC)

* **Tarefa:** Dar ao novo departamento de Marketing (`ana.silva`) acesso à sua pasta de rede (`\\SRV-DC01\Marketing`).
* **Método Incorreto:** Dar permissão diretamente ao utilizador `ana.silva`.
* **Método Profissional (O que eu fiz):** Usei o RBAC (Controle de Acesso Baseado em Função). O fluxo é: **Utilizador -> Grupo -> Pasta**.
    1.  **Criei o Grupo:** Em `dsa.msc`, criei um Grupo de Segurança chamado `G_Marketing_Usuarios`.
    2.  **Adicionei o Membro:** Adicionei a utilizadora `ana.silva` como membro deste grupo.
    3.  **Configurei as Permissões (As 2 Camadas):**
        * **Camada 1 (Partilha/Sharing):** Na aba `Sharing` da pasta, removi o `Everyone` (Todos) e adicionei `Authenticated Users` (Utilizadores Autenticados).
        * **Camada 2 (Segurança/NTFS):** Na aba `Security`, adicionei o grupo `G_Marketing_Usuarios` e dei-lhe permissão de `Modify` (Modificar) — seguindo o Princípio do Menor Privilégio (não dando `Full Control`).
* **Lição Aprendida:** Tive de fazer **Log Off** e **Log On** na `WKS-01` com a `ana.silva` para que a máquina "atualizasse o crachá" e descobrisse que ela pertencia ao novo grupo. Após isso, o acesso funcionou perfeitamente.

### Simulação 3: Hardening de Endpoint (Política de Grupo - GPO)

* **Tarefa:** Aplicar uma regra de segurança que impedisse o departamento de Marketing de usar ferramentas perigosas (neste caso, o `regedit.exe` ou `cmd.exe`).
* **A "Armadilha" (Lição Aprendida):** Eu não podia aplicar uma GPO à pasta `Users` padrão. Tive de ser organizado.
* **A Solução (Estrutura de OUs):**
    1.  Em `dsa.msc`, criei uma Unidade Organizacional (OU) principal `Minha_Empresa` e, dentro dela, a OU `Departamento_Marketing`.
    2.  Movi a utilizadora `ana.silva` para dentro da `Departamento_Marketing`.
* **O Método Profissional (GPO):**
    1.  Abri a ferramenta `Group Policy Management` (`gpmc.msc`).
    2.  Criei uma nova GPO chamada `GPO_Marketing_Restricoes`.
    3.  Editei a GPO e fui para `User Configuration` -> `Policies` -> `Administrative Templates` -> `System`.
    4.  Encontrei e ativei (coloquei como `Enabled`) a regra **`Prevent access to the command prompt`** (ou `Prevent access to registry editing tools`).
    5.  "Linkei" (vinculei) esta nova GPO diretamente à **OU `Departamento_Marketing`**.
* **O Teste Final:**
    1.  Fiz login na `WKS-01` como `ana.silva`.
    2.  Abri o `cmd` e executei o comando mágico: **`gpupdate /force`** (para forçar o cliente a "ler" as novas regras do servidor).
    3.  Fiz Log Off e Log On novamente.
    4.  Ao tentar abrir o `cmd` (ou `regedit`), fui bloqueado pela mensagem do administrador.

## 5. Conclusão Final do Projeto

Este laboratório foi finalizado com sucesso. Consegui construir um ambiente Active Directory funcional e, mais importante, aprendi a **diagnosticar e resolver problemas** do mundo real (Firewall, DNS, Permissões). Pratiquei as 3 tarefas centrais de um Analista de Suporte (Identidade, Acesso e Hardening).
