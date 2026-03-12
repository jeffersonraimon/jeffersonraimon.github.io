---
title: "15 | Equipamento resetado: Recuperando o acesso remotamente com NAT"
date: 2024-09-03 00:00:00 -0300
categories: [Tutoriais]
tags: [NAT, Networking, Troubleshooting]
image:
  path: assets/img/posts/post-15/capa.png
---

Já passei por uma situação em que precisei recuperar o acesso a um equipamento que foi restaurado para as configurações de fábrica. Consegui fazer isso utilizando o NAT do RouterOS. Foi bastante útil, pois não precisei conectar fisicamente ao equipamento, configurar o IP na mesma faixa da rede padrão na placa de rede, acessar o dispositivo e reconfigurá-lo manualmente.

Vamos entender como isso foi feito:

Topologia:

![](assets/img/posts/post-15/capa.png)

Podemos observar que o switch estava na mesma rede que o meu computador. No entanto, após ser resetado, o switch voltou às configurações padrão de IP e não tinha um gateway configurado, o que resultou na perda de acesso ao equipamento.

Para contornar esse problema, realizei o seguinte procedimento.

1 – Removi a interface ether2 da bridge (não precisava mas fiz assim mesmo por segurança)

2 – Adicionei um IP na interface na mesma faixa da rede padrão do SW (192.168.0.254)

![](assets/img/posts/post-15/01.png)

3- Utilizei o IP Scan do Mikrotik para identificar o IP

![](assets/img/posts/post-15/02.png)

4 – Criei uma regra de NAT para alterar meu IP de origem ao acessar a interface web do switch
``` 
/ip firewall nat add action=src-nat chain=srcnat dst-address=192.168.0.1 out-interface=ether2 protocol=tcp dst-port=80 src-address=10.0.0.0/24 to-addresses=192.168.0.254 to-ports=80
``` 
Essa linha de comando configura o roteador para alterar o endereço IP de origem dos pacotes TCP que vêm da sub-rede ``` 10.0.0.0/24```  e estão indo para o endereço ``` 192.168.0.1```  através da interface ``` ether2``` . O endereço IP de origem será modificado para ``` 192.168.0.254``` , e a porta de destino a 80 devido o acesso ser a interface WEB HTTP do switch.

Após isso consegui acessar a interface WEB do switch e realizar a reconfiguração. Depois disso é só desfazer os procedimentos.

Bônus
Vou simular o mesmo cenário em um laboratório para facilitar a compreensão. Capturando os pacotes podemos entender como aconteceu o processo.

![](assets/img/posts/post-15/03.png)
![](assets/img/posts/post-15/04.png)

PC > Roteador

Podemos ver que a origem é o IP da rede ``` 10.0.0.0/24```  com destino ao IP do Apache.

![](assets/img/posts/post-15/05.png)

Roteador > Servidor Apache

Agora vemos que o roteador alterou o IP de origem e conseguimos validar que se trata do mesmo pacote devido ao valor de Identification.

![](assets/img/posts/post-15/06.png)


Fim!