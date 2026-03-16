---
title: "18 | Qualidade de Serviço (QoS) em Mikrotik: Marcação de Pacotes e Queue"
date: 2024-09-25 00:00:00 -0300
categories: [Artigos]
tags: [QoS, Mikrotik, Networking]
image:
  path: assets/img/posts/post-18/capa.png
---

A Qualidade de Serviço (QoS) é um conjunto de técnicas que garante a performance e a eficiência de uma rede, priorizando o tráfego de dados mais sensível a atrasos, como o utilizado em jogos online, videoconferências e transmissões ao vivo. No contexto do Mikrotik, a implementação de QoS pode ser feita através da marcação de pacotes e da configuração de queues, assegurando que a largura de banda seja utilizada de maneira otimizada.

Não irei entrar em detalhes sobre QoS, pois você pode se aprofundar no tema pelo excelente artigo do Brasil Peering Forum: <https://wiki.brasilpeeringforum.org/w/Adocao_de_Quality_of_Service_em_Ambientes_de_Operadores_de_Redes>.

Uma forma também de reduzir ou eliminar latência da sua rede interna (Bufferbloat) com QoS abordo no artigo 01. Porém nesse texto, abordei o Queues de forma mais ampla. Agora, vamos explorá-lo em conjunto com a marcação de pacotes. Podemos considerar como uma parte 2 (mais avançada).

Marcação de Pacotes

A marcação de pacotes permite classificar e identificar o tráfego que precisa de prioridade. No Mikrotik, isso pode ser realizado utilizando mangle, onde você pode criar regras que marcam pacotes com base em critérios como protocolo, endereço IP ou porta. Para aplicações em tempo real é fundamental marcar esse tráfego, garantindo que a aplicação receba tratamento preferencial na fila de dados.

Configuração de Queues

Depois de marcar os pacotes, o próximo passo é configurar as queues. As queues no Mikrotik permitem controlar a quantidade de largura de banda alocada para diferentes tipos de tráfego, reduzindo a latência e evitando quedas durante a jogatina por exemplo ou travamentos em streamings.

Nota sobre o FastTrack

Uma consideração importante ao implementar QoS no Mikrotik é o uso do recurso FastTrack. O FastTrack acelera o processamento de pacotes, mas pode interferir na marcação e no gerenciamento de queues. Para garantir que a QoS funcione adequadamente, é recomendável desativar o FastTrack ou configurá-lo em modo “parcial”. Assim, você consegue manter o controle sobre o tráfego de rede, priorizando as aplicações mais críticas, como jogos.

Modo pacial? Sim, básicamente especificar qual a interface que o FasTrack irá ignorar. Por exemplo:
``` 
/interface list add name=WAN
/interface list add name=WIFI

/interface list member add interface=PPPOE-CLIENT list=WAN
/interface list member add interface=20-DECO-WIFI-DHCP list=WIFI

/ip firewall filter add action=fasttrack-connection chain=forward comment=FASTTRACK connection-state=established,related hw-offload=yes in-interface-list=WAN out-interface-list=!WIFI
```
```in-interface-list=WAN```: A regra se aplica ao tráfego que entra pela lista de interfaces WAN.

```out-interface-list=!WIFI```: O tráfego que sai não deve ir para a lista de interfaces WIFI, ou seja, ele deve ser direcionado a outras interfaces.

Dessa forma o FasTrack fica em modo parcial e poderemos aplicar Queues na rede WiFi.

Estudo de Caso: Implementação de QoS para Redução de Latência no jogo Wild Rift
Cenário
Neste estudo de caso com o jogo Wild Rift da Riot Games, utilizei um iPhone XR conectado na rede WiFi 5Ghz de um Deco M4R (junto com uns 9 aparelhos Wireless utilizando a rede no momento) gerenciados pelo roteador Mikrotik RB750GR3 RouterOSv7. O objetivo é otimizar a experiência de jogo, minimizando problemas de latência e jitter.

![](assets/img/posts/post-18/01.jpg)

## Problema

Atualmente, a latência para os servidores do Wild Rift, localizados em São Paulo, varia entre 24 e 34 ms de onde eu moro (na rede cabeada). Embora esses valores sejam geralmente aceitáveis, durante as sessões de jogo, a latência as vezes aumenta rapidamente, causando variações de jitter que prejudicam a jogabilidade. Essas oscilações podem resultar em atrasos nas respostas e em uma experiência de jogo insatisfatória e muita das vezes.

![](assets/img/posts/post-18/02.png)


Site: <https://lagreport.br.leagueoflegends.com/pt>

## Solução

Para resolver esse problema, irei implementar QoS (Qualidade de Serviço) com a marcação de pacotes no Mikrotik. A ideia é priorizar o tráfego específico do Wild Rift, garantindo que os pacotes do jogo tenham preferência sobre outras atividades na rede.

## Identificando os servidores:

Para identificar os endereços IP dos servidores do Wild Rift, utilizei o Packet Sniffer do RouterOS durante uma partida. Esse recurso permitiu filtrar os pacotes que o iPhone XR estava utilizando para se conectar aos servidores do jogo. Após a captura, validei o IP encontrados no site bgp.he.net, garantindo que estava utilizando os endereços corretos e anotar quais blocos a Riot usam no Brasil. Em seguida, criei uma Address List no Mikrotik contendo esses prefixos, facilitando a marcação e o gerenciamento do tráfego.

OBS: Muito importante capturar os pacotes DURANTE a partida, pois se for no menu do jogo ou Lobby pode acabar obtendo IPS de servidores CDN como a Akamai.

OBS: Para serviços como youtube, google, instagam etc, apenas colocando o dominio no address-list o MK se encarrega de obter os endereços IPS de forma dinamica, contanto que o DNS esteja configurado em IP > DNS.

![](assets/img/posts/post-18/03.png)

Fiz o download do pcap e abri no Wireshark. Também é possivel usar o Torch ao invés do Packet Sniffer caso queira.

![](assets/img/posts/post-18/04.png)

Verificando os blocos no site da Hurricane.

![](assets/img/posts/post-18/05.png)
![](assets/img/posts/post-18/06.png)


Adicionei todos os blocos na Address Lists.

```
/ip firewall address-list add address=45.7.36.0/22 list=LOL
/ip firewall address-list add address=45.7.39.0/24 list=LOL
/ip firewall address-list add address=45.7.37.0/24 list=LOL
/ip firewall address-list add address=45.7.38.0/24 list=LOL
```
![](assets/img/posts/post-18/07.png)

Marcação de Pacotes: Utilizei regras de mangle na chain forward para identificar e marcar os pacotes que pertencem ao Wild Rift, com base origem a interface list da rede WiFi e destino a address list criada anteriomente com os prefixos da Riot. Isso nos permitirá segmentar o tráfego do jogo de forma eficaz. Podemos ver que está funcionando quando começa a contabilizar bytes enquanto está jogando.

```
/ip firewall mangle add action=mark-packet chain=forward dst-address-list=LOL in-interface-list=WIFI new-packet-mark=LOL-INGAME passthrough=yes comment="LOL WILD RIFT"
```
![](assets/img/posts/post-18/08.png)

Configuração de Queues: Após marcar os pacotes, configurei uma queue que prioriza esse tráfego, assegurando que ele receba a largura de banda necessária.
Para isso eu utilizei a ```Queue Tree``` pois funciona melhor e a diferença dela para a Simple é que não é necessário ter marcações separadas para download e upload; apenas o upload será direcionado para a ```interface list WAN``` e apenas o download será direcionado para a ```interface list WIFI```.

Estou usando o FQCODEL como o tipo de queue conforme ensinei neste artigo. E a priore limitando a 2MB para teste.
```
/queue tree add limit-at=2M max-limit=2M name=queue1 packet-mark=LOL-INGAME parent=global queue=fq_codelx
```
Alteração do FastTrack: Para garantir que as regras de QoS funcionem corretamente, foi colocado em modo parcial, conforme explicado anteriormente.

Validação

Após aplicado o QoS o jogo se manteve em 30ms fixo praticamente, sem alterações bruscas, mesmo na rede WiFi e com outros aparelhos consumindo a rede. Resolvido o problema!.

![](assets/img/posts/post-18/09.jpeg)

Testes
Como teste extremo, mudei o limite de banda para 1k e o jogo praticamente travou devido a alta latência.

![](assets/img/posts/post-18/10.jpeg)
![](assets/img/posts/post-18/11.png)

Coloquei 8k de banda e a latência ficou na casa dos 30ms e variando até 100ms as vezes.

![](assets/img/posts/post-18/12.jpeg)
![](assets/img/posts/post-18/13.png)

Por ultimo, coloquei 16k e o jogo se manteve também em 30ms e variando para 34m pouquissimas vezes. Ou seja, até menos de 1M da para melhorar a jogatina.

Conclusão
Ao aplicar estas configurações de QoS, mantivemos a latência estável e sem alterações no jitter, melhorando a experiencia de jogo.

Bônus
Script para habilitar e desabilitar essas configurações quando for jogar você pode encontrar aqui.

Referências
<https://help.mikrotik.com/docs/display/ROS/Queues#Queues-Configurationexample.1>

{% include embed/youtube.html id='2S9qZSRKuzs' %}

{% include embed/youtube.html id='LELIuNeQS-E' %}