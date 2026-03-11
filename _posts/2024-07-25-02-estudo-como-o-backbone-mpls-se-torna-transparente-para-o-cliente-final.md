---
title: "02 | ESTUDO: Como o Backbone MPLS se torna transparente para o cliente final?"
date: 2024-07-25 00:00:00 -0300
categories: [Artigos]
tags: [MPLS, Backbone, Networking]
image:
  path: assets/img/posts/post-02/capa.png
---

Postado originalmente no linkedin em: 11/02/2024

O MPLS é uma tecnologia necessária (e imprescindível) nos dias de hoje. Imagina você ter toda uma infraestrutura de multilayer switchs e/ou roteadores assim:

![](assets/img/posts/post-02/01.png)

E para o cliente final ser algo parecido com isso:

![](assets/img/posts/post-02/02.png)


Ou até mesmo mais simples, pois a maquina do cliente na filial vai estar no mesmo dominio de broadcast (mesma LAN) que a maquina servidor na Matriz, como se tivesse diretamente conectada. Mas como isso é possivel? É mágica? Vamos entender…

Disclamer: não irei entrar em detalhes de todo o funcionamento dessa tecnologia e vou partir do principio que você já possui noções e conhecimentos básicos de redes/MPLS. O estudo a seguir foi feito com observações, análises práticas e pesquisas teoricas externas, caso encontre alguma informação divergente/equivocada, por favor informe nos comentarios para ser feito a correção.

MPLS

O segredo do MPLS (Multiprotocol Label Switching) está na forma com que os frames são encaminhados. Esqueça a figura do frame ter que ir para camada 3 (virando um pacote) e consumir processamento da CPU ao consultar a tabela de roteamento. Estamos falando de labels, que são “anexadas” antes dos pacotes IP, agilizando muito o processo, já que o IGP (OSPF, por exemplo) já cumpriu seu papel de nos fornecer os melhores caminhos para as redes do backbone. Após isso, o MPLS designa labels para essas redes, formando um tipo de circuito virtual. Vejamos:

![Tabela de roteamento (esquerda) e tabela de encaminhamento (direita)](assets/img/posts/post-02/03.png)


Temos acima ao lado esquerdo, a tabela de roteamento preenchida pelo OSPF e ao lado direito, a tabela de encaminhamento, onde podemos notar que de fato o MPLS utiliza as rotas providas pelo IGP.

Adicionando esses labels após as rotas fornecidas pelo IGP, o MPLS já tem toda a topologia da rede mapeada. Vamos ver por exemplo do sentido RT1 para o RT8 ( ou seja da loopback 10.1.1.1 até 10.8.8.8 ) qual foi o ”caminho” marcado pelos labels.

![Rotas para 10.8.8.8 na pespectiva de RT-1](assets/img/posts/post-02/04.png)


Pelo IGP, vemos que possuimos 3 caminhos disponivel, sendo considerado o melhor a terceira opção (com o asterisco) via 10.127.12.2.  Dessa forma se o cliente mandasse um pacote destinado a maquina servidora atras o equipamento que dispoe da loopback 10.8.8.8, esse pacote teria que passar pelo processo de roteamento indo para a camada 3, o que seria mais lento e demandaria processamento de CPU. Porém, vamos ver o trabalho do MPLS:


![Tabela de encaminhamento para o destino 10.8.8.8](assets/img/posts/post-02/05.png)

Podemos ver os mesmos 3 caminhos e principalmente o label de saida correspondente ao melhor caminho designado pelo OSPF, o label 214.

Agora vamos ver como ficou o LSP do destino 10.8.8.8, no sentido RT-1 > RT-8

![Ida RT1 > RT8:](assets/img/posts/post-02/06.png)

![Caminho do traceroute](assets/img/posts/post-02/07.png)

![RT-5](assets/img/posts/post-02/08.png)

Vimos que o RT5 tem como label de saida o POP Label, ou seja, ao encaminhar o frame para o destino, ele remove o label e com isso o pacote chega em RT8 em forma de pacote IP tradicional indo para a camada 3 e sendo lida pelo mesmo.

Por curiosidade, a volta RT8 > RT1:

![](assets/img/posts/post-02/09.png)


![Captura de um frame com cabeçalho MPLS](assets/img/posts/post-02/10.png)

Capturando um frame por exemplo, podemos ver como onde fica o cabeçalho de controle do MPLS, conhecido como Shim Header, ela fica no meio dos cabeçalhos de camada 2 e 3, e por isso dizem ser camada 2.5. Para nossa análise, os campos MPLS Label e MPLS Bottom Of Label Stack serão os mais uteis

MPLS Label: Utilizado para a decisão de encaminhamento

MPLS Bottom Of Label Stack: Bit do empilhamento dos cabeçalhos, 0 – Infraestrutura (LSP – Label Switching Path), 1 – Serviço (VPN de camada 2)

VPN de camada 2 no Backbone

Vamos finalmente utilizar o MPLS em conjunto com VPN de camada 2 (VPWS – Virtual Private Wire Service) para prover conectividade do cliente da filial com a matriz, por meio do nosso backbone que será transparente para ele.

Bom, como a matriz fica atrás da RT8, fizemos com que o RT01 seja vizinho LDP do RT08 anteriormente, pois eles não estão diretamentes conectados.

![LDP Neighbors](assets/img/posts/post-02/11.png)


Para transportar os pacotes da maquina cliente da filial para a matriz, fechamos uma VPN de camada 2 entre o RT8 e RT1 passando a vlan do cliente (vlan 10)


![VPWS Up](assets/img/posts/post-02/12.png)

![Teste da conectividade do VPWS](assets/img/posts/post-02/13.png)


![Representação do túnel VPWS](assets/img/posts/post-02/14.png)

Configuração no cliente

IPs configurados nos hosts e podemos visualizar os respectivos MACs e os switchs do cliente apenas recebem a vlan 10 como tag e remove a tag ao encaminhar para a LAN:

PC CLIENTE: IP 172.25.0.1 / MAC: 50:E7:6A:00:3C:00

PC SERVIDOR: IP 172.25.0.10 / MAC: 50:C3:E9:00:41:00


![Configuração de IPs no cliente e servidor](assets/img/posts/post-02/15.png)


![Configuração da VLAN 10 nos Switchs do cliente](assets/img/posts/post-02/16.png)


![Teste de conectividade entre o PC e o Servidor do cliente](assets/img/posts/post-02/17.png)

Descobrindo o segredo por trás da mágica

Vamos analisar passo a passo o caminho dos pacotes ICMP acima e entender finalmente como todo o backbone se tornou transparente para o cliente final.

Sentido: SW-CLIENTE-FILIAL > RT-1 > RT-4 > RT-5 > RT-8 > SW-CLIENTE-MATRIZ

Ao capturar os pacotes a interface interna da rede LAN do Switch (e0/1) na Filial, podemos ver frames ARP com MAC Address da maquina Servidora na Matriz, provando que todo o backbone se tornou transparente para o cliente.


![Frames ARP](assets/img/posts/post-02/18.png)


![ICMP](assets/img/posts/post-02/19.png)

Apenas como comparação, em uma rede roteada (sem MPLS), o MAC que veriamos seria do Roteador do ISP (RT-01), pois ele seria o gateway do cliente e roteadores não transpoem camada 2. Conforme imagem de exemplo abaixo, o roteador ao ler o conteudo em camada 3  (passo 3) e realizar o roteamento (passo 4), ele adiciona na camada 2 o proprio endereço MAC da sua interface física (passo 5).


![Roteamento na visão do Modelo OSI](assets/img/posts/post-02/20.png)

Ao capturar os pacotes na interface externa do Switch da Filial (e0/0) podemos ver a VLAN 10 que o provedor nos designou para realizar o transporte. Até aqui tudo bem…


![VLAN ID 10 no campo 802.1Q Virtual LAN](assets/img/posts/post-02/21.png)

RT-1 (Gi0/2.14)

Capturando os pacotes na interface de saida do RT-1 onde ele está encaminhando os frames, já podemos acompanhar a mágica acontecendo. É possivel ver o encapsulamento Dot1q da vlan 14 da interface, e o principal: foi criado uma pilha (stack) de cabeçalho MPLS com 2 labels, 408 e 800, sendo o 408 referente a infra (LSP – Label Switching Path) e 800 o de serviço da VPN, conforme o campo MPLS Bottom of Label Stack

![Captura ICMP](assets/img/posts/post-02/22.png)


Abrindo o cabeçalho do 802.1Q, podemos ver que o campo EtherType está definido como 0x8847, fazendo com que a consulta não considere a tabela de encaminhamento de pacotes IP, e sim uma tabela específica para operações com labels. (a Forwarding-Table do MPLS)

![](assets/img/posts/post-02/23.png)

Verificando no RT1, podemos ver que o label 408 se trata do label correspondente para o neighbor LDP 10.8.8.8 via Gi0/2.14.


![Tabela de encaminhamento para 10.8.8.8 no RT-1](assets/img/posts/post-02/24.png)

RT-4 (Gi0/1.45)

Na interface de saida do RT-4, podemos notar como mudança: as alterações de MAC do frame, o encapsulamento dot1q da vlan 45 da interface e a alteração do label da stack 0 (LSP), label 510


![Mudanças no cabeçalhos MPLS ao ser encaminhado por RT-4](assets/img/posts/post-02/25.png)

Verificando no RT-4


![Tabela de encaminhamento correspondente ao label 408 no RT-4](assets/img/posts/post-02/26.png)

RT-5 (Gi0/0.58)

No RT-5, podemos ver que ao encaminhar o frame para o RT-8, por ser o destino final da VPN, ele remove o cabeçalho LSP, ficando apenas o do serviço VPN.


![Captura em que consta a remoção do cabeçalho LSP](assets/img/posts/post-02/27.png)

Podemos validar que no destino 10.8.8.8, temos um Pop Label (remoção da label da infraestrutura)

![Tabela de encaminhamento correspondente ao label 510 no RT-5 com Pop Label](assets/img/posts/post-02/28.png)


RT-8 (Gi0/3.10)

Finalmente no RT-8, podemos ver que ao ser encaminhado para o cliente, foi removido a label 800 referente ao serviço da VPN de camada 2, e validando no roteador em Outgoing Label, não existe label para esse destino, acabando aqui o encapsulamento do MPLS e com isso o payload é entregue via Gi0/3.10 constando os endereços de camada 2 dos hosts do cliente, estando no mesmo dominio de broadcast.


![Captura RT-8 entregando frame ao switch do cliente na Matriz](assets/img/posts/post-02/29.png)



![Label 800 referente ao VPWS no RT-8](assets/img/posts/post-02/30.png)

Por curiosidade, nesse demonstração, a resposta ICMP foi por outro caminho, RT-8 > RT-6 > RT-2 > RT1.


![Captura da resposta ICMP no RT-8](assets/img/posts/post-02/31.png)

Verificando no RT-1 o label 100


![Label 100 referente ao VPWS no RT-1](assets/img/posts/post-02/32.png)

De forma didática e simples , seria mais ou menos assim todo o processo (pacote ida):


![Imagem didática do processo – Fonte: Autor](assets/img/posts/post-02/capa.png)

### Conclusão

Conforme vimos com esse estudo, ao analisarmos o MPLS, descobrimos como essa tecnologia atua em conjunto com o IGP e a mágica por trás da transparência do backbone para o cliente final.

—

Autor: Jefferson Raimon

Referências

Labs feitos no PNETLAB v6, utilizando Cisco vIOS 15, Cisco IOL, e Linux Debian

https://www.cisco.com/c/en/us/td/docs/ios_xr_sw/iosxr_r3-7/mpls/command/reference/gr37fwd.html

https://www.youtube.com/watch?v=b_n7A6Pejak

https://www.youtube.com/watch?v=YFOBLyf2SG0

https://community.cisco.com/t5/blogues-de-routing-switching/o-que-%C3%A9-mpls-e-por-que-mpls/ba-p/4284475

https://www.cisco.com/c/pt_br/support/docs/multiprotocol-label-switching-mpls/mpls/213238-mpls-l2vpn-pseudowire.html