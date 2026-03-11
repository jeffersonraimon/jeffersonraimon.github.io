---
title: "03 | ESTUDO: Manipulação de Rotas via BGP"
date: 2024-07-25 00:00:00 -0300
categories: [Artigos]
tags: [BGP, Routing, Estudos]
image:
  path: assets/img/posts/post-03/capa.png
---

Postado originalmente no linkedin em: 16/03/2024

A confiabilidade e eficiência das comunicações de rede são fundamentais para a operação bem-sucedida de sistemas modernos. No entanto, em ambientes de rede complexos, como a Internet, a ocorrência de perdas de pacotes pode ser inevitável, resultando em interrupções temporárias ou degradação da qualidade do serviço. O Border Gateway Protocol (BGP) é o protocolo de roteamento dinâmico principal utilizado para facilitar a comunicação entre sistemas autônomos (AS) na Internet. O administrador da rede também pode se valer de realizar manipulações de forma manual em caso de necessidade perante essas problemáticas.

Neste estudo, criei um cenário típico de ambiente ISP em que possuo algumas rotas para um determinado destino. Mediante alguns desafios como por exemplo perdas de pacotes, faremos algumas manipulações de rotas com o BGP para desviar o tráfego para contornar o problema (mesmo que temporariamente).

**_Disclamer: Foi utilizado um cenário e configurações simples, sem muito aprofundamento. O estudo a seguir foi feito com observações, análises práticas e pesquisas teoricas externas, caso encontre alguma informação divergente/equivocada, por favor informe nos comentarios para ser feito a correção._**

Topologia:

![](assets/img/posts/post-03/01.png)


Topologia do ambiente ISP
Nós somos o AS com o ASN 10 e temos como objetivo proporcionar ao PC-CLIENTE acesso de qualidade ao Servidor Web que se encontra no AS30. Os outros AS de transito possuem suas redes também, mas não será nosso foco.

![](assets/img/posts/post-03/02.png)

AS10
Informações de configurações do AS 10

![](assets/img/posts/post-03/03.png)
![](assets/img/posts/post-03/04.png)


_RT-1_

![](assets/img/posts/post-03/05.png)
![](assets/img/posts/post-03/06.png)

_RT-2_


Perguntas que podem surgir:

Por que no iBGP o Nexthop deve ser configurado como self?

No exemplo da rota 2001:db8:20::/64 que consta no RT-1, o RT-2 não conhece o IP fd10:10:20::1, que seria o gateway para essa rede, e não ocorreria comunicação. No entanto, o RT-1 conhece esse IP, pois é o endereço de uma rede diretamente conectada a ele.

![](assets/img/posts/post-03/07.png)

Por que o RT1 não está anunciando sua rota para 2001:db8:30::/64 via AS20 para o RT-2?

Por padrão, o BGP anuncia apenas as rotas que estão na FIB (Forwarding Information Base). No cenário em que o RT-1 está recebendo a rota 2001:db8:30::/64 via iBGP e foi definido como o melhor caminho devido ao atributo AS-PATH, o RT-1 não irá anunciar essa rota para o RT-2.

Como prova disso, podemos eliminar a rota via AS30 do RT-2 e observar que iremos receber a rota via RT-1. Esse comportamento também pode ser alcançado aumentando a local-preference no RT-1, priorizando assim a rota recebida por ele.

![](assets/img/posts/post-03/08.png)

Informações de configurações do AS30

RT-WEB

![](assets/img/posts/post-03/09.png)
![](assets/img/posts/post-03/10.png)


_AS 30_


Explicado a topologia, agora vamos de mão na massa!

Cenário: Perdas de pacote

PC-CLIENTE (AS 10)

![](assets/img/posts/post-03/11.png)

_Navegando no servidor WEB e teste de traceroute para o destino_

Realizando um traceroute podemos notar que está ocorrendo tudo bem e estamos alcançando o destino via IX.

Agora simulando perdas de pacote em algum ponto.

![](assets/img/posts/post-03/12.png)

Podemos notar que as perdas estam estão ocorrendo no enlace com o IX pois está ocorrendo perdas no ponto a ponto fd10:10:30::/127

Visualizando com captura de pacotes

![](assets/img/posts/post-03/13.png)

_Wireshark – Sem resposta para esse pacote em questão_

Vamos utilizar manipulação de rotas com o BGP para remover o tráfego pelo IX e amenizar os impactos.

Irei atuar primeiramente no RT-1 e desviar o tráfego para o AS20, utilizando aumento do local-preference de 200 para a rede de destino.

![](assets/img/posts/post-03/14.png)

_Adicionado regra para aumentar o local-preference em 200_


Verificando o efeito

![](assets/img/posts/post-03/15.png)


Verificando no cliente, constatamos que houve a mudança da rota pelo AS20, porém ainda ocorrem perdas de pacotes. Mas por quê?

Isso acontece porque apenas alteramos nosso upload, mantendo inalterado o caminho para o download. Dessa forma, o servidor web ainda nos alcança via IX.

Por sorte, o AS30 possui looking glass e será de grande valia no troubleshoot.

![](assets/img/posts/post-03/16.png)

_Validamos que o download está vindo via IX._

Entendendo manipulação de download e upload com BGP

Todos os anúncios que fizermos, determinam nosso DOWNLOAD, e todos os anúncios que recebermos, determinam nosso UPLOAD.

Quando divulgamos nossos anúncios, esses interferem em como o mundo nos alcança, refletindo no recebimento de tráfego, no caso o DOWNLOAD.

Quando recebemos anúncios, esses interferem em como alcançamos o mundo, refletindo no envio de tráfego, no caso o UPLOAD.

Manipulando trafego de Upload

Existem duas formas principais, com BGP, de fazer com que um determinado link seja prioritário para upload de algum destino específico: weight e local-preference.

Para ambos os parâmetros, quanto maior o valor, mais preferível uma rota será. Entretanto o parâmetro weight não é suportado por todos os fabricantes, sendo assim, é melhor a utilização do local-preference.

![](assets/img/posts/post-03/17.png)

Manipulação tráfego de Download

Para isso existem básicamente duas formas, anunciar o prefixo precisa mais ou menos específico pelo link desejado ou ter um AS-PATH menor.

Exemplo do anuncio mais ou menos especifico: estou anunciando meu prefixo /64 para todos os meus peers, porém, posso criar um prefixo /32 e anunciar para o neighbor AS 30, com isso o AS 30 irá ter na sua tabela de roteamento uma rede mais proxima do IP do destino via AS 50, tendo ela como melhor rota (/64).

Para manipular o AS-PATH podemos utilizar de Prepend, em que irá “mentir” para o peer informando que possui mais AS no caminho.

Criei essa imagem para visualizar de forma mais didática.

![](assets/img/posts/post-03/18.png)

Agora que entendemos que precisamos realizar manipulações tanto no download quanto no upload, vamos seguir.

Irei utilizar Prepend no RT-2 para o AS-30.

![](assets/img/posts/post-03/19.png)

_Adicionando prepend de 4x_

![](assets/img/posts/post-03/20.png)

_Policy completa_

Agora, sem perdas

![](assets/img/posts/post-03/21.png)
![](assets/img/posts/post-03/22.png)

_O servidor Web agora nos alcança através do AS 50_

Com essas manipulações, podemos notar o seguinte:

AS Path do tráfego de UPLOAD: 10 > 20 > 30

AS Path do tráfego de DOWNLOAD: 30 > 50 > 40 > 10

![](assets/img/posts/post-03/capa.png)

_Na perspectiva do PC-Cliente – Em Azul: Upload e em Vermelho: Download_

Opção 2: Via AS 40

Por algum motivo eu não quero mais meu upload pelo AS 20. Para isso, posso dropar a rota via AS20 no RT-1 e aumentar o local preference do peer AS 40 no RT-2.

![](assets/img/posts/post-03/23.png)

_Alterações no RT-1_

![](assets/img/posts/post-03/24.png)

_Alteração no RT-2 – Aumentando local-preference no AS40_

![](assets/img/posts/post-03/25.png)


Agora estamos tendo tráfego de Up e Down pelo mesmo path

Após verificarmos que está tudo normal com o IX, podemos reverter as configurações.

Conclusão:

É evidente que não existe uma abordagem única ou universal para lidar com perdas de pacotes, uma vez que as condições da rede podem variar amplamente e as necessidades dos usuários podem ser diversas. No entanto, as estratégias (com BGP) discutidas oferecem um conjunto valioso de ferramentas para os administradores de rede enfrentarem esses desafios.

Como foi simulado perdas de pacotes:

![](assets/img/posts/post-03/26.png)
![](assets/img/posts/post-03/27.png)

_Adicionado 2% de Loss (PNETLAB)_

Referências:

[Como fazer com que um determinado conteudo saia por um link especifico](https://wiki.brasilpeeringforum.org/w/Como_fazer_com_que_um_determinado_conteudo_saia_por_um_link_especifico)

[Manipulando Trefego BGP](https://mum.mikrotik.com/presentations/BR17/presentation_4920_1510758756.pdf)