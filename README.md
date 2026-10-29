# Windows Server

**Responsável técnica:** Alexandra Carneiro (Técnica de Redes e Segurança)

Um servidor Windows Server 2025 a funcionar como Domain Controller de um domínio Active Directory, com gestão de utilizadores e grupos, partilhas de ficheiros por departamento, perfis móveis, acesso remoto (RDS), atualizações centralizadas (WSUS) e backups agendados e manuais.

Ambiente de laboratório numa máquina virtual VMware.

## Ambiente

| Item | Valor |
| --- | --- |
| Hipervisor | VMware Workstation |
| Sistema operativo | Windows Server 2025 Standard, versão 24H2 (build 26100.32522) |
| Servidor | DC01 (renomeado de WIN-75MM01H238A) |
| IP | 192.168.80.100/24 (estático) |
| Gateway | 192.168.80.1 |
| DNS | 192.168.80.100 (o próprio servidor) |
| Domínio | iefp.internal |
| Roles | Active Directory Domain Services, DNS Server, File and Storage Services, Remote Desktop Services, WSUS |

## Domain Controller

- IP estático 192.168.80.100, gateway 192.168.80.1 e DNS próprio (192.168.80.100). Um Domain Controller não pode depender de DHCP, porque ele próprio é a referência de DNS da rede.
- Servidor renomeado de WIN-75MM01H238A para DC01.

<img src="imagens/02-renomear-dc01.png" width="700" alt="Renome do servidor para DC01">

*Renome do servidor de WIN-75MM01H238A para DC01.*
- Role Active Directory Domain Services instalado; domínio iefp.internal criado.
- Preparação da VM: VMware Tools instalado, Windows atualizado, 4 pacotes atualizados com `winget upgrade --all` (Visual C++ 2015-2022 Redistributable x86 e x64, App Installer e Windows Terminal) e snapshot "CLEAN STATE" criado no VMware antes das configurações seguintes.

<img src="imagens/01-preparacao-winget.png" width="700" alt="Terminal com winget upgrade --all">

*Atualização dos pacotes com `winget upgrade --all`: Visual C++ 2015-2022 (x86 e x64), App Installer e Windows Terminal.*

## Active Directory: OUs, grupos e utilizadores

Três unidades organizacionais (OUs), uma por departamento, cada uma com o seu grupo de segurança (âmbito Global):

| OU | Grupo | Utilizadores |
| --- | --- | --- |
| Administração | GRP_Administracao | Bruno Boss |
| Comercial | GRP_Comercial | Joao Negocios, Ana Propostas, Jose Descontos, Andre Clientes |
| Informática | GRP_TI | Nelson Debug, Guilherme Reboot, Diogo Script, Alexandra Root |

- Padrão de logon `nome.apelido` (por exemplo, `nelson.debug`), com a opção "User must change password at next logon" ativa.
- Cada utilizador foi associado ao grupo do seu departamento.
- Um cliente Windows 11 foi integrado no domínio iefp.internal, com autenticação centralizada.

Os nomes dos utilizadores são nomes de teste de laboratório.

## Perfis móveis (roaming profiles)

- Pasta `C:\Perfis` criada no servidor e partilhada para Authenticated Users.
- Profile Path configurado no AD: `\\dc01\perfis\%username%`.
- Verificação: a criação da pasta `alexandra.root.V6` no servidor confirma que o perfil acompanhou o utilizador.
- Se o servidor do perfil ficar indisponível, o utilizador entra com a cópia local em cache; num computador novo onde nunca entrou, pode receber um perfil temporário.

## Partilhas de ficheiros por departamento

Pastas criadas em `C:\Partilhas`: `Administracao`, `Comercial` e `Informatica`.

| Pasta | Permissão de partilha | Permissão NTFS |
| --- | --- | --- |
| Administracao | Authenticated Users, Full Control | GRP_Administracao com Modify |
| Comercial | Authenticated Users, Full Control | GRP_Comercial com Modify |
| Informatica | Authenticated Users, Full Control | GRP_TI com Modify |

A restrição por departamento é feita ao nível NTFS: ao nível da partilha, Authenticated Users tem Full Control nas três pastas; é o NTFS que limita cada grupo à sua pasta (Modify, sem Full Control).

Testes de acesso: um utilizador de cada grupo acedeu à pasta do seu departamento e criou lá um ficheiro de teste (Teste GRP_TI, Teste GRP_Comercial, Teste GRP_Administracao).

Drives mapeadas em todas as sessões: X: Informatica, Y: Comercial, Z: Administracao (`\\dc01`).

<img src="imagens/05-drives-mapeadas-xyz.png" width="700" alt="Drives mapeadas X, Y e Z">

*Drives mapeadas: X: Informatica, Y: Comercial e Z: Administracao (\\dc01).*

## Acesso remoto (RDS)

- Role Remote Desktop Services instalado no DC01, com as ferramentas de licenciamento RDS.
- Grupo GRP_TI autorizado na política "Allow log on through Remote Desktop Services" e adicionado ao grupo Remote Desktop Users.

<img src="imagens/06-rds-politica-grp-ti.png" width="700" alt="Política Allow log on through Remote Desktop Services">

*Política "Allow log on through Remote Desktop Services" com o grupo GRP_TI autorizado.*
- Políticas aplicadas com `gpupdate /force`.
- Validação: ligação RDP a 192.168.80.100 com credenciais de domínio, com duas sessões ativas em simultâneo no Task Manager (Administrator e nelson.debug).

## Atualizações centralizadas (WSUS)

- Windows Server Update Services configurado no DC01.
- Classificações sincronizadas: apenas Critical Updates, Definition Updates e Security Updates.
- Sincronização automática diária às 03:00.
- Uma sincronização manual concluída com 25 novas atualizações.

<img src="imagens/07-wsus-sincronizacoes.png" width="700" alt="Consola WSUS com as sincronizações">

*Consola WSUS: sincronizações manuais, uma delas com 25 novas atualizações.*

## Backups

- Volume C: reduzido em 10.000 MB no Disk Management.
- Novo volume "Backups Agendados (M:)", 9,76 GB, NTFS com compressão de ficheiros.
- Backup agendado: diário às 03:00, VSS Copy Backup, da pasta `C:\Administrator` para o volume M:.
- Backup urgente manual: pasta `C:\ProgramData` para o volume "Backups Urgentes (U:)".
- Restauro de validação: dados restaurados para a localização alternativa `C:\Urgente`.
- Backup e restauro por linha de comandos com wbadmin: `wbadmin start backup -backupTarget:U: -include:"C:\Secrets" -quiet` e `wbadmin get versions` (duas versões listadas, 13/04/2026 às 11:40 e 11:54). Restauro de `C:\Secrets` para `C:\Socorro` concluído.

<img src="imagens/08-wbadmin-restauro-socorro.png" width="700" alt="Restauro por wbadmin">

*Restauro por wbadmin: "Successfully recovered C:\Secrets to C:\Socorro", 2 ficheiros recuperados, 0 com falha.*

## Relatório final

O relatório final do trabalho descreve a infraestrutura no contexto de uma empresa fictícia (PixelForge Studios, domínio pixelforge.internal, departamentos Programação, Arte e Produção). Está no repositório como `Avaliacao-Relatorio-final.pdf`.

## Ficheiros

- `1-Preparacao-da-VM.pdf` - VMware Tools, atualizações, winget e snapshot
- `2-Domain-Controller.pdf` - IP estático, renome do servidor e AD DS
- `3-Active-Directory.pdf` - OUs, grupos, utilizadores e junção ao domínio
- `3b-Perfis-moveis.pdf` - roaming profiles
- `4-Partilhas-de-ficheiros.pdf` - partilhas, permissões e mapeamento
- `5-Acesso-remoto-RDS.pdf` - Remote Desktop Services
- `5b-WSUS.pdf` - Windows Server Update Services
- `6-Backups.pdf` - volume de backups, backup agendado e wbadmin
- `Avaliacao-Relatorio-final.pdf` - relatório final

## Notas

- Ambiente de laboratório. Os nomes dos utilizadores e dos departamentos são de teste.
- O domínio documentado nos ficheiros é iefp.internal; o relatório final usa o domínio pixelforge.internal no contexto da empresa fictícia.


---

## 👩‍💻 Autoria

**Alexandra Carneiro**  
*Graduanda em Engenharia de Software | Técnica Especialista em Cibersegurança & Redes*

- **GitHub:** [@alexandra-carneiro](https://github.com/alexandra-carneiro)  
- **LinkedIn:** [carneiro-alexandra](https://www.linkedin.com/in/carneiro-alexandra/)  
- **Portfólio:** [alexandracarneiro.tech](https://alexandracarneiro.tech/)
