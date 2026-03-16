---
title: "22 | Sobre DNS – Parte 01"
date: 2025-02-24 00:00:00 -0300
categories: [Artigos]
tags: [DNS, Networking, Internet]
image:
  path: assets/img/posts/post-22/capa.png
---

Você já se perguntou como um simples clique no seu navegador leva você ao site desejado? Um dos segredos está no DNS (Domain Name System), que traduz nomes de domínio em endereços IPs. Mesmo que você não tenha digitado nenhum site explicitamente na URL do navegador, várias consultas DNS são realizadas por baixo dos panos.

O DNS é um sistema responsável por traduzir nomes amigáveis (exemplo: jeffersonraimon.com), conhecido como Domínios, em endereços IP numéricos (192.168.1.1), necessários para localizar recursos na internet. É como se fosse a sua agenda em que é relacionado o nome do contato com o numero de telefone da pessoa.

O DNS talvez seja um dos componentes mais importantes da Internet junto ao BGP, pois sem o DNS, precisaríamos memorizar os IPs de todos os sites que acessamos, o que seria um processo complexo e impraticável. Outro problema seria que, caso fosse necessário alterar o IP de um servidor, seria difícil descobrir o novo endereço. Mas com o DNS, esse processo se torna muito mais fácil.

Por exemplo, eu mantenho um homelab com diversas VMs rodando aplicações Docker, como o Jellyfin. Para facilitar o acesso a essas VMs sem a necessidade de ficar digitando o IP de cada uma, utilizo o AdGuard Home como meu servidor DNS doméstico. Com ele, posso associar domínios aos IPs que eu desejo. Dessa forma, posso configurar a VM do Jellyfin, com o IP 10.1.10.13, para ser acessada através do nome jellyfin.mylab.lan, por exemplo. Sendo esse um FQDN (Fully Qualified Domain Name) pois é um dominio completo, já que inclui o nome da VM (jellyfin).

Talvez você já tenha estudado sobre o processo de consulta DNS, em que é citado o Servidor DNS Recursivo, Root Servers, TLD e Servidor Autoritativo e como eles se relacionam, por exemplo nessa imagem:

 ![](assets/img/posts/post-22/capa.png)

Até agora, nunca vi uma explicação prática de como esse processo ocorre, especialmente no nível da captura de pacotes, onde podemos observar os IPs dos servidores e suas respectivas respostas. Neste post, vou compartilhar essas capturas e como consegui obtê-las. Isso porque, ao usar um servidor DNS qualquer e realizar uma captura de pacotes simples na sua maquina, você apenas terá acesso ao resultado final (o IP do servidor), sem visualizar a sequência com os outros Servidores. Por exemplo:

 ![](assets/img/posts/post-22/01.png)

Podemos observar que, ao requisitar o IP do site amazon.com.br (captura 28), a resposta já contém o IP (captura 30). Então, onde está todo aquele processo? Bem, vamos deixar para mais adiante. Antes, que tal revisarmos rapidamente o cabeçalho do DNS?

O cabeçalho do DNS tem alterações de acordo com o tipo de pacote. Vejamos alguns exemplos:

Cabeçalho de Consulta simples

 ![](assets/img/posts/post-22/02.png)

Transaction ID

Tamanho: 16 bits

Descrição: É um identificador único que acompanha uma consulta ou resposta. O cliente gera um ID aleatório para cada consulta, e esse ID será retornado pelo servidor na resposta, para que o cliente possa associar a resposta à consulta original.

Flags

Tamanho: 16 bits
Descrição: As flags indicam o tipo de consulta ou resposta. As flags mais comuns são:

- QR (Query/Response): Indica se a mensagem é uma consulta (0) ou uma resposta (1).
- Opcode: Define o tipo da consulta. O valor mais comum (0) é para uma consulta de resolução padrão (QUERY).
- TC (Truncated): Indica se a resposta foi truncada.
- RD (Recursion Desired): Indica se o cliente deseja que o servidor faça a resolução recursiva.
- Z: Reservado para uso futuro, atualmente sempre é 0.
- AD (Authenticated Data): Indica que a resposta foi autenticada.

Contadores

Tamanho: 16 bits cada (4 campos no total)

- QDCOUNT (Question Count): Indica o número de consultas na mensagem.
- ANCOUNT (Answer Count): Indica o número de respostas na mensagem.
- NSCOUNT (Authority Count): Indica o número de registros de autoridade (servidores autoritativos) na resposta.
- ARCOUNT (Additional Count): Indica o número de registros adicionais (geralmente, informações de servidores de recursos adicionais, como o IP de um servidor autoritativo).

Consulta (Question Section)

Descrição: Contém a pergunta feita ao servidor DNS, que inclui:

- Nome: O nome do domínio que está sendo consultado (por exemplo, “amazon.com.br”).
- Tipo: O tipo de registro solicitado (A, AAAA, CNAME, MX, etc.).
- Classe: A classe do registro (geralmente, o valor é “IN” para Internet).

Adicionais (Additional Section)

Descrição: Contém registros adicionais que podem ser úteis para a consulta. Por exemplo, o IP de um servidor de nome autoritativo, que pode ser necessário para resolver a consulta.

Captura de uma Consulta Simples demostrando as informações acima.

 ![](assets/img/posts/post-22/03.png)

Cabeçalho de Resposta

 ![](assets/img/posts/post-22/04.png)

Temos como diferença:

Flags

Descrição: As flags indicam o tipo de consulta ou resposta. As flags mais comuns são:

- Authoritative Answer – 1 bit: Indica se a resposta vem de um servidor autoritativo.
- RA (Recursion Available) – 1 bit: Indica se o servidor oferece suporte à pesquisa recursiva.
- Answer Authenticated – 1 bit: indica que a resposta da consulta DNS foi autenticada com sucesso através de assinaturas criptográficas válidas. Usado em DNSSEC.
- RCODE (Reply Code) – 4 bits: Código de resposta, que indica o status da consulta (por exemplo, “sem erro”, “formato de nome inválido”, “servidor não encontrado”, etc.).

 ![](assets/img/posts/post-22/05.png)

Resposta com informação de Servidor de Nomes Autoritativo

 ![](assets/img/posts/post-22/06.png)

No cabeçalho DNS, o campo “authoritative nameservers” (servidores de nomes autoritativos) indica quais servidores DNS são responsáveis por fornecer a resposta oficial e definitiva para uma consulta DNS, ou seja, aqueles que têm autoridade sobre um domínio específico.

Também é informado que o tipo do NS é SOA. O SOA (Start of Authority) é um tipo de registro no sistema de nomes de domínio (DNS) que contém informações cruciais sobre a zona de um domínio, como a autoridade sobre a zona e a configuração de atualizações. Ele é um dos registros mais importantes dentro de um arquivo de zona DNS.

 ![](assets/img/posts/post-22/07.png)

Alguns Registros DNS

- A (Address): Mapeia um nome de domínio para um endereço IPv4.
- AAAA (IPv6 Address): Mapeia um nome de domínio para um endereço IPv6.
- MX (Mail Exchange): Define servidores de e-mail para o domínio.
- CNAME (Canonical Name): Alias para outro nome de domínio.
- NS (Name Server): Aponta para o servidor autoritativo responsável pelo domínio.

Tipos de Servidores DNS

- Servidor Recursivo: Este servidor recebe as consultas de nome de domínio e busca as respostas, se necessário, realizando várias consultas a outros servidores. Ele armazena as respostas em cache para acelerar futuras requisições.

- Servidor de Encaminhamento: Quando o servidor recursivo não pode resolver uma consulta localmente (ou seja, não tem o IP no cache), ele pode encaminhar a consulta para um servidor DNS externo (geralmente um servidor mais próximo ou especializado). Em redes domésticas, é o próprio roteador, pois ele adiciona seu IP de gateway nas informações de DNS.

- Root Servers: São os servidores principais que conhecem os servidores TLD (como .com, .org, .br). Eles são os primeiros a serem consultados quando não há um cache ou encaminhamento configurado.

- Servidores TLDs (Top-Level Domains): Os TLDs são as extensões finais dos nomes de domínio, como .com, .org, .net, .br, etc. Cada TLD tem seus próprios servidores de nomes, chamados de servidores de nomes TLD. Esses servidores são responsáveis por saber onde encontrar os servidores DNS autoritativos para os domínios específicos de cada TLD.

- Servidores Autoritativos: Esses servidores têm a autoridade final sobre um domínio específico e fornecem as respostas definitivas para a consulta de nomes de domínio, geralmente devolvendo o IP correspondente.

Agora vamos entender o processo passo a passo da resolução DNS com captura de pacotes.

Para isso, criei um Servidor Recursivo usando o Unbound e adicionei os IPS dos Root Servers para o Unbound utilizar.

 ![](assets/img/posts/post-22/08.png)

O Fluxo Completo:

O usuário digita um nome de domínio, como “uol.com.br”.

O computador verifica primeiro se já possui a tradução do domínio para o IP armazenada em seu cache (Ou no arquivo etc/hosts.). Isso acelera o processo, evitando que a consulta seja feita do zero. Caso não encontre, a consulta é encaminhada para o próximo nível.

 ![](assets/img/posts/post-22/09.png)

O servidor recursivo não sabe o IP, então ele começa a consulta. Ele faz uma solicitação para os servidores root para saber onde encontrar os servidores TLD do domínio “.br”.

 ![](assets/img/posts/post-22/10.png)

O servidor root responde com os servidores TLD responsáveis por “.br”. Podemos ver que o endereço IP vem no campo Additional Records.

 ![](assets/img/posts/post-22/11.png)

O servidor recursivo então consulta o servidor TLD para “com.br”.

 ![](assets/img/posts/post-22/12.png)

O Servidor TLD responde informando quem é o servidor responsável. (Nesse caso, é o mesmo servidor do “.br”. O servidor utilizado é o ‘a.dns.br’, e o IP correspondente já foi fornecido anteriormente – Passo 4.) Também podemos ver que a flag Autoritative está com 1 bit e informa que é um servidor autoritário do domínio.

 ![](assets/img/posts/post-22/13.png)

O servidor recursivo então consulta o servidor TLD dessa vez perguntando por “uol.com.br”.

 ![](assets/img/posts/post-22/14.png)

O servidor TLD responde com as informações dos servidores autoritativos de uol.com.br

 ![](assets/img/posts/post-22/15.png)

O servidor recursivo então consulta o servidor autoritativo do domínio “uol.com.br”. Nesse caso foi o servidor “borges.uol.com.br”

 ![](assets/img/posts/post-22/16.png)

O servidor autoritativo responde com o endereço IP associado ao “uol.com.br” – 200.147.3.157.

 ![](assets/img/posts/post-22/17.png)

E finalmente o servidor recursivo recebe o IP e o retorna ao usuário.


Uma informação importante também é o TTL (Time to live) do registro DNS consultado. Conforme mencionado anteriormente, o host mantém um cache local. O TTL do registro informa ao seu computador quando parar de considerar o registro como válido – ou seja, quando ele deve solicitar os dados novamente, em vez de depender da cópia em cache. Esse TTL é medido em segundos. No exemplo capturado podemos ver que é 60 segundos de TTL.

Resumo da Relação:

- Servidor recursivo: inicia a consulta e segue os passos até obter a resposta final.
- Servidores root: são os primeiros pontos de contato e informam onde encontrar os servidores TLD.
- Servidores TLD: sabem onde encontrar os servidores autoritativos para domínios específicos.
- Servidores autoritativos: fornecem a resposta final com os dados reais do domínio (como o IP).

Exemplo mostrando com o comando dig (mais alguns filtros para melhorar a saida).

 ![](assets/img/posts/post-22/18.png)

Esse processo é muito rápido, apesar de envolver múltiplos passos, e é a base da navegação na web.

Mais informações sobre DNS em:

<https://blog.justincarver.work/decoding-dns-understanding-and-troubleshooting-dns-fundamentals>

<https://wiki.brasilpeeringforum.org/w/DNS_Recursivo_Anycast_Hyperlocal>