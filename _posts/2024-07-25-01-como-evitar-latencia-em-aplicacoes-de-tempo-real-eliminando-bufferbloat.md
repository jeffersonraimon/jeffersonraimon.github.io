---
title: "01 | Como evitar latência em aplicações de tempo real (Eliminando Bufferbloat)"
date: 2024-07-25 00:00:00 -0300
categories: [Artigos]
tags: [Bufferbloat, Latency, Networking]
image:
  path: assets/img/posts/post-01/capa.png
---

# Postado originalmente no LinkedIn em 30/10/2022

Você sabe o que é **Bufferbloat** e como isso pode estar afetando a sua gameplay online ou seu streaming de vídeo sem você saber? Está passando por essas dificuldades? Espero te ajudar com esses conhecimentos!

Eu poderia simplesmente te mostrar o que deve ser feito para contornar essa problemática. Porém, acho que entender todo o processo, abrirá a sua mente.

Então vamos lá!

É comum pensarmos que a causa do **lag** nos jogos ou **streaming** com baixa qualidade/travando é por conta de:

- rotas dinâmicas da internet,
- da sua provedora de internet (ISP),
- do próprio servidor do jogo/stream que pode estar congestionado por conta da alta demanda,
- ou pela localização geográfica do servidor. (Até porque a latência do Brasil para EUA é por volta de 120ms)

Algumas configurações que a maioria das pessoas fazem para tentar evitar isso é abrir portas **UDP/TCP** no roteador ou modificar o **MTU** (Max Transmission Unit).

Porém, e se eu te disser que existe mais uma variável nessa história (pouco conhecida) e que na maioria das vezes, pode ser o maior problema de todos? É o tal do **Bufferbloat**.

### Trazendo uma definição:

**Bufferbloat** é a alta latência em redes com comutação de pacotes causada pelo excesso de **buffering**¹ de pacotes. Principalmente na sua rede interna. (A rede da sua casa, ou seja, seu roteador, computador, celular que estão conectados por cabo ou Wi-Fi).

O **bufferbloat** também pode causar variação de atraso de pacotes (também conhecido como **jitter**). Quando um roteador ou switch está configurado para usar buffers excessivamente grandes, mesmo as redes de alta velocidade podem tornar-se praticamente inutilizáveis para muitas aplicações interativas, como **VoIP**, **jogos on-line** e até navegação na web.

¹ **Buffer**: Basicamente seria uma memória temporária em que quando os pacotes chegam no roteador, esses pacotes ficam guardados nessa memória, esperando os outros pacotes que chegaram primeiro, serem encaminhados.

Alguns fabricantes de equipamentos de comunicação colocaram buffers excessivamente grandes em alguns de seus produtos de rede. Em tal equipamento, o **bufferbloat** ocorre quando um link de rede fica congestionado, fazendo com que os pacotes fiquem enfileirados com buffer durante muito tempo. Os buffers excessivamente grandes resultam em filas mais longas, com maior latência, e não melhoram o rendimento da rede.

Ou seja, não adianta querer uma baixa latência em jogos ou streaming sem interrupções que estão em outras redes (Internet) se a sua rede interna (intranet) está congestionada. **Ah, só um spoiler: ter uma alta velocidade de banda larga contratada (Ex: 300Mbps) não vai garantir sanar sua dificuldade. ESSA É A PREMISSA.**

Agora que entendemos o problema e causa dele, vamos contornar isso!

### A nossa solução contra o **bufferbloat** é o **QoS (Quality of Service)**, que usa filas de pacotes com algoritmos como **CoDel**, **FQ_CoDel** ou **Cake**.

#### Entendendo melhor:

**QoS (Quality of Service)** é o uso de mecanismos ou tecnologias que privilegiam o tráfego de dados em determinados aparelhos da sua rede e os serviços que estão sendo utilizados. (Definidas pelo usuário). A ideia do **QoS** é que nem todo tipo de tráfego é igual em importância, sendo possível determinar quais dispositivos e serviços terão maior prioridade de conexão. **Streaming**, **jogos multiplayer** são exemplos de atividades que precisam de melhor conexão para funcionar bem, o que não é totalmente necessário para atualizações de sistema, troca de mensagens via **WhatsApp** e navegar na Internet, por exemplo.

A tecnologia de rede **QoS** funciona marcando pacotes para identificar tipos de serviço e, em seguida, configurando roteadores para criar filas virtuais separadas para cada aplicativo, com base em sua prioridade. Dessa forma, a largura de banda é reservada para aplicativos essenciais que receberam acesso prioritário.

Ou seja, quando os pacotes da comunicação entre o seu **PC** e o servidor do jogo chegar ao roteador, o **QoS** criará uma fila prioritária (como se fosse uma fila de banco para idosos, gestantes etc.), tendo assim uma maior velocidade ao encaminhar esses pacotes e evitará perder tempo no buffer do roteador. Garantindo uma baixa latência.

**Um detalhe importante**: Nada é de graça, você terá que abrir mão da largura de banda total que tem, utilizando menos banda no seu computador para criar uma fila mais efetiva. Normalmente indicam se utilizar 40% a 70% da sua banda total.

Veja a imagem como exemplo:

![](assets/img/posts/post-01/01.png)

Na imagem acima, a área em verde é a fila criada por algoritmo do **QoS**, e os pacotes que estão nessa área, são do jogo. Veja que essa área é menor, por isso menos banda. Essas filas fazem com que os primeiros pacotes que chegam no roteador, sejam os primeiros a saírem. (Aqui entra o conceito de **FIFO** – **First In, First Out**, que fica como lição de casa caso tenha curiosidade em saber mais).

Existem diversos algoritmos **QoS** que gerenciam essas filas, os mais indicados para jogos são **CoDel** (Controlled Delay), **FQ_CoDel** (Fair Queuing Controlled Delay) e **Cake** (Common Applications Kept Enhanced). Não irei entrar em detalhes quais as vantagens ou como cada um funciona, fica como outra lição de casa.

Agora sim vamos fazer alguns testes e aplicar o **QoS**.

### Será utilizado um **Notebook Acer Aspire 5**, com **Windows 11 22H2**, na rede **Wi-Fi 5GHz**.

#### Teste no site **Speedtest**:

![](assets/img/posts/post-01/02.png)

As informações destacadas em vermelho são a variação de latência em **Download** e **Upload**, respectivamente.

#### Fazendo outro teste no site **Waveform Bufferbloat**:

![](assets/img/posts/post-01/03.png)

Esse site dará mais detalhes sobre a latência e também uma nota dependendo do resultado.

Perceba que o teste recebeu uma nota **B** e aviso de que poderá ocorrer **bufferbloat** em **Gaming**.

### Agora vamos à solução.

Irei utilizar o roteador **Mikrotik RB750GR3** na versão **RouterOSv7.5** com o algoritmo de **FQ_CoDel**. (Esse e outros algoritmos citados anteriormente foram adicionados na versão 7.1)

#### Algoritmos **QoS** no **RouterOS**:

![](assets/img/posts/post-01/04.png)

Vídeo mostrando o processo de configuração no **RouterOS**:

Aplicando o **QoS** do tipo **FQ_CoDel** no **IP** atribuído ao meu notebook e definindo a banda total para **50Mb de Download** e **20Mb de Upload** (poderia ser mais).

![](assets/img/posts/post-01/05.png)

No meu caso, só preciso definir qual será o **IP** da máquina prioritária, não precisando definir o tipo de serviço (alguns fabricantes de roteadores têm essa opção).

### Vamos novamente aos testes:

#### **Speedtest**:

![](assets/img/posts/post-01/06.png)

#### **Waveform Bufferbloat**:

![](assets/img/posts/post-01/07.png)

Monitorando o tráfego do teste:

![](assets/img/posts/post-01/08.png)

Veja só a diferença depois do **QoS** funcionando. Sem **latência** e **bufferbloat**!

Agora sim, ao jogar utilizando essa máquina, todos os pacotes do jogo terão prioridade na minha rede interna, reduzindo drasticamente a latência alta. Caso eu não esteja mais jogando, só desabilitar o **QoS** e poderei ter toda a banda contratada para o notebook novamente.

Apliquem **QoS** e façam seus testes. Caso não tenha roteador Mikrotik, existem diversos outros que têm essa funcionalidade, como **EdgeRouter X**, **NetGear**, etc.

**Autor**: Jefferson J. Raimon

---

### Referências e leitura complementar:

- [Bufferbloat - Wikipedia](https://en.wikipedia.org/wiki/Bufferbloat)
- [Bufferbloat: What Can I Do About It?](https://www.bufferbloat.net/projects/bloat/wiki/What_can_I_do_about_Bufferbloat)
- [Fortinet - QoS (Quality of Service)](https://www.fortinet.com/br/resources/cyberglossary/qos-quality-of-service)
- [GeeksforGeeks - CoDel Queue Discipline](https://www.geeksforgeeks.org/controlled-delay-codel-queue-discipline)
- [pfSense - CoDel Limiters](https://docs.netgate.com/pfsense/en/latest/recipes/codel-limiters.html)
- [pfSense - Altq Scheduler Types](https://docs.netgate.com/pfsense/en/latest/trafficshaper/altq-scheduler-types.html#altq-codel)
- [Clube do Hardware - Mikrotik FQ_CoDel ou Cake Queue](https://www.clubedohardware.com.br/forums/topic/1581897-mikrotik-fq_codel-ou-cake-queue)
- [Mikrotik - Queues Documentation](https://help.mikrotik.com/docs/display/ROS/Queues)