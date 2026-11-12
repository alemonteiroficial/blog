---
layout: post
title: "Amazon VPC: redes, sub-redes e conectividade na AWS"
description: "Um guia técnico para a SAA-C03 sobre CIDR, subnets públicas e privadas, tabelas de rotas, Internet Gateway, NAT Gateway, controles de tráfego e conectividade privada."
date: 2026-11-12
lang: pt
category: AWS · Networking
permalink: /pt/aws-vpc-fundamentos-saa-c03/
---

## 1. O que é uma VPC?

A **Amazon Virtual Private Cloud (VPC)** é uma rede virtual isolada logicamente dentro da AWS. Ela permite definir um espaço de endereçamento IP, criar sub-redes, configurar rotas e controlar como os recursos se comunicam entre si, com a internet e com outras redes.

Para a prova **AWS Certified Solutions Architect – Associate (SAA-C03)**, não basta decorar definições. É importante entender qual componente resolve cada requisito de conectividade e quais condições precisam existir para que o tráfego percorra o caminho esperado.

> **Modelo mental:** a VPC define o ambiente de rede; a subnet delimita uma faixa de IP em uma Availability Zone; a route table determina para onde o tráfego segue; security groups e network ACLs controlam o que pode passar.

## 2. Escopo: região, Availability Zones e subnets

Uma VPC pertence a **uma região AWS** e pode abranger várias Availability Zones (AZs). Uma subnet, por outro lado, pertence a **uma única AZ**.

Isso é importante para arquiteturas resilientes: uma VPC pode conter subnets em diferentes AZs, permitindo distribuir instâncias e outros recursos para reduzir o impacto de uma falha localizada.

Cada subnet recebe um bloco CIDR que deve estar contido no CIDR da VPC e não pode se sobrepor a outras subnets da mesma VPC. Planeje o endereçamento considerando crescimento, conectividade com redes on-premises e possíveis conexões com outras VPCs.

![Diagrama de VPC distribuída em duas Availability Zones]({{ '/assets/images/aws-vpc-two-az.svg' | relative_url }})
<small>Figura 1 — Uma VPC regional pode conter subnets em múltiplas AZs. Cada subnet pertence a apenas uma AZ.</small>

### CIDR e endereços reservados

Exemplo: uma VPC com bloco `10.0.0.0/16` pode ser dividida em subnets menores, como `10.0.1.0/24` e `10.0.2.0/24`.

A AWS reserva **cinco endereços IPv4 em cada subnet**. Em uma subnet `10.0.1.0/24`, por exemplo:

- `10.0.1.0`: endereço de rede.
- `10.0.1.1`: reservado para o roteador da VPC.
- `10.0.1.2`: reservado para o DNS da VPC.
- `10.0.1.3`: reservado para uso futuro da AWS.
- `10.0.1.255`: endereço reservado de broadcast (a AWS não oferece broadcast em VPCs).

Portanto, uma subnet IPv4 `/24` possui 256 endereços no bloco, mas **251 endereços utilizáveis para recursos**.

## 3. Subnet pública versus subnet privada

A diferença não é o nome, nem o fato de uma instância ter ou não IP público. A classificação é determinada principalmente pela **tabela de rotas associada à subnet**.

- **Subnet pública:** possui uma rota para o Internet Gateway (IGW), normalmente `0.0.0.0/0 → igw-...`.
- **Subnet privada:** não possui rota direta para o IGW. Pode ter saída para a internet por meio de um NAT Gateway ou permanecer sem acesso à internet.

Uma instância em subnet pública ainda precisa de um endereço IPv4 público ou Elastic IP, além de regras de security group e NACL compatíveis, para estabelecer comunicação IPv4 direta com a internet.

## 4. Internet Gateway (IGW)

O **Internet Gateway** é o componente que permite comunicação entre uma VPC e a internet. Ele é altamente disponível e escalável, mas sua simples existência não torna uma subnet pública.

Para uma instância iniciar comunicação IPv4 com a internet, verifique o conjunto:

1. O IGW está anexado à VPC.
2. A route table da subnet possui uma rota para o IGW, geralmente `0.0.0.0/0`.
3. A instância possui endereço IPv4 público ou Elastic IP.
4. Security group, NACL e firewall do sistema operacional permitem o tráfego necessário.

**Pegadinha de prova:** criar ou anexar um IGW sem adicionar a rota correspondente não fornece conectividade à internet.

## 5. NAT Gateway: saída para a internet sem exposição direta

O **NAT Gateway** permite que recursos em subnets privadas iniciem conexões com destinos externos, sem permitir que esses destinos iniciem conexões não solicitadas de volta para os recursos privados.

No cenário comum de acesso à internet:

- O NAT Gateway é criado em uma **subnet pública**.
- Ele utiliza um Elastic IP para comunicação com a internet.
- A subnet pública do NAT Gateway tem rota para o IGW.
- A route table da subnet privada aponta a rota padrão para o NAT Gateway: `0.0.0.0/0 → nat-...`.

![Fluxo de tráfego de subnet privada por NAT Gateway]({{ '/assets/images/aws-vpc-nat-routing.svg' | relative_url }})
<small>Figura 2 — Instâncias privadas iniciam conexões de saída através do NAT Gateway. O NAT Gateway não é um mecanismo para publicar serviços privados na internet.</small>

### NAT Gateway: detalhes importantes

- O NAT Gateway é um serviço gerenciado; não é necessário administrar uma instância EC2 como acontece em uma solução NAT instance.
- Um NAT Gateway público é criado em uma subnet pública e usa Elastic IP.
- Para alta disponibilidade e evitar dependência de uma única AZ, uma prática comum é criar um NAT Gateway por AZ e direcionar cada subnet privada ao NAT Gateway da mesma AZ.
- O NAT Gateway não aceita conexões iniciadas espontaneamente da internet para os recursos privados.
- NAT Gateway tem cobrança por hora e por volume de dados processado; considere também custos de transferência de dados.

**NAT Gateway não é Internet Gateway:** o IGW conecta a VPC à internet; o NAT Gateway permite que recursos privados iniciem conexões de saída usando um endereço público compartilhado.

## 6. Route tables e decisão de roteamento

Toda subnet deve estar associada a uma route table. Se nenhuma associação explícita for feita, ela utiliza a **main route table** da VPC.

Uma tabela de rotas contém destinos (prefixos CIDR ou prefix lists) e targets, como:

| Destino | Target | Finalidade |
|---|---|---|
| `10.0.0.0/16` | local | Comunicação dentro da VPC |
| `0.0.0.0/0` | Internet Gateway | Rota padrão para IPv4 na internet |
| `0.0.0.0/0` | NAT Gateway | Saída IPv4 de subnet privada |
| Prefix list do S3 | Gateway Endpoint | Acesso privado ao S3 |

A AWS utiliza **longest prefix match**: entre rotas compatíveis, a rota com o prefixo mais específico prevalece. Por isso, uma rota para `10.0.0.0/16` pode ser escolhida em vez da rota padrão `0.0.0.0/0`.

A rota `local` permite comunicação entre recursos que possuem conectividade de rede dentro dos CIDRs da VPC. Ela não significa que security groups e NACLs deixam de ser aplicados.

## 7. Security Groups versus Network ACLs

Essa comparação é recorrente na SAA-C03.

| Característica | Security Group (SG) | Network ACL (NACL) |
|---|---|---|
| Escopo | Interface de rede/recursos associados | Subnet |
| Estado | Stateful | Stateless |
| Regras | Allow | Allow e Deny |
| Avaliação | Todas as regras aplicáveis | Em ordem numérica; primeira correspondência |
| Retorno do tráfego | Automaticamente permitido pelo estado da conexão | Deve ser permitido explicitamente |

### Security Groups

Security Groups são **stateful**. Se uma conexão de entrada é permitida, o tráfego de resposta é permitido automaticamente, mesmo que não exista uma regra de saída específica para esse retorno.

Eles suportam regras de permissão, não regras explícitas de negação. É possível referenciar outro security group como origem ou destino em cenários suportados, facilitando políticas como “o banco aceita tráfego apenas do SG da aplicação”.

### Network ACLs

Network ACLs são **stateless** e atuam no nível da subnet. É necessário permitir tanto o tráfego de ida quanto o de retorno. Regras são avaliadas em ordem numérica crescente, e a primeira regra correspondente determina a decisão.

Isso torna as NACLs úteis para controles amplos de subnet, inclusive bloqueios explícitos por faixa IP. Ao analisar uma conexão que falha, lembre-se das portas efêmeras usadas nas respostas TCP.

> **Atalho para a prova:** SG = stateful, associado à interface/recurso e somente allow. NACL = stateless, associado à subnet, allow e deny, com avaliação ordenada.

## 8. VPC Endpoints: acessar serviços AWS sem depender de NAT/IGW

VPC Endpoints permitem conectar a VPC a serviços AWS de forma privada, sem exigir que o tráfego passe pela internet pública.

Os dois tipos mais cobrados:

**Gateway Endpoint**
- Disponível para Amazon S3 e DynamoDB.
- É associado às route tables selecionadas.
- Não utiliza ENI nem endereço IP privado dentro da subnet.
- Não tem cobrança adicional pelo uso do endpoint em si; permanecem os custos normais do serviço e de transferência aplicáveis.

**Interface Endpoint (AWS PrivateLink)**
- Cria uma ou mais Elastic Network Interfaces (ENIs) nas subnets selecionadas.
- Usa endereços IP privados e security groups.
- É utilizado por muitos serviços AWS e serviços publicados por parceiros ou outras contas.
- Possui cobrança por hora e processamento de dados.

**Cenário clássico:** uma aplicação em subnet privada precisa acessar o S3 sem utilizar NAT Gateway. Um Gateway Endpoint para S3 pode atender ao requisito, com a route table apropriada e políticas de endpoint/bucket compatíveis.

## 9. Conectividade entre VPCs e ambientes on-premises

Conheça a finalidade de cada opção:

- **VPC Peering:** conexão privada entre duas VPCs. Não é transitiva; uma VPC não utiliza outra VPC como roteador para alcançar uma terceira por meio do peering.
- **AWS Transit Gateway:** hub central para conectar múltiplas VPCs e redes on-premises, permitindo arquiteturas hub-and-spoke.
- **Site-to-Site VPN:** conexão criptografada pela internet entre a rede on-premises e a AWS.
- **AWS Direct Connect:** conexão de rede dedicada entre o ambiente on-premises e a AWS; não é, por si só, criptografia. VPN pode ser usada em conjunto quando a criptografia é necessária.

A escolha depende de escala, topologia, requisitos de latência, redundância, custo e necessidade de conectividade transitiva.

## 10. VPC Flow Logs e troubleshooting

**VPC Flow Logs** capturam informações sobre o tráfego IP que entra e sai de interfaces de rede, subnets ou VPCs. Podem ser publicados no CloudWatch Logs, Amazon S3 ou Amazon Data Firehose, conforme a configuração suportada.

Use Flow Logs para investigar padrões de tráfego e identificar registros aceitos ou rejeitados. Eles não são uma captura de pacotes: não registram o conteúdo completo da comunicação.

Em um troubleshooting de conectividade, siga uma sequência:

1. Confirme CIDR, IP e subnet da origem e do destino.
2. Verifique a route table associada à subnet.
3. Confirme IGW, NAT Gateway, endpoint, peering ou Transit Gateway, conforme o caminho esperado.
4. Analise as regras do Security Group.
5. Analise NACLs de entrada e saída, incluindo portas efêmeras.
6. Verifique DNS, firewall do sistema operacional e se a aplicação está escutando na porta correta.
7. Utilize VPC Flow Logs como evidência adicional.

## 11. Cenários que você precisa reconhecer na SAA-C03

| Requisito do enunciado | Solução que você deve considerar |
|---|---|
| Instância pública precisa acessar a internet | IGW + rota na subnet + IPv4 público/EIP + regras |
| Instâncias privadas precisam baixar atualizações da internet | NAT Gateway + rota na subnet privada |
| Acessar S3 sem sair para internet pública | S3 Gateway Endpoint |
| Acessar serviço AWS via IP privado/PrivateLink | Interface Endpoint |
| Conectar duas VPCs sem hub central | VPC Peering, se não houver necessidade de trânsito |
| Conectar muitas VPCs em topologia hub-and-spoke | Transit Gateway |
| Conectar escritório/datacenter à AWS por VPN | Site-to-Site VPN |
| Conexão dedicada com a AWS | Direct Connect |
| Bloquear tráfego por faixa IP no nível da subnet | Network ACL |
| Permitir tráfego entre tiers por identidade de grupo | Security Groups |
| Investigar tráfego aceito/rejeitado | VPC Flow Logs |

## 12. Teste seu entendimento

Tente responder antes de abrir a explicação. O objetivo é reconhecer o requisito principal e eliminar alternativas que resolvem outro problema.

<details>
<summary><strong>Questão 1 — Acesso privado ao S3</strong></summary>

Uma aplicação em subnet privada precisa acessar objetos no Amazon S3. A empresa quer evitar NAT Gateway e não deseja que o tráfego utilize a internet pública. Qual opção é mais adequada?

**Resposta:** S3 Gateway Endpoint associado à route table da subnet. Verifique também as políticas do endpoint e do bucket.
</details>

<details>
<summary><strong>Questão 2 — Regras de retorno</strong></summary>

Uma conexão de entrada é permitida por um Security Group. É necessário criar uma regra específica para permitir o tráfego de resposta?

**Resposta:** Não. Security Groups são stateful e permitem automaticamente o tráfego de retorno de conexões permitidas.
</details>

<details>
<summary><strong>Questão 3 — Comunicação com três VPCs</strong></summary>

A arquitetura possui várias VPCs e precisa de conectividade centralizada e transitiva. Qual serviço tende a simplificar essa topologia?

**Resposta:** AWS Transit Gateway. VPC Peering não é transitivo.
</details>

<details>
<summary><strong>Questão 4 — Instância privada sem acesso externo</strong></summary>

Uma EC2 em subnet privada não consegue baixar atualizações. A route table não possui rota padrão para NAT Gateway. Qual ajuste é esperado?

**Resposta:** Criar ou utilizar um NAT Gateway em subnet pública, com rota para o IGW, e adicionar na route table privada uma rota `0.0.0.0/0` apontando para o NAT Gateway.
</details>

## 13. Checklist de revisão

- [ ] VPC é regional; subnet pertence a uma AZ.
- [ ] Uma subnet pública possui rota para um Internet Gateway.
- [ ] Para IPv4 público direto, a instância precisa de endereço público/EIP.
- [ ] NAT Gateway permite conexões de saída iniciadas por recursos privados.
- [ ] Route tables usam longest prefix match.
- [ ] Security Groups são stateful; NACLs são stateless.
- [ ] NACLs avaliam regras em ordem numérica e suportam allow/deny.
- [ ] Gateway Endpoint atende S3 e DynamoDB; Interface Endpoint usa ENIs.
- [ ] VPC Peering não é transitivo; Transit Gateway atende topologias hub-and-spoke.
- [ ] Flow Logs registram metadados de tráfego, não payloads de pacotes.

## Referências oficiais

- [Amazon VPC — User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html)
- [VPC CIDR blocks](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html)
- [Subnet route tables](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)
- [Internet gateways](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)
- [NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
- [Security groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
- [Network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)
- [VPC endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html)
- [VPC Flow Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)

---

*Este material é uma nota de estudo independente, organizada para apoiar a preparação para a AWS SAA-C03. Consulte sempre a documentação oficial para detalhes e atualizações dos serviços.*
