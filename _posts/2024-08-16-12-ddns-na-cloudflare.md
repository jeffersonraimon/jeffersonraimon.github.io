---
title: "12 | DDNS na Cloudflare"
date: 2024-08-16 00:00:00 -0300
categories: [Tutoriais]
tags: [DDNS, Cloudflare, DNS]
image:
  path: assets/img/posts/post-12/capa.jpg
---

Conforme mostrei no post 11, eu tenho um domínio e aponto o dns type AAAA para uma VM rodando o Traefik Proxy. Meu IPv6 GUA é entregue pelo provedor de forma dinâmica.
Como eu tenho esse domínio na Cloudflare, posso usar a API deles para sempre atualizar o IP.

Segue como fiz:

Primeiro passo é criar um token da API na cloudflare:

Para criar um token da API do Cloudflare para sua zona DNS, siga estas etapas:

- Acesse <https://dash.cloudflare.com/profile/api-tokens>.
- Clique em Criar Token.
- Dê um nome ao token, por exemplo, ```cloudflare-ddns```.
- Conceda ao token as seguintes permissões:

- Zona – Configurações da Zona – Leitura

- Zona – Zona – Leitura

- Zona – DNS – Editar

Defina os recursos da zona para:

- Incluir – Todas as zonas

- Complete o assistente e copie o token gerado.

O segundo passo é criar um registro do tipo AAAA com o seu IPv6 ou algum outro IPv6 de teste, pois o container que vamos usar tem dificuldade em criar do zero.

![](assets/img/posts/post-12/01.png)

Use o docker-compose abaixo na VM desejada e altere as informações. Agradecimentos ao oznu
```
version: '2'
services:
  cloudflare-ddns:
    image: oznu/cloudflare-ddns:latest
    restart: always
    network_mode: host
    environment:
      - API_KEY=XXXXXXXXXXXXXXXXXXXXXXXXXXXX
      - ZONE=xxxxxxxx.xxxx
      - PROXIED=false
      - RRTYPE=AAAA
```      
Em ZONE: Seria o seu domínio raiz. Ex: teste.com

Após subir, analise os logs com o comando
```
docker logs NOME_DO_CONTAINER
```
![](assets/img/posts/post-12/02.png)

No RouterOSv7
Caso queira adicionar o IP dinâmico que recebe na sua RB, eu adaptei um script que encontrei. Você pode encontrar meu script aqui (veja o link do script original para entender como conseguir as informações).