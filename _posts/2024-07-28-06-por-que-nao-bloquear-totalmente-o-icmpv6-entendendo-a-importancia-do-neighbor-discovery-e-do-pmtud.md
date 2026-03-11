---
title: "06 | Por que não bloquear totalmente o ICMPv6: Entendendo a importância do Neighbor Discovery e do PMTUD"
date: 2024-07-28 00:00:00 -0300
categories: [Artigos]
tags: [ICMPv6, IPv6, PMTUD]
image:
  path: assets/img/posts/post-06/capa.jpg
---

Com a crescente adoção do IPv6, é crucial entender o funcionamento e a importância dos protocolos associados a ele. O ICMPv6, muitas vezes visto como uma simples ferramenta de diagnóstico, desempenha um papel vital no ecossistema IPv6, especialmente no que diz respeito ao Neighbor Discovery (ND) e ao Path MTU Discovery (PMTUD). Neste artigo, explico por que bloquear o ICMPv6 pode ser uma decisão equivocada e como ele é essencial para o bom funcionamento das redes modernas.

O Que é ICMPv6?

O ICMPv6 (Internet Control Message Protocol for IPv6) é uma extensão do ICMP usado no IPv4, projetado para o IPv6. Ele é responsável por transmitir mensagens de erro, informar sobre condições de rede e desempenhar funções de diagnóstico. Enquanto o ICMPv4 era limitado em suas funcionalidades, sendo basicamente utilizado para diagnósticos de rede. e antigamente foi utilizado como métodos de ataques DDoS (Distributed Denial of Service), é fácil pensar porque ainda muitos profissionais acabam bloqueando erroneamente o ICMPv6, assumindo que desempenha o mesmo objetivo que o protocolo da versão anterior.

Porém, o ICMPv6 expande significativamente seu papel, tornando-se crucial para várias operações do IPv6.

O papel crucial do Neighbor Discovery (ND)
O Neighbor Discovery Protocol (NDP) é uma parte essencial do IPv6, utilizado para determinar a presença de outros dispositivos na mesma rede, descobrir roteadores, resolver endereços IPv6 em endereços MAC e mais. O NDP utiliza várias mensagens ICMPv6, incluindo Neighbor Solicitation, Neighbor Advertisement, Router Solicitation e Router Advertisement.

Verificação de IP duplicado: o ICMPv6 também é utilizado para verificar duplicidade de endereços IP na rede, um processo conhecido como Duplicate Address Detection (DAD). Esse processo é uma parte essencial do Neighbor Discovery Protocol (NDP) no IPv6.

Funcionamento: 

Quando é configurado um IPv6 no dispositivo, ele envia uma mensagem Neighbor Solicitation (NS) para esse mesmo endereço e caso receba a mensagem de resposta Neighbor Advertisement (NA), indica que o endereço já está em uso e caso não receba uma resposta NA dentro de um certo período de tempo, o device assume que o endereço é único e pode utilizá-lo.

Teste prático:

Temos 2 servidores Ubuntu (20.04.6 LTS) na mesma LAN e o primeiro teste será adicionar um endereço IPv6 ULA (Unique Local Address) no SRV-5 e verificar a mensagem Neighbor Solicitation (NS).

![](assets/img/posts/post-06/01.png)
![](assets/img/posts/post-06/02.png)


Podemos observar o pacote de nº 3, que o host enviou como origem o endereço IPv6 :: (endereço não especificado) e como destino um IPv6 do bloco Multicast (ff00::/8) uma mensagem NS e não teve retorno com NA.

Agora, adicionando o mesmo endereço no SRV-6, podemos ver o pacote de nº 6 que se trata da resposta NA do SRV-5, devido ao endereço MAC. Apesar do conflito, o linux manteve o IP na interface, porém ao realizar um teste de ping, não ocorre a comunicação e analisando os pacotes podemos ver que as mensagens NS e NA com os endereços MACs dos dois hosts, logo após, os hosts passaram a utilizar os seus endereços Link-Local (pacote nº 17 em diante) mas ainda sem sucesso na comunicação.

![](assets/img/posts/post-06/03.png)


Descoberta de Vizinhos e Resolução de Endereços MAC: Os dispositivos utilizam o NDP para descobrir outros dispositivos na rede local. Sem isso, a comunicação local seria impossível.

Teste prático:

![](assets/img/posts/post-06/04.png)


Aqui podemos ver a comunicação correndo normalmente e sendo feito a descoberta de vizinho com as mensagens NS e NA, semelhante ao protocolo ARP do IPv4 em que é informado o endereço MAC dos hosts correspondentes ao endereço IP

Autoconfiguração de Endereços: Permite que dispositivos obtenham endereços IP automaticamente, sem a necessidade de um servidor DHCP, processo esse conhecido como SLAAC (Stateless Address Auto-configuration).

Teste prático:

![](assets/img/posts/post-06/05.png)
![](assets/img/posts/post-06/07.png)




Podemos ver que para isso acontecer, é utilizado as mensagens ICMPv6 Router Solicitation (RS) e Router Advertisement (RA)

Resumo das Mensagens
Router Solicitation (RS):
- Enviado pelo host.
- Endereço de destino: ff02::2 (todos os roteadores).
- Solicita informações de configuração dos roteadores.

Router Advertisement (RA):

- Enviado pelos roteadores.
- Endereço de destino: ff02::1 (todos os nós).
- Contém informações como prefixo de rede, opções de configuração, tempo de vida do prefixo, e outros parâmetros relevantes.
Podemos verificar no pacote nº 95 que foi feito o DAD novamente.

Bloquear o ICMPv6 interrompe essas funções, resultando em problemas como a incapacidade de descobrir vizinhos e resolver endereços, o que pode levar a falhas de comunicação e redes ineficientes.

Path MTU Discovery (PMTUD)

O Path MTU Discovery (PMTUD) é um mecanismo que determina o maior tamanho de pacote que pode ser transmitido sem fragmentação ao longo do caminho entre a origem e o destino. Isso é fundamental para a eficiência da rede, pois evita a fragmentação excessiva dos pacotes, que pode reduzir o desempenho.

O ICMPv6 facilita o PMTUD enviando mensagens de erro quando um pacote é muito grande para ser transmitido sem fragmentação. Esses erros informam o remetente para reduzir o tamanho dos pacotes, ajustando dinamicamente a MTU e garantindo uma transmissão eficiente.

Bloquear o ICMPv6 impede que essas mensagens de erro sejam enviadas, resultando em pacotes que podem ser fragmentados ou descartados, causando lentidão e ineficiência na rede.

Testes práticos:

Teste 1: Packet Too Big

Cenário

Temos um cenário simples de cliente x servidor web com variações de MTU nos roteadores pelo caminho conforme imagem. O RT-3 tem bloqueio incorreto de ICMPv6 e vamos ver a consequência disso.

![](assets/img/posts/post-06/08.png)


Teste prático de navegação web:

No PC cliente, ao abrir o servidor web, podemos ver que a página web abriu “normalmente”, porém analisando os pacotes, podemos notar que houve retransmissão na camada de transporte. É possível verificar que o MSS foi de 1440 bytes, maior que o MTU de 1400 bytes dos roteadores e 1300 bytes do enlace do servidor com o gateway (para forçar o problema).


![](assets/img/posts/post-06/09.png)
![](assets/img/posts/post-06/10.png)

Revisão rápida de MSS:

O MSS (Maximum Segment Size) é o tamanho máximo de dados que um segmento TCP pode transportar. É calculado subtraindo o tamanho dos cabeçalhos IP e TCP do valor MTU (Maximum Transmission Unit).

Cálculo do MSS para IPv6
MTU: 1500 bytes
Cabeçalho IP: 40 bytes
Cabeçalho TCP: 20 bytes
MSS = MTU – (Cabeçalho IP + Cabeçalho TCP)

MSS = 1500 – (40 + 20)

MSS = 1500 – 60

MSS = 1440 bytes

Portanto, para um enlace com MTU de 1500 bytes, o MSS seria 1440 bytes para IPv6.

Teste prático de ping:

Ao realizar um teste de ping com o tamanho de 1500 bytes, recebemos do gateway um pacote ICMPv6 do tipo Packet Too Big, informando qual seria o MTU ideal.


![](assets/img/posts/post-06/11.png)
![](assets/img/posts/post-06/12.png)
![](assets/img/posts/post-06/13.png)

Porém como o RT3 está bloqueando ICMPv6 de forma incorreta, o PMTUD foi quebrado. Para conseguir comunicação temos que ir testando qual seria o maior MTU possível.

![](assets/img/posts/post-06/14.png)
![](assets/img/posts/post-06/15.png)


Após tentativas, vemos que o MTU seria 1258 bytes para conseguir o pong

Agora refazendo os testes com o desbloqueio do ICMPv6 no RT3

Retornado o MTU para 1500 do enlace do servidor x gateway para facilitar

![](assets/img/posts/post-06/16.png)

Teste prático de ping:

Automaticamente é feito ajuste do MTU devido ao PMTUD e conseguimos o pong sem ficar precisando testar outros valores de bytes

![](assets/img/posts/post-06/17.png)

OBS: Apesar de mostrar 1508 bytes, de baixo do capô houve a fragmentação devido ao enlace de 1400 bytes dos roteadores

![](assets/img/posts/post-06/18.png)

Teste prático de navegação web:

Podemos ver que o MSS foi ajustado para 1340 bytes

![](assets/img/posts/post-06/19.png)

Riscos e Considerações de Segurança

Algumas organizações bloqueiam o ICMPv6 por preocupações de segurança, temendo ataques como a inundação de ICMP (ICMP flood) ou a exploração de vulnerabilidades. No entanto, bloquear completamente o ICMPv6 não é a solução ideal.

Mitigando riscos sem bloquear:

Filtragem inteligente: Configurar regras de firewall para permitir o tráfego ICMPv6 legítimo, bloqueando apenas as mensagens suspeitas.
Exemplo:

Mikrotik:
```
/ipv6 firewall filter add action=accept chain=input comment="Allow ND Neighbor Solicitation" icmp-options=135:0 protocol=icmpv6 in-interface-list=WAN

/ipv6 firewall filter add action=accept chain=input comment="Allow ND Neighbor Advertisement" icmp-options=136:0 protocol=icmpv6 in-interface-list=WAN

/ipv6 firewall filter add action=accept chain=input comment="Allow Router Solicitation" icmp-options=133:0 protocol=icmpv6 in-interface-list=WAN

/ipv6 firewall filter add action=accept chain=input comment="Allow Router Advertisement" icmp-options=134:0 protocol=icmpv6 in-interface-list=WAN

/ipv6 firewall filter add action=accept chain=input comment="Allow Echo Request" icmp-options=128:0 protocol=icmpv6 in-interface-list=WAN

/ipv6 firewall filter add action=accept chain=input comment="Allow Echo Reply" icmp-options=129:0 protocol=icmpv6 in-interface-list=WAN

/ipv6 firewall filter add chain=input action=drop protocol=icmpv6 icmp-options=2:0 comment="Allow Packet Too Big" in-interface-list=WAN

/ipv6 firewall filter add chain=input action=drop protocol=icmpv6 icmp-options=3:0 comment="Allow Time Exceeded" in-interface-list=WAN

/ipv6 firewall filter add chain=input action=drop protocol=icmpv6 icmp-options=4:0 comment="Allow Parameter Problem" in-interface-list=WAN

/ipv6 firewall filter add action=drop chain=input comment="Drop other ICMPv6" protocol=icmpv6 in-interface-list=WAN
```

[Lista de codigos ICMPv6 Options: ](https://www.iana.org/assignments/icmpv6-parameters/icmpv6-parameters.xhtml)

Monitoramento e logging: Utilizar ferramentas de monitoramento para detectar e responder a atividades suspeitas sem bloquear o ICMPv6 essencial.

Conclusão

O ICMPv6 é fundamental para o funcionamento adequado do IPv6, suportando funções críticas como o Neighbor Discovery e o Path MTU Discovery. Bloquear o ICMPv6 pode causar problemas significativos de rede, desde falhas de comunicação até redução de desempenho. Em vez de bloquear, recomenda-se implementar medidas de segurança que permitam o tráfego ICMPv6 legítimo, mantendo a rede segura e eficiente.

Referências:

[Path MTU discovery in practice](https://blog.cloudflare.com/path-mtu-discovery-in-practice)

[Fixing an old hack - why we are bumping the IPv6 MTU](https://blog.cloudflare.com/increasing-ipv6-mtu)

[Funcionalidades Básicas IPV6](https://www.ipv6.br/post/funcionalidades-basicas)

[Livro IPV6](https://www.ipv6.br/pagina/livro-ipv6)