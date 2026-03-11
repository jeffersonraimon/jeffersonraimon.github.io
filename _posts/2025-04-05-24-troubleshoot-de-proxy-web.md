---
title: "24 | Troubleshoot de Proxy Web"
date: 2025-04-05 00:00:00 -0300
categories: [Artigos]
tags: [Proxy, Networking, Troubleshooting]
image:
  path: assets/img/posts/post-24/capa.png
---

No meu homelab de uns tempos para cá, eu estava enfrentando um problema chato em que de forma aleatória ao acessar meus serviços internos via proxy, no navegador apresentava o erro 502 bad gateway, e ao apertar F5 para recarregar a paǵina, o serviço abria tranquilamente como se nada tivesse acontecido.

Com um tempo livre resolvi investigar esse B.O.

Nesse post foi resumido bastante os testes feitos para ja direcionar aos passos que resultou a solução.

Primeira coisa que fiz foi tentar interceptar o que acontecia na comunicação Navegador x Proxy. Utilizei o burp para isso e com meus conhecimentos atuais de programação web, não identifiquei nada diferente que possa estar acontecendo e sendo a causa.

![](assets/img/posts/post-24/01.png)

Depois disso pensei um pouco, no NPM (nginx proxy manager) eu configurei todos os serviços usando custom domains que eu criei para minha rede interna, pois tenho um servidor DNS local usando Ad Guard Home. 

Ficando assim a topologia:

![](assets/img/posts/post-24/capa.png)

Docker compose do NPM (resumido):
```
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    dns:
      - 10.1.10.55
      - fd10:1:10::55
```
Como estão na mesma rede local, descartei a possibilidade de ser alguma coisa no meu Router (que ta cheio de regras de firewall) causando esse delay.

Então decidi utilizar como teste apenas o IP de um serviço para ver se teria o mesmo comportamento. e pasme, não aconteceu mais o problema. Ou seja, alguma coisa ocorria no processo de consulta DNS (timeout, baixo tempo de ttl, delays na resolução de nomes etc).

![](assets/img/posts/post-24/03.png)

Fiz varios testes ajustando timeouts no NPM, setando 300s de TTL nas configurações de DNS do Adguard mas não houve efeito.

```
proxy_connect_timeout 60s;
proxy_send_timeout 60s;
proxy_read_timeout 60s;
send_timeout 60s;
```
Agora era validar quem estava dormindo no ponto, o NPM ou o ADGUARD.

Acessei o shell dentro do container e realizei testes de conexão diretamente para avaliar como estava os tempos de resposta. (no resultado abaixo alterei meu domínio para teste.exemplo.lan).

```
root@NPM:~# docker exec -it npm-app-1 bash

[root@docker-08fafbb7df35:/app]# time curl -I http://10.1.10.101:3001
HTTP/1.1 200 OK
Cache-Control: no-store
Content-Type: text/html; charset=UTF-8
X-Content-Type-Options: nosniff
X-Xss-Protection: 1; mode=block
Date: Sat, 05 Apr 2025 14:27:55 GMT

real    0m0.011s
user    0m0.004s
sys     0m0.002s
[root@docker-08fafbb7df35:/app]# time curl -I http://10.1.10.101:3001
HTTP/1.1 200 OK
Cache-Control: no-store
Content-Type: text/html; charset=UTF-8
X-Content-Type-Options: nosniff
X-Xss-Protection: 1; mode=block
Date: Sat, 05 Apr 2025 14:27:56 GMT

real    0m0.011s
user    0m0.003s
sys     0m0.003s
[root@docker-08fafbb7df35:/app]# time curl -I http://teste.exemplo.lan:3001
HTTP/1.1 200 OK
Cache-Control: no-store
Content-Type: text/html; charset=UTF-8
X-Content-Type-Options: nosniff
X-Xss-Protection: 1; mode=block
Date: Sat, 05 Apr 2025 14:28:00 GMT

real    0m0.011s
user    0m0.003s
sys     0m0.004s
[root@docker-08fafbb7df35:/app]# time curl -I http://teste.exemplo.lan:3001
HTTP/1.1 200 OK
Cache-Control: no-store
Content-Type: text/html; charset=UTF-8
X-Content-Type-Options: nosniff
X-Xss-Protection: 1; mode=block
Date: Sat, 05 Apr 2025 14:28:04 GMT

real    0m4.013s
user    0m0.002s
sys     0m0.005s
```

Como podemos ver, de 2 tentativas 1 passou de 4s na conexão usando domínio. Realmente confirmando a suspeita.

Para aprofundar os testes usei o comando dig dentro do container e foi o que demostrou definitivamente a causa desse B.O.

```
[root@docker-08fafbb7df35:/app]# apt install dnsutils
[root@docker-08fafbb7df35:/app]# dig teste.exemplo.lan

; <<>> DiG 9.18.33-1~deb12u2-Debian <<>> teste.exemplo.lan
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 64476
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;teste.exemplo.lan.      IN      A

;; ANSWER SECTION:
teste.exemplo.lan. 15    IN      A       10.1.10.101

;; Query time: 1 msec
;; SERVER: 127.0.0.11#53(127.0.0.11) (UDP)
;; WHEN: Sat Apr 05 14:29:48 UTC 2025
;; MSG SIZE  rcvd: 58
```
Fora do container:

```
root@NPM:~# dig teste.exemplo.lan

; <<>> DiG 9.18.30-0ubuntu0.24.04.2-Ubuntu <<>> teste.exemplo.lan
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 28100
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;teste.exemplo.lan.      IN      A

;; ANSWER SECTION:
teste.exemplo.lan. 15    IN      A       10.1.10.101

;; Query time: 0 msec
;; SERVER: 10.1.10.55#53(10.1.10.55) (UDP)
;; WHEN: Sat Apr 05 11:30:18 -03 2025
;; MSG SIZE  rcvd: 58
```

Bingo! mesmo configurado no docker-compose para usar meu DNS interno, o container ainda assim utilizava o DNS padrão da rede docker, e estava causando delay.

Que plot twist, hein?!

Para solucionar o problema definitivamente, eu alterei o modo da rede do NPM para host e com isso a rede interna do docker para de atuar (porém, lembrando que mapeamentos de portas para dentro do container não funcionam mais, vai ser exposta as portas configuradas da aplicação).
```
network_mode: "host"
```

Antes:
```
root@NPM:~/npm# netstat -tulnap | grep 443
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      10088/docker-proxy  
tcp6       0      0 :::443                  :::*                    LISTEN      10096/docker-proxy
```
Depois:
```
root@NPM:~# netstat -tulnap | grep 443
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      4918/nginx: master  
tcp6       0      0 :::443                  :::*                    LISTEN      4918/nginx: master  
```