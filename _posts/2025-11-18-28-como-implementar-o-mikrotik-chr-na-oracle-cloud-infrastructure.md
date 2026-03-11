---
title: "28 | Como Implementar o Mikrotik CHR (Cloud Hosted Router) na Oracle Cloud Infrastructure"
date: 2025-11-18 00:00:00 -0300
categories: [Tutoriais]
tags: [Mikrotik, CHR, OracleCloud, Cloud, Router]
image:
  path: assets/img/posts/post-28/capa.png
---

Este tutorial guia você no processo de criar uma instância do CHR (Cloud Hosted Router) na Oracle Cloud Infrastructure (OCI) grátis para sempre, começando pela criação da conta até a configuração da licença. Siga cada passo cuidadosamente para garantir que tudo seja feito corretamente. Testei em Novembro de 2025.

Importante, para conseguir subir a imagem VMDK, é necessário uma conta nova, por conta do Trial de 30 dias.

Passo 1: Criar a Conta na Oracle Cloud Infrastructure
- Acesse o site da Oracle Cloud.
- Clique em “Começar Grátis” para criar uma conta na Oracle Cloud.
- Preencha seus dados e informações de pagamento. Lembre-se de que a Oracle oferece um período de 30 dias com crédito gratuito, o que te permitirá realizar a configuração sem custos durante esse período.
- Complete o processo de verificação para ativar sua conta e, em seguida, acesse o painel de controle da Oracle Cloud.

Passo 2: Baixar o CHR VMDK
- Acesse o site oficial do MikroTik.
- Encontre a seção de downloads para o Cloud Hosted Router (CHR).
- Baixe a imagem no formato VMDK, que é compatível com a Oracle Cloud.

Nota: O formato VMDK é necessário para importar a imagem no armazenamento da Oracle Cloud.

![](assets/img/posts/post-28/01.png)


Passo 3: Criar um Bucket e Fazer Upload do VMDK
- No painel da Oracle Cloud, vá para a seção Storage > Buckets.
- Clique em “Create Bucket” (Criar Bucket) e defina um nome único para ele.
- Após criar o bucket, acesse-o e clique em Objects > Upload Objects para carregar o arquivo VMDK que você baixou.
- Aguarde até que o upload seja concluído.

![](assets/img/posts/post-28/02.png)
![](assets/img/posts/post-28/03.png)

Passo 4: Importar a Imagem
Agora que o VMDK está armazenado no bucket, você pode importá-lo para a Oracle Cloud e usá-lo como uma imagem para criar instâncias.

- No painel da Oracle Cloud, vá para Compute > Custom Image > Import Image.
- Escolha a opção Operating System > Generic Linux.
- Selecione Import from Object Storage bucket e localize o arquivo VMDK que você carregou.
- Em Image type, selecione vmdk.
- No campo Launch mode, escolha paravirtualized. Este modo otimiza o desempenho da imagem na nuvem.
- Clique em “Next” e aguarde a importação. O status da operação mudará de “Importing” para “Available”. Isso pode levar alguns minutos.

Importante: Com a conta nova, você pode fazer isso gratuitamente durante os primeiros 30 dias, e a imagem continuará disponível mesmo após esse período, sem custos adicionais.

![](assets/img/posts/post-28/04.png)

Passo 5: Criar a VCN (Virtual Cloud Network)
A VCN é essencial para conectar sua instância à internet e a outros recursos da nuvem. Siga os passos abaixo para criar a sua:

- Vá até Networking > VCN e clique em “Create VCN”.
- Defina um nome para a VCN.
- Em IPv4 CIDR Blocks, adicione um bloco CIDR, por exemplo, 10.0.0.0/24.
- Habilite a opção Assign an Oracle-allocated IPv6 /56 prefix para ativar o suporte a IPv6.

![](assets/img/posts/post-28/05.png)
![](assets/img/posts/post-28/06.png)

Crie uma Subnet:
- Defina o mesmo bloco CIDR IPv4, por exemplo, 10.0.0.0/24.
- Defina uma subnet v6, por exemplo, 1a (faça o ajuste conforme o seu bloco CIDR v6).

![](assets/img/posts/post-28/07.png)
![](assets/img/posts/post-28/08.png)

Crie um Internet Gateway para permitir o acesso à internet. em Gateways

![](assets/img/posts/post-28/09.png)
![](assets/img/posts/post-28/10.png)

Modifique o Security List (Lista de Segurança) Defaut já existente:
- Na aba Security Rules > Ingress Rules, exclua as regras existentes e crie novas regras que permitam tráfego de entrada tanto em IPv4 quanto em IPv6.
- Exemplo de regras:
IPv4: Permitir tudo (0.0.0.0/0).
IPv6: Permitir tudo (::/0)

![](assets/img/posts/post-28/11.png)

Associe o Internet Gateway à Route Rules da VCN:
- Vá até Routing > Default Route Table existente.
- Route Rules e adicione uma rota padrão tanto para IPv4 quanto para IPv6 apontando para o Internet Gateway.

![](assets/img/posts/post-28/12.png)

Passo 6: Criar a Instância
Agora você está pronto para criar a instância que usará a imagem do CHR.

- Vá para Compute > Instances > Create Instance.
- Defina um nome para sua instância.
- Em Image, escolha a opção My Images e selecione a imagem do CHR que você importou.
- Em Shape, escolha Virtual Machine > Speciality > VM Always Free-eligible (para usar os recursos gratuitos da Oracle).
- Em Networking, selecione a Virtual Cloud Network existente e a Subnet que você criou anteriormente.
- Em Private IPv4 Address, selecione Automatically assign private IPv4 address.
- Habilite IPv6 Address e deixe a opção Automatically assign IPv6 addresses from prefix ativada.
- Clique em Next, Next e depois em Create para finalizar a criação da instância. (Caso informe sobre a chave SSH, pode ignorar)

Passo 7: Aguardar o Provisionamento
Após criar a instância, aguarde até que o processo de provisionamento seja concluído. Isso pode levar alguns minutos.

- Assim que o provisionamento estiver completo, o painel mostrará o IPv4 Público da instância.
- Com o IP público, você poderá acessar sua instância via Winbox ou WEB.

![](assets/img/posts/post-28/13.png)

Passo 8: Configurar o IPv6
- Não esqueça de habilitar o IPv6 na CHR
```
/ipv6 dhcp-client add add-default-route=yes interface=ether1 request=address
```
![](assets/img/posts/post-28/14.png)

Passo 9: Adicionar Sua Licença
Por fim, adicione sua licença para o MikroTik CHR:

- No painel de administração do MikroTik, acesse a seção de licenciamento.
- Insira a chave de licença do seu CHR para ativar as funcionalidades completas do produto.

Com esses passos, sua instância do MikroTik CHR estará pronta para uso na Oracle Cloud. Se você estiver utilizando a conta gratuita da Oracle, lembre-se de que você tem 30 dias para aproveitar os créditos gratuitos, mas a imagem ficará disponível após esse período, sem custos adicionais.

Caso não tenha dominio, você pode utilizar o DDNS gratuito do DuckDNS e usar esse script na CHR para sempre alterar o IP via API caso necessário.