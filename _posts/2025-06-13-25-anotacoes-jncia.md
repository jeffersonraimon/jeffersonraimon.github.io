---
title: "25 | Anotações JNCIA"
date: 2025-06-13 00:00:00 -0300
categories: [Artigos]
tags: [Juniper, JNCIA, Networking, Estudos]
image:
  path: assets/img/posts/post-25/capa.jpg
---

Estive estudando para a certificação JNCIA – 2025, através do curso disponível no [Juniper Learning Portal](https://learningportal.juniper.net/juniper/user_activity_info.aspx?id=12035)
Neste post, reunirei minhas anotações dos módulos do curso, focando nos comandos, seus significados e aplicação prática da teoria passada no curso. O objetivo é manter um registro simples e direto, sem aprofundamentos excessivos.

OBS: O voucher é apenas para a prova online e ao fazer o exame do curso, tem até 30 dias para fazer a prova. Então faça o exame quando tiver tempo e dinheiro.

A prova são 65 questões e é necessário acertar 63% para aprovação

<https://www.juniper.net/br/pt/training/certification/tracks/junos/jncia-junos.html>

Outra ótima fonte de estudo é o JNCIA Study Guide Part 1 e 2 que pode ser encontrado na internet.

## Junos OS

- Sistema operacional dos dispositivos de rede da Juniper
- Roda em todos os equipamentos: roteadores, switchs e firewalls, desde pequenos a roteadores de datacenter
- Praticamente todo o sistema é igual para os dispositivos, com algumas pequenas mudanças a depender da plataforma (Algumas builds são especificas de cada plataforma)
- Recursos novos são consistentes em atualizações maiores

É possivel gerenciar os dispositivos com:

- CLI
- WEB (algumas plataformas, como firewall)
- Linguagem de programação (Off-Box Scripts. Ex: Python, com a biblioteca criada pela Juniper, PyEZ)
- Controladores externos (Plataformas da Juniper: Mist, Apstra, API ou Ansible para plataformas de terceiros)
- Codigos (On-Box Scripts – scripts salvos na caixa e rodando automaticamente)

O Junos é voltado e incentivado para automação, oferecendo recursos de graça para isso.

![](assets/img/posts/post-25/01.png)

### Versões de Releases

Exemplo: 22.4R1.10

22: Ano

22.4: Major release

R1: Feature release (+bugs fixes) R2,R3,RX 

MR: Maintence Release (bug fixes basicamente)

S1, S2, SX: Service release (SR – lançado antes do MR para corrigir bugs criticos ou caso especificos de usos de clientes)

10: Numero interno da build

Hoje em dia, a Juniper lança duas Major Releases por ano.

### Dispositivos que rodam Junos

Duas categorias: Fixed-port Devices e Modular Devices

![](assets/img/posts/post-25/02.png)

Fixed-Ports: Chassis que já tem uma quantidade exata de portas de fabrica. Exemplo: MX204

- 4 portas QSFP (100GB)
- 8 portas SFP+ (10GB)
- Portas de gerência outbound e console

![](assets/img/posts/post-25/03.png)

Modular Devices: Chassis que ja tem algumas portas mas é possivel adicionar slots (line cards). Exemplo: MX480

MX480

![](assets/img/posts/post-25/04.png)

Sendo possivel customizar com slots de quantidades de portas dependendo da necessidade

Line Cards

Possui tipos: Line Card de interfaces de redes e Routing Engine (o cerebro da caixa)

Line Card de interfaces de redes:

- Podem ser slots de trafego, recursos de segurança etc.
- Cada line card tem um numero e será refletido no CLI
- Tambem possui um Line Card CPU (para comunicar com o RE e gerenciar o PFE)

![](assets/img/posts/post-25/05.png)

Line Card de Routing Engine:

![](assets/img/posts/post-25/06.png)

FPC e PIC

FPC = Flexible PIC Concentrator (o line card em si)
PIC = Physical Interface Card

![](assets/img/posts/post-25/07.png)

## Produtos Juniper

Roteadores:

- MX (faz muita coisa, bastante usado em bordas ou como BNG/BRAS)
- PTX (mais focado em encaminhamento (extremamente rápido), e usado em redes grandes, mais utilizado no CORE da rede)
- ACX (mais usado no endpoint, tem varios tipos de conexões fisicas para concentrar os links)

Firewalls:

- SRX (hardware dedicado para filtragem de trafego e firewall de proxima geração)
   - Equipamento integrado com ATP (anti-malware, traffic insights, threat profing e DNS security)

Switches:

- EX
- QFX

Wireless

Virtual:

- Imagens para virtualização (pode ser usado em cloud, Virtual labs (EVE, GNS3, PNETLAB)

MistAI

Poderosa plataforma de gerênciamento de dispositivos da Juniper (usado bastante em Switches EX)

Oferece:

- Visibilidade, controle e automação
- Aprende e monitora o trafego de rede
- GUI
- Templates
- AI (machine learning)
- Arruma problemas antes de sabermos que existem (VLANS, WIFI, Portas problematicas etc)
- Assistente virtual de rede (Marvis) – Pode monitorar, controlar e manter a rede (LAN, WAN,SD-WAN e WIRELESS)

## Arquitetura do Junos OS

De forma resumida, a arquitetura do Junos é modular, ou seja, um processo não interfere em outro (memória protegida). Também o plano de controle é separado do plano de encaminhamento.

Routing Engine (Plano de controle) – RE

É a CPU da caixa. x86 based

Lida com:

- Junos OS em si
- Monitoramento, gerenciamento, processos, sistema, chassis, protocolos etc
- Gerência do PFE
- Manutenção das tabelas de roteamento (RIB) e de encaminhamento (FIB)
- Responde ping e traceroute (exceptions traffic)

Em termos de roteamento e encaminhamento, o RE fica responsável em escolher as rotas ativas (melhores rotas para os prefixos), instalar na tabela de encaminhamento e escrever no hardware dedicado de encaminhamento

A tabela de encaminhamento só tem as informações necessárias para realizar a tarefa.

- Interface de saida
- IP e MAC do proximo salto
- Type (dest, intf, perm, user) 
dest = remoto, intf = instalado, perm = permanente, user = criado por protocolo de roteamento/usuario
- Types Next-hop (unicast, discard, broadcast etc)

RIB
```
root@ROUTER_2> show route 
```
FIB
```
root@ROUTER_2> show route forwarding-table
```
Verificar informações de CPU, memoria, etc
```
raimon> show chassis routing-engine 
```
Packet Forwarding Engine (Plano de encaminhamento) – PFE

Conjunto de hardware dedicado para encaminhamento de pacotes.

É um chip chamado ASIC (Application-specific Integrated Circuit)

Tipos:

Express chip – PTX
Trio Chip – MX

Cada ASICs tem sua propria copia da FIB

Tarefas:

- Encaminhar tráfego
- Manipular pacotes (Cabeçalho Ethernet, Tags de VLAN, decrementar TTL e Hop-Limit)
- Controlar serviços que impactam trafego (rate-limiting, firewall filters, priorização de tráfego etc)
- Reportar contador de tráfego para o RE
- Enviar as exceptions traffic para o RE

### Separação dos Planos

A Juniper é pioneira nesse tipo de arquitetura

Vantagens:

- Se o RE crashar, o PFE continha encaminhando tráfego
- Cada hardware pode ser otimizado para realizar suas tarefas
- Cada hardware pode ser atualizado independentemente um do outro
- O Junos adiciona rate-limiting e Proteção de DDoS no link interno entre RE e PFE (tem preferência com congestionamentos)

![](assets/img/posts/post-25/08.png)
![](assets/img/posts/post-25/09.png)

### Exception Traffic

Todo trafego que é destinado a caixa (host-inbound) ou da caixa para outro destino (host-outbound)

Exemplo:

- SSH/Telnet
- Ping/Traceroute
- Mensagens de protocolos (OSPF, BGP etc)
- ARP/NDP
- SNMP
- Pacotes IP com IP Options
- Trafegos que requerem mensagens ICMP

Pórem, alguns exceptions traffic pode ser processado pelo Line Card CPU ao inves de enviar para o RE, como por exemplo Time Exceeded ou mensagens de erro ICMP em geral

O que não é exceptions traffic:

Pacotes de transito (pacotes destinados a outros destinos ou processado apenas pelo PFE)

Pacotes de controle interno (pacotes de controle entre o RE e Line Card CPU do PFE)

### FreeBSD e Evolved

O Junos roda em cima do FreeBSD

2017 usava o freebsd 11
2021 em diante usa o freebsd 12

Para atualizar o Junos as vezes precisa também atualizar o FreeBSD

Para acessar enquanto está no modo de operação:
```
root@ROUTER_2> start shell
```

### Daemons

São os processos que rodam no sistema e são responsáveis por varios recursos da caixa.

routing, chassis, interfaces, management, snmp

*rpd = daemon de roteamento*

Controla o roteamento, mantém as tabelas de roteamento, determina as rotas ativas e as instala na cópia da tabela de encaminhamento da RE (Routing Engine).

*mgd = daemon de gerenciamento*

Controla o gerenciamento, atuando como interface entre os métodos de acesso (CLI, scripts etc.).

*dcd = daemon de controle do dispositivo*

Envia a configuração das interfaces físicas para as portas.

*chassid = daemon do chassi*

Controla o chassi, alarmes, fonte de alimentação elétrica, ventoinhas etc. Monitora a integridade do hardware, banco de dados de inventário, LEDs etc.

*ppmd = daemon de gerenciamento periódico de pacotes*

Permite que a CPU da Line Card (PFE – Packet Forwarding Engine) execute certas tarefas no lugar da RE.
Exemplos: pacotes OSPF hello, BPDUs, LACP, BFD etc.

Junos Evolved (EVO)

É o sistema rodando em cima de Linux, pode rodar num whitebox, integrar com apps de terceiros.

No EVO cada função são como aplicativos e não daemons

Utiliza banco de dados distribuido

## CLI

Baseado em hierarquias

Na verdade ao dar comandos no CLI, estamos usando a API XML para interagir com os processos e usa também o RPC (Remote Procedure Call)

![](assets/img/posts/post-25/10.png)

Exemplo de hierarquias:

- system
- chassis
- interfaces
- protocols

![](assets/img/posts/post-25/11.png)

Diferente de alguns fabricantes, as configurações feitas no CLI da Juniper é tornada ativa apenas mediante commit. Sendo uma vantagem para avaliar a configuração feita, e reverte-la caso necessário uma vez feito o commit (rollback).

O Junos salva as ultimas 49 configurações realizadas (commitadas), sendo a config 0 a ativa no momento

Modos de operação: Operacional mode e Configuration mode

### Operacional mode:

O que é possivel fazer

![](assets/img/posts/post-25/12.png)

Comandos mais utilizados nesse modo: SHOW , MONITOR , CLEAR , REQUEST

![](assets/img/posts/post-25/13.png)

Comandos uteis de show system:

```
root@ROUTER> show system information 
Model: vmx
Family: junos
Junos: 21.1R1.11
Hostname: ROUTER

root@ROUTER> show system uptime 
Current time: 2025-04-01 13:40:49 BRT
Time Source:  LOCAL CLOCK 
System booted: 2025-03-26 21:48:35 BRT (5d 15:52 ago)
Protocols started: 2025-03-26 21:52:18 BRT (5d 15:48 ago)
Last configured: 2025-04-01 13:40:41 BRT (00:00:08 ago) by root
 1:40PM  up 5 days, 15:52, 1 users, load averages: 0.59, 0.40, 0.38

root@ROUTER> show version and haiku (gera uma frase no final do output)
Hostname: ROUTER
Model: vmx
Junos: 21.1R1.11
JUNOS OS Kernel 64-bit  [20210308.e5f5942_builder_stable_11]
JUNOS OS libs [20210308.e5f5942_builder_stable_11]
...

        Another day dies                
        More joy than I can fathom
        Yet more tomorrow
``` 
Outros: show chassis hardware , show interfaces , show route

Output com pipeline

- match (mostra apenas textos que tenham o padrão definido)
- except (mostra apenas textos que não tenham o padrão definido)
- count (mostra a quantidade de linhas)
- find (mostra a primeira ocorrência do padrão definido)

Ex: 
```
root@ROUTER> show interfaces terse | match down 
ge-0/0/1                up    down
ge-0/0/1.16386          up    down

root@ROUTER> show configuration | match "description|address"     
        description INTERNET;
                address 172.16.10.254/24;

root@ROUTER> show interfaces terse | except down   
Interface               Admin Link Proto    Local                 Remote
ge-0/0/0                up    up
ge-0/0/0.0              up    up   inet     10.0.137.248/24 

root@ROUTER> show interfaces terse | count          
Count: 86 lines

root@ROUTER> show configuration | display set | find INTERNET    
set interfaces ge-0/0/0 description INTERNET
``` 

Dica: As vezes no CLI alguns comandos ficarão omitidos por conta dos numeros de caracteres. É possivel aumentar usando o seguinte comando no modo operacional (apenas válido para a sessão atual):

```
root@ROUTER-01> set cli screen-width 1000 
Screen width set to 1000
```
O padrão é 80, o máximo é 1024

mais em: <https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/getting-started.html>

## Lendo configurações do Junos

Duas formas comuns de verificar configurações no Junos são:

- Hierarchy View – Modo padrão. (mostra de forma identada e é melhor para visualizar configurações complicadas)

```
root@ROUTER> show configuration interfaces 
apply-groups DESCRIPTION-INTERFACES;
ge-0/0/0 {
    description INTERNET;
    unit 0 {
        family inet {
            dhcp;
            filter {
                output FW;
            }
        }
    }
```

- Set View (mostra na forma que foi feito a configuração)

```
root@ROUTER> show configuration interfaces | display set 
set interfaces apply-groups DESCRIPTION-INTERFACES
set interfaces ge-0/0/0 description INTERNET
set interfaces ge-0/0/0 unit 0 family inet dhcp
set interfaces ge-0/0/0 unit 0 family inet filter output FW
```
## Configurando o Junos

Junos verifica a sintaxe e rejeita erros

Para salvar as configurações realizadas, precisa fazer o commit

- Ao criar uma configuração, ela fica salva numa localização chamada candidate configuration, após commitar, ela vai para a active configuration.

O Junos mantem as 49 configurações feitas anteriormente

![](assets/img/posts/post-25/14.png)

Para visualizar e comparar:

```
root@ROUTER-01> show system commit 
0   2025-04-06 10:20:50 BRT by root via cli
1   2025-04-06 10:19:54 BRT by root via cli

root@ROUTER-01> show configuration | compare rollback 9    
[edit]
+ groups {
+     DESCRIPTION-INTERFACES {
```

Modo de configuração:

- Adicionar/Deletar configuração
- Mudar/Renomear coisas
- Mover/Copiar coisas
- Desativar coisas
- Usar “run” para rodar comandos do operational mode

Outros tipos de commits:

- commit check (verificar a sintaxe/possiveis problemas antes de realizar o commit de fato)
- commit and-quit (commitar e sair do modo configuração)
- commit confirmed X (para salvar gasolina, commitar e desfazer após X minutos caso nao realize o commit novamente para aplicar de fato)
- commit comment “TEXTO” (adicionar um comentário ao commit)
- commit at “2025-04-06 18:00:00”

Para entrar no modo de configuração, basta digitar edit ou configure

```
root@ROUTER> edit 
Entering configuration mode

root@ROUTER> configure 
Entering configuration mode
```

Exemplo de configuração e verificação do que foi feito (compare)

```
[edit]
root@ROUTER# set system host-name ROUTER-01 

[edit]
root@ROUTER# show | compare 
[edit system]
-  host-name ROUTER;
+  host-name ROUTER-01;

[edit]
root@ROUTER# commit comment "mudando o nome do roteador"
commit complete
```

Para desfazer o que foi feito no candidate configuration, basta usar o comando rollback que irá copiar o active configuration para o candidate.

```
[edit]
root@ROUTER-01# set system host-name TESTE 

[edit]
root@ROUTER-01# show | compare 
[edit system]
-  host-name ROUTER-01;
+  host-name TESTE;

[edit]
root@ROUTER-01# rollback 
load complete

[edit]
root@ROUTER-01# show | compare 

[edit]
root@ROUTER-01# 
```

Para desfazer uma configuração antiga e as outras subsequentes

[edit]
root@ROUTER-01# rollback 9 
load complete

[edit]
root@ROUTER-01# show | compare 
[edit groups]
-  DESCRIPTION-INTERFACES {
```

Dependendo do comando, o set pode adicionar ou modificar configurações

![](assets/img/posts/post-25/15.png)

Comando Delete

Apaga configurações, porém ele é hierarquico.

![](assets/img/posts/post-25/16.png)

Comandos ```disable/deactivate``` (e ```activate```)

```disable```

Útil para desabilitar interfaces e protocolos
```
[edit]
root@ROUTER-01# set interfaces ge-0/0/0 disable
[edit]
root@ROUTER-01# set protocols ospf disable
```

```deactivate/activate```

O Junos não vai processar configurações que estejam como deactivate
```
[edit]
root@ROUTER-01# deactivate firewall filter FW 

[edit]
root@ROUTER-01# activate firewall filter FW 
```

Comando ```Rename/Replace Pattern```

Alterar configurações diretamente:
```
[edit]
root@ROUTER-01# show | display set | match lo0     
set interfaces lo0 unit 0 family inet address 10.1.1.1/32

[edit]
root@ROUTER-01# rename interfaces lo0.0 family inet address 10.1.1.1/32 to address 10.2.2.2/32    
```
Replace Pattern (muda TUDO que encontrar com o padrão especificado)
```
[edit]
root@ROUTER-01# replace pattern ge-0/0/0 with ge-0/0/1
```

### Outros modos de candidate configuration:

configure exclusive

- Outros usuarios podem entrar no modo config porém não podem aplicar mudanças
- Ao sair sem commitar, as configurações feitas são descartadas, diferente do modo normal que mantem ainda no candidate

configure private

- Cria uma copia unica da candidate configuration
- Multiplos usuarios podem configurar diferentes hierarquias ao mesmo tempo
- Os candidates configuration são temporários e mudanças sem commits são perdidas ao sair do modo
- Tem avisos para evitar conflitos
- Só é possivel commitar no top level ([edit])

Configurando dentro de hierarquia (usando comando edit)
```
[edit]
root@ROUTER-01# edit firewall filter FW 

[edit firewall filter FW]
root@ROUTER-01# set term BLOCK_GAMING then reject 
```
Entrando e saindo de hierarquias
```
[edit firewall filter FW]
root@ROUTER-01# edit term BLOCK_GAMING    

[edit firewall filter FW term BLOCK_GAMING]
root@ROUTER-01# up 

[edit firewall filter FW]
root@ROUTER-01# exit 

[edit]
root@ROUTER-01# 
```
Omitir saida de configurações no show configuration (modo hierarquia)

apply-flags omit (comando escondido, nao completa com TAB)
```
set routing-options apply-flags omit
```
Para mostra: digitando a hierarquia toda ou displays
```
show configuration routing-options
show configuration | display omit
show configuration | display set
```
Proteger configuração contra alterações
```
[edit]
root# protect routing-options    

[edit]
root# show routing-options       
##
## protect: routing-options
##
router-id 10.1.1.1;
```
```unprotect``` para desfazer

Anotar mensagens em configurações
```
[edit]
root# annotate routing-options "NAO ALTERAR"

root> show configuration
/* NAO ALTERAR */
routing-options {
    router-id 10.1.1.1;
}
```
### Arquivos

Salvar saidas em arquivos (txt por exemplo)
```
raimon> show interfaces terse | save INTERFACES.txt  
Wrote 69 lines of output to 'INTERFACES.txt'

raimon> file show INTERFACES.txt                       
Interface               Admin Link Proto    Local                 Remote
ge-0/0/0                up    up
```
Mostrar arquivos no diretório do usuário:
```
raimon> file list 

/var/home/raimon/:
INTERFACES.txt
```
Comparar configuração com arquivos é so usar o ```| compare NOME-ARQUIVO```

Comparar configurações em 2 arquivos diferentes
```
raimon> file compare files ROUTER-ID1.txt ROUTER-ID2.txt         
1c1
< router-id 10.1.1.1;
---
> router-id 10.2.2.2;
```
< – primeiro arquivo | > – segundo arquivo

Numeros de linhas: a – adição, c – mudança, d – subtração

Adicionar saida no final de um arquivo
```
raimon> show ospf neighbor | append OSPF_NEI.txt
```
Enviando e recebendo arquivos via FTP/SCP

![](assets/img/posts/post-25/17.png)

## Carregando configurações

Colando configuração hierarquica (CTRL+D para terminar)
```
[edit]
raimon# load override terminal 
[Type ^D at a new line to end input]
```
Esse comando substitui todas as configurações candidatas atuais

Para juntar configurações (algumas configs são substituidas e outras adicionadas)
```
[edit]
raimon# load merge terminal [relative]
```
A opção ```relative``` é para uma configuração mais profunda na hierarquia

Também é possivel carregar configurações de arquivos: no lugar de terminal, coloca-se o nome do arquivo

Colando configuração SET
```
[edit]
raimon# load set terminal 
[Type ^D at a new line to end input]
```
Enviar configurações automaticamente para servidor externo

![](assets/img/posts/post-25/18.png)


### Configurações automáticas para hierarquias

![](assets/img/posts/post-25/19.png)
![](assets/img/posts/post-25/20.png)

Usando ```groups``` é possivel definir configurações que replica para todas as interfaces, conforme exemplo acima

Para visualizar, deve-se usar o ```display inheritance```

Para uma interface que não quer essa mudança basta colocar o comando ```apply-groups-except```

Configuração de Ranges

![](assets/img/posts/post-25/21.png)

Apagando muitas configurações de vez usando Wildcard delete

![](assets/img/posts/post-25/22.png)

## Administração do Dispositivo

Management Interface

Separar trafego de usuario com o de gerencia da CX

Interface MGMT Outbound
fxp0 – MX ou me0 (copper port – Switches) , re0:mgmt-0 – PTX
Adiciona IP igual outras interfaces

Configurar Hora e Data

Manual
```
{master:0}[edit]
root@vqfx-re# set system time-zone America/Sao_Paulo 

{master:0}
root@vqfx-re> set date 202504130120
```

No modo de operação, formato: YYYYMMDDHHMM

Usando NTP
```
{master:0}[edit]
root@vqfx-re# set system ntp server X.X.X.X

{master:0}
root@vqfx-re> show ntp associations 
```

DNS
```
{master:0}[edit]
root@vqfx-re# set system name-server XX.X.X.X
```
Usuários
```
{master:0}[edit]
root@vqfx-re# set system login user TESTE class super-user authentication plain-text-password
```
Junos usa como criptografia o SHA512 (começa com $6$)

## Permissões

CATEGORIAS – FLAGS

ALL – permissão total
clear – usar o clear
configure – entrar em modo config e commitar
network – pingar, tracetear, ssh e telnet
view – visualizar

Classes

SUPER-USER usa a categoria ALL

UNAUTHORIZED – Não faz nada no equipamento, pode ser usado pra novos empregados mas que ainda não vai logar, limitar autenticação

OPERATOR – clear, view, network, reset, trace

READ-ONLY

Criar classes customizadas
```
[edit]
raimon# set system login class TESTE permissions [ clear network view] 
[edit]
raimon# set system login class TESTE allow-commands "(configure private)" 
[edit]
raimon# set system login class TESTE deny-commands "(file)"  
[edit]
raimon# set system login class TESTE allow-configurations "(interfaces) | (firewall)" 
[edit]
raimon# set system login class TESTE deny-configuration "(groups)"  
[edit]
raimon# set system login class TESTE idle-timeout 5   
```
Ordem de processamento

![](assets/img/posts/post-25/23.png)

Acesso WEB (J-Web)

Mais usado nos SRX SERIES
```
set system services web-management http ou https system-generated-certificate
```
![](assets/img/posts/post-25/24.png)

RADIUS | TACACS+
```
[edit]
raimon# set system radius-server 10.1.10.1 secret test 

[edit]
raimon# set system tacplus-server 10.1.10.2 secret test  

[edit]
raimon# set system authentication-order [ radius tacplus password ]
```

password = senha local na caixa

Mesmo sem configurar o password, se os servidores estiverem off, o Junos verifica os usuarios locais

## Provisionando um novo dispositivo Junos

Cada plataforma vem com configs default diferentes. SWS vem na preconfig com RSTP. Alguns roteadores/firewall vem sem IP, mas o SRX vem com um IP configurado na LAN.

Porta console

- 8p8c port
- RS-232 protocolo
- 9600 (speed)

O Amnesiac significa que o dispositivo ainda não tem um hostname

Configuração mandatória: Configurar uma senha para o usuário root
```
{master:0}[edit]
root@vqfx-re# set system root-authentication plain-text-password 
```
## Recomendações de primeira configurações para um dispositivo novo:

- Configurar pelo menos um usuário
- Definir um hostname
- Habilitar SSH
- Definir Data e horário
- Configura uma interface para gerência

EX: Habilitando SSH pro root no QFX
```
{master:0}[edit]
root@vqfx-re# set system services ssh root-login allow    
```
Banner

\n = Quebra de linha
```
{master:0}[edit]
root@vqfx-re# set system login message "MENSAGEM ANTES DE LOGAR" 

{master:0}[edit]
root@vqfx-re# set system login announcement "ISSO AI\nLOGOU COM SUCESSO" 
```
Configuração de fábrica
```
{master:0}[edit]
root@vqfx-re# load factory-default 

{master:0}[edit]
root@vqfx-re# show | compare    
+   commit {
+       factory-settings {
```
É possivel deletar seções
```
{master:0}[edit]
root@vqfx-re# delete system commit factory-settings 
```
Configuração de Resgate

É possivel criar uma configuração de resgate (snapshot) para poder dar rollback sempre que puder, já que o Junos salva apenas as ultimas 49 alterações
```
{master:0}
root@vqfx-re> request system configuration rescue save 

{master:0}[edit]
root@vqfx-re# rollback rescue
```
Reiniciar e Desligar o dispositivo

Reiniciar
```
{master:0}
root@vqfx-re> request system reboot
```
Desligar o Junos (e depois pessoalmente desligar o hardware)
```
{master:0}
root@vqfx-re> request system halt
```
Desligar o Junos remotamente
```
{master:0}
root@vqfx-re> request system power-off
```
## Provisionamento Zero-Touch

É possivel enviar configurações para configurar de forma autonôma um dispositivo Junos. Usando para isso ZTP Server, Junos Image Server e Config Server.

O ZTP utiliza DHCP para fazer requisição de configuração ou atualização de software, se aproveitando o DHCP Options para fornecer as informações necessárias.

## Recuperando a senha do root

É possivel resetar a senha do root, porém envolve acessar via porta Console, rebootar a caixa e apertar combinações de teclas para interromper o boot

Mais em: <https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/recovering-root-password.html>

## Gestão de Usuários

Visualizar usuários logados

```
raimon> show system users  
```
Derrubar um usuário logado
```
raimon> request system logout user teste
```
## Interfaces de rede

Nomes das interfaces físicas

- fe (fast)
- ge (gigabit)
- xe (ten gigabit)
- et (40 gigabit +)
- ae (LAG)
- gr (GRE)

Exemplo:

```xe-4/2/0```, 4 é o line card, 2 pode ser a interface dentro do line card ou uma sessão específica do line card, 0 é o numero da porta em si.

A nomenclatura das interfaces é consistente em todos os equipamentos da Juniper, facilitando scripts etc.

Comando úteis:

![](assets/img/posts/post-25/25.png)

extensive

![](assets/img/posts/post-25/26.png)

Outro:
```
show interface descriptions
```
Para configurar questões de Speed, Duplex, L2 MTU é feito direto na interface. Exemplo: xe-4/2/0

_Algumas saidas do show foram omitidas_
```
root@ROUTER# set interfaces ge-0/0/0 speed 1g 
root@ROUTER# set interfaces ge-0/0/0 link-mode full-duplex
root@ROUTER# set interfaces ge-0/0/0 mtu 9000

root@ROUTER> show interfaces ge-0/0/0              
Physical interface: ge-0/0/0, Enabled, Physical link is Up
  Description: INTERNET
  Link-level type: Ethernet, MTU: 1514, MRU: 1522, LAN-PHY mode,
  Speed: 1000mbps, BPDU Error: None, Loop Detect PDU Error: None,
  Current address: 50:1f:d5:00:8f:02, Hardware address: 50:1f:d5:00:8f:02
  Last flapped   : 2025-03-29 15:12:11 BRT (1w0d 18:16 ago)
```
Para configurações de protocolos, vlans, L3 MTU, é feito apenas nas interfaces lógicas. Exemplo: ```xe-4/2/0.0 (unit 0)```

_Algumas saidas do show foram omitidas_
```
set interfaces ge-0/0/0.0 family inet address 10.1.10.1/30
set interfaces ge-0/0/0.0 family inet6 address fd10:1:10::1/126
set interfaces ge-0/0/0.0 family inet mtu 1500
set protocols ospf3 area 0.0.0.0 interface ge-0/0/0.0 interface-type p2p

root@ROUTER> show interfaces ge-0/0/2.0    
  Logical interface ge-0/0/0.0 (Index 332) (SNMP ifIndex 547)
    Protocol inet, MTU: 1500
      Addresses, Flags: Is-Preferred Is-Primary
        Destination: 10.0.137/24, Local: 10.0.137.248, Broadcast: 10.0.137.255
```
Toda interface sempre tem pelo menos 1 logical unit

Interface loopback (está sempre up e pode ser alcançada pela rede, quando divulgada no OSPF)
```
[edit]
root@ROUTER-01# set interfaces lo0.0 family inet address 10.1.1.1/32 
set interfaces lo0.0 family inet6.0 address fd10:1:1::1/32
```
Interfaces VLANS (multiplos logical units) – Router on a Stick

Atribui a propriedade de vlan tag direto na interface, depois cria os logical units e associando a vlan-id
```
set interfaces ge-0/0/3 vlan-tagging
set interfaces ge-0/0/3 unit 10 vlan-id 10
set interfaces ge-0/0/3 unit 20 vlan-id 20
```
Interface IRB (Integrated routing and bridging) – Mais utilizado Switch L3
```
{master:0}[edit]
root@SWITCH-01# set interfaces irb.10 family inet address 10.1.10.1/24 
{master:0}[edit]
root@SWITCH-01# set vlans Server l3-interface irb.10   

{master:0}
root@SWITCH-01> show interfaces terse irb

{master:0}
root@SWITCH-01> show vlans Server detail  
```
## LAG

Feito em 3 passos
```
[edit]
root@ROUTER-01# set chassis aggregated-devices ethernet device-count 1

[edit]
root@ROUTER-01# set interfaces ae0.0 family inet address 10.1.1.1/24 
[edit]
root@ROUTER-01# set interfaces ae0 aggregated-ether-options lacp active   

[edit]
root@ROUTER-01# set interfaces ge-0/0/4 gigether-options 802.3ad ae0 
[edit]
root@ROUTER-01# set interfaces ge-0/0/5 gigether-options 802.3ad ae0 

-----

root@ROUTER-01> show interfaces terse | match ae0 
ge-0/0/4.0              up    up aenet    --> ae0.0
ge-0/0/5.0              up    up aenet    --> ae0.0
ae0                     up    up
ae0.0                   up    up inet     10.1.1.1/24 
```

device-count 1 = ae0 , device-count 3 = ae0,ae1,ae2

LLDP

Podemos configurar LLDP nas interfaces para melhor identificação da topologia
```
{master:0}[edit]                        
root@vqfx-re# set protocols lldp interface all 
```

## Preferred e Primary IP Address

Uma interface pode ter varios IP, qual será usado na saida de tráfego, dependerá da situação (destino)

Preferred = para definir o ip como origem do tráfego na mesma subnet

Primary = para definir o ip como origem do tráfego fora da subnet (por padrão o menor ip é selecionado)

![](assets/img/posts/post-25/27.png)


Mas é possivel definir manualmente

![](assets/img/posts/post-25/28.png)
![](assets/img/posts/post-25/29.png)

## Roteamento

Preferência / Distância administrativa (para desempatar prefixos iguais de protocolos diferentes)

O menor valor é o mais preferivel

Diretamente conectado	| 0
Estático	| 5
OSPF	|10
IS-IS	| 15
BGP	| 170

RIB = Routing information base (todas as opções de rotas)

FIB = Forwarding information base (as melhores rotas serão usadas aqui para encaminhamento)

![](assets/img/posts/post-25/30.png)

Verificar tabela de roteamento
```
root@ROUTER-01> show route
...
root@ROUTER-01> show route table inet (apenas ipv4)

inet.0: 13 destinations, 13 routes (13 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

0.0.0.0/0          *[Access-internal/12] 5d 03:08:07, metric 0
                    >  to 10.0.137.254 via ge-0/0/0.0
                                        
root@ROUTER-01> show route table inet6.0 (apenas ipv6)

inet6.0: 16 destinations, 16 routes (16 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2001:db8:0:10::/64 *[Direct/0] 5d 03:08:06
                    >  via ge-0/0/2.10
```
Rotas estáticas

IPv4 (rib inet.0 não obrigatório)
```
[edit]
root@ROUTER-01# set routing-options rib inet.0 static route 200.200.200.0/22 next-hop 10.5.5.5     
```
IPv6 (rib inet6.0 obrigatório)
```
[edit]
root@ROUTER-01# set routing-options rib inet6.0 static route 2001:db8:0:40::/64 next-hop 2001:db8:0:100::2
```
Rota estática de backup (hop secundário e aumentando a preferencia)
```
[edit]
root@ROUTER-01# set routing-options static route 0.0.0.0/0 qualified-next-hop 10.2.2.2 preference 7
```
Next-hop remoto (recursivo – Precisa ter OSPF/IS-IS na rede)

![](assets/img/posts/post-25/31.png)
![](assets/img/posts/post-25/32.png)

Podemos ver o ```metric2``` que foi obtivo do OSPF (informativo apenas, não será usado para decisão de roteamento)

### Roteamento Dinâmico

OSPFv2/v3 (mesma sintaxe)
```
[edit]
root@ROUTER-01# set routing-options router-id 10.1.1.1
set protocols ospf area 0.0.0.0 interface ge-0/0/3.0 interface-type p2p

set protocols ospf3 area 0.0.0.0 interface ge-0/0/2.10 passive
```
### Routing Instances

Basicamente é a criação de tabelas de roteamento de forma virtual em que não se falam (podem ter os mesmos prefixos, sem conflitos)

![](assets/img/posts/post-25/33.png)

```
[edit]
root@ROUTER-01# set routing-instances INSTANCIA-CLIENTE-A instance-type virtual-router 

[edit]
root@ROUTER-01# set routing-instances INSTANCIA-CLIENTE-A interface ge-0/0/6.0 

[edit]
root@ROUTER-01# set routing-instances INSTANCIA-CLIENTE-A routing-options static route 0/0 next-hop 10.50.50.1 

[edit]
root@ROUTER-01# run show route table INSTANCIA-CLIENTE-A.inet.0    

[edit]
root@ROUTER-01# run show interfaces terse routing-instance INSTANCIA-CLIENTE-A 

[edit]
root@ROUTER-01# run ping 10.1.1.1 routing-instance INSTANCIA-CLIENTE-A  
```

## Switch

A Juniper possui 2 linhas de switchs. QFX e EX

![](assets/img/posts/post-25/34.png)


Switch EX:

- Versátil, tanto para pequenos escritórios quanto datacenters
- Recursos como: Port Security, Protocolos de prevenção de loops, PoE, Virtual Chassis, MACsec, EVPN-VXLAN

Abaixo exemplo do EX4300

![](assets/img/posts/post-25/35.png)
![](assets/img/posts/post-25/36.png)

PIC 0 – exclusivo para portas de rede, PIC 2 – Pode ser usado para portas de rede ou usada em outra aplicação (Virtual Chassis)

PIC 1 – Portas de 40GB (fica atrás do equipamento)

Switch QFX:

- Design voltado para datacenters e grandes empresas
- Otimizado para protocolos L3, grande tabela MAC e de roteamento

![](assets/img/posts/post-25/37.png)

Acessando sem IP

Porta MGMT

Verificando VLANS
```
{master:0}
root@vqfx-re> show vlans 

Routing instance        VLAN name             Tag          Interfaces
default-switch          Servers               10           xe-0/2/2.0*
```
O (*) significa que a interface está UP

Criando VLANS

Os nomes são Case-sensitive
```
{master:0}[edit]
root@vqfx-re# set vlans v20 vlan-id 20   
{master:0}[edit]
root@vqfx-re# commit 

{master:0}[edit]
root@vqfx-re# show vlans 
v20 {
    vlan-id 20;
}
```
O show mostra em ordem alfabetica e não por ID

Access Ports
```
set interfaces xe-0/0/2 unit 0 family ethernet-switching interface-mode access
set interfaces xe-0/0/2 unit 0 family ethernet-switching vlan members Servers

{master:0}[edit]
root@vqfx-re# show interfaces xe-0/0/2    
unit 0 {
    family ethernet-switching {
        interface-mode access;
        vlan {
            members Servers;
        }
    }
}
```
A configuração da VLAN na interface se da pelo nome e não pelo ID (Servers nesse caso é a vlan de ID 10)

Também notamos que é feito na unit 0 e não na interface em si

Verificar tabela MAC
```
{master:0}
root@vqfx-re> show ethernet-switching table 
```
![](assets/img/posts/post-25/38.png)

Trunk Ports

Ao criar uma interface trunk, as vlans ficam bloqueadas por padrão essa interface. Para permitir é necessário adicionar explicitamente
```
set interfaces xe-0/0/1 unit 0 family ethernet-switching interface-mode trunk
set interfaces xe-0/0/1 unit 0 family ethernet-switching vlan members Servers
set interfaces xe-0/0/1 unit 0 family ethernet-switching vlan members Test
```
Outras formas

![](assets/img/posts/post-25/39.png)

## Logs, Troubleshoot e Monitoramento

O Junos possui 2 arquivos de syslogs por padrão, porém podemos criar quantos quisermos.

messages (eventos gerais do sistema) e interactive-commands (mostra os comandos dos usuarios no CLI)
```
raimon> show configuration system
syslog {
    file messages {
        any notice;
        authorization info;
    }
    file interactive-commands {
        interactive-commands any;
    }
}
```
Podemos ter logs com base:

- Facility: kernel, usuario, autorização etc
- Nivel de severidades: 7 niveis + any
- Destino dos logs: local, servidor externo ou CLI

Alguns logs é melhor entendido usando sistemas de monitoramento (SNMP)

Quando o arquivo de log fica cheio, o Junos compacta em .gz

Algumas plataformas tem o limite de 128KB de logs e outros de 5MB

Logs Facility (abaixo não tem todas)

![](assets/img/posts/post-25/40.png)

any = todos

Nive de severidade de logs

![](assets/img/posts/post-25/41.png)

Criando um arquivo de logs

![](assets/img/posts/post-25/42.png)

Fica salvo em /var/log

world-readable = todos os usuarios podem ler

Visualizar logs em tempo real
```
raimon> monitor start messages [ou arquivo]
```
monitor stop
Decifrando mensagens de logs

Alguns logs possuem codigos e é possivel usar o comando abaixo para entender melhor do que se trata
```
raimon> help syslog RPD_BGP_NEIGHBOR_GSHUT_SENDER_COMPLETED 
Name:          RPD_BGP_NEIGHBOR_GSHUT_SENDER_COMPLETED
Message:       graceful-shutdown sender functionality for peer: <peer-name> is
               enabled, local-preference: <bgp-local-pref>, completed
               advertising routes to the BGP peer with gshut community
Help:          Finished marking all routes to the BGP Peer with the
               graceful-shutdown community
Description:   Finished queuing all routes to be advertised to the BGP peer
               with the graceful-shutdown community.
Type:          Event: This message reports an event, not an error
Severity:      warning
Facility:      LOG_DAEMON
Action:        None
```
Enviar logs para servidor externo

![](assets/img/posts/post-25/43.png)

Enviar logs direto para o CLI

![](assets/img/posts/post-25/44.png)

Logs mais detalhados

Para poder ter mais detalhes em logs, podemos habilitar o traceoptions nas configurações de alguma hierarquia. Exemplo: OSPF
```
[edit]
raimon# set protocols ospf traceoptions file OSPF_TRACE.txt size 1M  
[edit]
raimon# set protocols ospf traceoptions flag event detail                   
[edit]
raimon# set protocols ospf traceoptions flag error detail   
```

![](assets/img/posts/post-25/45.png)

Podemos ver que o erro de MTU não mostra em messages

Pings (com opções)

```rapid``` (enviar pings de forma imediata assim que tiver resposta da anterior)

```rapid count 100``` (enviar 100 probes rapidamente)

```size 1472 do-not-fragment``` (otimo para testar MTU)

Traceroute

```monitor``` (mtr)

ARP / NDP
```
raimon> show arp
```
```
raimon> show ipv6 neighbors
```

Visualizar tráfego em tempo real
```
raimon> monitor interface traffic 
```
Interface específica
```
raimon> monitor interface ge-0/0/1
```
Visualizar pacotes (exceptions) – TCPDUMP
```
raimon> monitor traffic interface ge-0/0/0 [detail | extensive] no-resolve
```
## Documentações de ajuda no Junos

Lembrar comandos (muda se você estiver em modo config ou operacional)
```
[edit]
raimon# help apropos host-name 
set system host-name <host-name> 
    Hostname for this router
```
Aprender algum tópico
```
raimon> help topic ospf bfd-liveness-detection 
                       Example: Configuring BFD for OSPF

   This example shows how to configure the Bidirectional Forwarding Detection
   (BFD) protocol for OSPF.
     * Requirements
     * Overview
     * Configuration
     * Verification
```     
Lembrar sintaxe
```
raimon> help reference ospf area    
area
  Syntax
     area area-id {
```
## SNMP

Configuração mínima
```
[edit]
raimon# set snmp community public authorization read-only 

[edit]
raimon# set snmp community public clients 172.16.0.0/24   
```
Traps
```
[edit]
raimon# set snmp trap-group TRAPS version v2  

[edit]
raimon# set snmp trap-group TRAPS categories chassis 

[edit]
raimon# set snmp trap-group TRAPS targets 172.16.0.1   
```
Visualizar mudanças no CLI repedidamente (2 segundos)
```
raimon> show ospf neighbor | refresh 2   
```

## Firewall

Stateful = Security polices (mantém os estados das conexões)

Stateless = Firewall filters (não mantem estados e é usado para bloquear violações de seguranças mais obvias)

### Firewall Filters

Não se limita apenas a bloquear e permitir tráfego


![](assets/img/posts/post-25/46.png)

Declarações de from/then. Chamadas de Term

TERM 1 FROM {condição] THEN {ação)

Condições


![](assets/img/posts/post-25/47.png)

Podemos ter varios elementos de condições no mesmo termo.

- Condições igual irá agir como um OR
- Condições diferentes irá agir como um AND

Ação


![](assets/img/posts/post-25/48.png)

Ações sem terminação

São implicitamente ACCEPT


![](assets/img/posts/post-25/49.png)

Exemplos:


![](assets/img/posts/post-25/50.png)

Exemplo
```
set firewall family inet filter FW term BLOCK_GAMING from source-address 172.16.10.22/32
set firewall family inet filter FW term BLOCK_GAMING from source-address 172.16.10.44/32
set firewall family inet filter FW term BLOCK_GAMING from destination-address 198.51.100.48/32
set firewall family inet filter FW term BLOCK_GAMING from protocol tcp
set firewall family inet filter FW term BLOCK_GAMING from destination-port 666
set firewall family inet filter FW term BLOCK_GAMING then log
set firewall family inet filter FW term BLOCK_GAMING then discard

set firewall family inet filter FW term GUEST_COUNT_WEB from source-address 172.16.30.0/24
set firewall family inet filter FW term GUEST_COUNT_WEB from protocol tcp
set firewall family inet filter FW term GUEST_COUNT_WEB from destination-port http
set firewall family inet filter FW term GUEST_COUNT_WEB from destination-port https
set firewall family inet filter FW term GUEST_COUNT_WEB then count PACKETS_WEB

set firewall family inet filter FW term BLOCK_GUEST_PING from source-address 172.16.30.0/24
set firewall family inet filter FW term BLOCK_GUEST_PING from protocol icmp
set firewall family inet filter FW term BLOCK_GUEST_PING from icmp-type echo-request
set firewall family inet filter FW term BLOCK_GUEST_PING from icmp-type echo-reply
set firewall family inet filter FW term BLOCK_GUEST_PING then discard

set firewall family inet filter FW term ALLOW then accept
```
Para ativar de fato, deve-se adicionar interface
```
[edit]
raimon# set interfaces ge-0/0/0.10 family inet filter output FW
```
Dependendo do sentido do tráfego ou localização do device na topologia, pode ser output ou input (no caso acima, dropo na interface de saida pra internet)

Adicionando varios filtros

![](assets/img/posts/post-25/51.png)


Caso nenhum termo seja atendido, por padrão o Junos tem a politica implicita de DROPAR tudo, por isso deve-se criar um term com accept no final.

Sem especificar se é porta de origem ou destino, o Junos checa ambos

Sem especificar se é TCP ou UDP, o Junos checa ambos

Verificar counters
```
raimon> show firewall counter PACKETS_WEB filter FW
```

### Adicionando novo termo em um Firewall Filter existente

Ao adicionar com o set, esse termo vai para o final, após o accept explicito. Para adicionar acima, é recomendado usar o Insert
```
[edit]
raimon# insert firewall family inet filter FW term TESTE before term ALLOW  
```
ou
```
[edit]
raimon# insert firewall family inet filter FW term TESTE after term BLOCK_GAMING  
```
Usando prefix-list
```
set policy-options prefix-list REDE-LAN-RAIMON 10.1.10.0/24
set policy-options prefix-list REDE-LAN-RAIMON 10.1.20.0/24
```
Prefix-list automática (usando rotas diretamente conectadas por exemplo)

![](assets/img/posts/post-25/52.png)
![](assets/img/posts/post-25/53.png)


Para proteger o RE, é aconselhavel criar regras para a loopback (input)

Policer (rate-limit traffic)
```
set firewall policer 8MB if-exceeding bandwidth-limit 8m
set firewall policer 8MB if-exceeding burst-size-limit 1500
set firewall policer 8MB then discard
```
![](assets/img/posts/post-25/54.png)

Combinando

![](assets/img/posts/post-25/55.png)

Boas práticas:

Definir explicitamente o que é definido implicitamente

![](assets/img/posts/post-25/56.png)


Considerar as ordens dos termos, adicionando no topo as condições que mais satisfazem o filtro. Também os protocolos mais importantes

Sempre faça filtros usando o address family

![](assets/img/posts/post-25/57.png)


Jeito legado de fazer (direto na hierarquia firewall)

![](assets/img/posts/post-25/58.png)

## Policy Options

Usado para mudar o comportamento padrão dos protocolos de roteamento em relação a instalação de prefixos na tabela de roteamento

Tem a mesma sintaxe de Firewall Filters

Import/Export

Do ponto de vista da tabela de roteamento, a instalação de prefixos nela (vindo de um protocolo) é o Import, e o oposto é o Export

Cada protocolo tem seu comportamento padrão em relação a Export.

OSPF = Não exporta nada da tabela de rotamento (os LSA que tomam conta disso para manter toda a topologia identica para os roteadores). Ou seja, por padrão o OSPF não anuncia prefixos da tabela de roteamento que foram aprendidos por outros protocolos. Para Import a ação implicita é aceitar os prefixos anunciados por OSPF (LSA).

Os LSA tem que ser igual em todos os roteadores, por isso não é possivel filtrar prefixos internos do OSPF com import/export polices

BGP = Exporta os prefixos que tem na tabela de roteamento aprendidos por BGP. Para import a ação implicita é aceitar os prefixos anunciados por BGP. (porém tem questões de IBGP para evitar loops que não é o foco)

Exemplo OSPF:

![](assets/img/posts/post-25/59.png)
![](assets/img/posts/post-25/60.png)

Várias policies (Policy Chaining)

![](assets/img/posts/post-25/61.png)

Route-filters

Usado para filtrar prefixos de forma mais específicas

![](assets/img/posts/post-25/62.png)

exact

![](assets/img/posts/post-25/63.png)

orlonger

![](assets/img/posts/post-25/64.png)

longer

![](assets/img/posts/post-25/65.png)

upto

![](assets/img/posts/post-25/66.png)

prefix-length-range

![](assets/img/posts/post-25/67.png)


## Atualizando o Junos OS

Em resumo é feito em 4 passos:

![](assets/img/posts/post-25/68.png)


No site ao pesquisar a imagem do Junos, tem duas opções e cada uma é um metodo diferente

![](assets/img/posts/post-25/69.png)

A primeira é para atualizar o Junos salvando a imagem no proprio dispositivo e a outra opção é usando um pen drive.

Existe a imagem Limited, mas apenas alguns paises precisam desse.

Imagens NET é usado quando não tem acesso ao CLI da caixa (situação de emergência) usando o loader prompt

Nomenclatura do arquivo

![](assets/img/posts/post-25/70.png)


Verifica-se primeiro se tem espaço em disco
```
raimon> show system storage
```
O arquivo é baixo em ```/var/tmp```

![](assets/img/posts/post-25/71.png)


Caso necessário, para liberar espaço
```
raimon> request system storage cleanup dry-run
```
Baixando a imagem pro device
```
raimon> file copy "LINK DE DOWNLOAD DA IMAGEM" /var/tmp
```
Verificar versão

raimon> show version

Atualizar
```
raimon> request system software add /var/tmp/IMAGEM
```
Algumas caixas tem arquitetura diferente, com isso o comando muda

EX: MX204
```
raimon> request vmhost software add /var/tmp/IMAGEM
```
Rebootar
```
raimon> request system reboot
```
Mais opções (se necessário)

![](assets/img/posts/post-25/72.png)

Reverter

O Junos mantem uma cópia da versão antiga quando atualiza de forma bem sucedida
```
raimon> request system software rollback
```
## Extra

### Server DHCP – Switch
```
set system services dhcp-local-server group server-rede-10 interface irb.10
set access address-assignment pool REDE-10-POOL family inet network 172.16.10.0/24
set access address-assignment pool REDE-10-POOL family inet dhcp-attributes domain-name 1.1.1.1
set access address-assignment pool REDE-10-POOL family inet dhcp-attributes router 172.16.10.254
set access address-assignment pool REDE-10-POOL family inet excluded-address 172.16.10.254
```

### RPM (Real-Time Performance Monitoring)

Realiza pings periodicamente 

Exemplo:
```
set services rpm probe IP-KEEPALIVE test VLAN3001-2 target address 192.168.47.2
set services rpm probe IP-KEEPALIVE test VLAN3001-2 probe-type icmp-ping
set services rpm probe IP-KEEPALIVE test VLAN3001-2 source-address 192.168.47.1
set services rpm probe IP-KEEPALIVE test VLAN3001-2 destination-interface ae9.3001
set services rpm probe IP-KEEPALIVE test VLAN3001-2 probe-count 3
set services rpm probe IP-KEEPALIVE test VLAN3001-2 probe-interval 5
set services rpm probe IP-KEEPALIVE test VLAN3001-2 test-interval 5
```