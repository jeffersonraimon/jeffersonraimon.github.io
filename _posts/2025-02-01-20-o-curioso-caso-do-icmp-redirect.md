---
title: "20 | O curioso caso do ICMP Redirect"
date: 2025-02-01 00:00:00 -0300
categories: [Artigos]
tags: [ICMP, Routing, Networking]
image:
  path: assets/img/posts/post-20/capa.png
---

Fiz um laboratório no PnetLab e me deparei com uma situação interessante envolvendo uma topologia específica: o ICMP Redirect. Neste post, vou mostrar com mais detalhes o que aconteceu.

## Topologia

Fiz uma topologia mais simples para exemplificar a situação.

![](assets/img/posts/post-20/capa.png)
 

*Pergunta: O que vai acontecer com os pacotes ICMP ao realizar um ping do Linux (```100.100.1.2```) para o RT-10 (```10.1.1.1```) considerando essa topologia?*

Primeiro vamos conferir como estão as tabelas de roteamento em todos os hosts.

LINUX

![](assets/img/posts/post-20/01.png)

MK

![](assets/img/posts/post-20/02.png)

RT-12

![](assets/img/posts/post-20/03.png)

RT-11

![](assets/img/posts/post-20/04.png)

RT-10

![](assets/img/posts/post-20/05.png)

Como podemos ver, nada fora do padrão, terá conectividade normalmente, porém na rede ```100.100.1.0/24``` ocorrerá a situação do ICMP Redirect.

RESPOSTA

O Gateway do Linux (MK nesse caso) não irá rotear o pacote e vai responder com um ICMP Redirect informando o IP do GW dele (```100.100.1.254``` – Pois ambos estão na mesma rede local e o linux poderá alcançar esse IP).

![](assets/img/posts/post-20/06.png)

Podemos validar capturando os pacotes

Quando o Linux mandou o ICMP request (2), o MK respondeu com o ICMP Redirect (3) com o IP do gateway dele. (Com isso o Linux fará um ARP request para pegar o MAC).

![](assets/img/posts/post-20/07.png)

No pacote ICMP Request seguinte (4) podemos ver que o MAC de destino agora é do RT-12 (```100.100.1.254```).

![](assets/img/posts/post-20/08.png)

No pacote ICMP Reply (5) podemos ver que o MAC de origem é do RT-12 (```100.100.1.254```)

![](assets/img/posts/post-20/09.png)

Também podemos ver que sempre o cliente vai tentar com o MAC do seu GW, porém nunca terá resposta ICMP até que envie para o novo GW.

![](assets/img/posts/post-20/10.png)

A pegadinha é que pode nem mostrar no traceroute/mtr

![](assets/img/posts/post-20/11.png)

Fim!