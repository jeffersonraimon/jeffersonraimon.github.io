---
title: "27 | MikroTik sem conectar ao servidor NTP"
date: 2025-11-18 00:00:00 -0300
categories: [Tutoriais]
tags: [Mikrotik, NTP, Troubleshooting]
image:
  path: assets/img/posts/post-27/capa.png
---

Em muitos cenários, especialmente em conexões de provedores menores, pode acontecer do tráfego na porta UDP 123 — usada pelo protocolo NTP — ser filtrado ou bloqueado.
O resultado: o MikroTik simplesmente não consegue sincronizar o horário, mesmo configurado corretamente.

Aqui em casa passei por essa situação: o NTP não conectava de jeito nenhum. Depois de verificar firewall, DNS, rotas e servidores NTP, a única explicação lógica era bloqueio da porta 123 pelo provedor.

![](assets/img/posts/post-27/capa.png)


A solução: alterar a porta de saída usando NAT

Como o Mikrotik envia requisições NTP sempre na porta UDP 123, se o provedor estiver filtrando esse tráfego, podemos “driblar” o bloqueio modificando a porta de origem antes do pacote sair para a internet.

Isso pode ser feito facilmente com uma regra de NAT:
``` 
/ip firewall nat add action=masquerade chain=srcnat comment="NTP NAT masquerade" dst-port=123 protocol=udp to-ports=11100-11350
``` 
O que essa regra faz?

- Ela pega pacotes UDP destinados à porta 123 (NTP)
- Aplica masquerade
- E altera dinamicamente a porta de saída, usando a faixa 11100–11350

Assim, para o provedor, o tráfego deixa de parecer “um pacote NTP padrão” e passa a ser apenas um pacote UDP comum — deixando de ser bloqueado.

![](assets/img/posts/post-27/02.png)

Resultado
Depois de aplicar a regra, o MikroTik imediatamente conseguiu sincronizar com os servidores NTP.

Problema resolvido!

![](assets/img/posts/post-27/03.png)
![](assets/img/posts/post-27/04.png)
