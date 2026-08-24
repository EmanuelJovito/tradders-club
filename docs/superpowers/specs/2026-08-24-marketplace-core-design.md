# Tradders Club — Marketplace Core (Fase 1)

Data: 2026-08-24

## Contexto

Tradders Club é um marketplace C2C de skins de CS, onde vendedores anunciam
skins e a troca acontece via API da Steam. O comprador paga na plataforma; o
saldo fica retido até a confirmação da troca, quando é liberado para o
vendedor sacar.

O projeto tem quatro subsistemas relativamente independentes:

1. **Marketplace core** (este spec) — catálogo, anúncios, checkout simulado
2. **Escrow/pagamento** — saldo trancado, liberação, saque
3. **Integração Steam (trade offers)** — envio/confirmação da troca real
4. **Conciliação/disputas** — falhas de troca, contestação, reembolso

Cada subsistema terá seu próprio spec e plano de implementação. Este
documento cobre apenas o **Marketplace Core**.

## Objetivo desta fase

Ter um marketplace funcional de ponta a ponta — login, anúncio de skins reais
do inventário Steam do vendedor, navegação/busca no catálogo, e um checkout
**simulado** (sem pagamento real, sem escrow) — que sirva de base para as
fases seguintes.

## Fora de escopo nesta fase

- Pagamento real e saldo trancado (escrow) — fase 2
- Envio/confirmação de troca via Steam Trade Offers — fase 3
- Disputas e reembolso — fase 4
- Reputação/avaliação entre usuários
- Notificações (e-mail, push)
- Valor exato de float do item (a API pública de inventário da Steam só
  fornece a faixa de desgaste, ex.: "Minimal Wear"; o float exato exige
  falar com o Game Coordinator do CS2, algo bem mais complexo — fica para
  uma melhoria futura)

## Arquitetura

- **Next.js (App Router), fullstack**: UI + Route Handlers como backend.
  Um único serviço nesta fase — deliberadamente, para não somar a
  complexidade de multi-serviço à curva de aprendizado de backend do zero.
  A extração de um backend dedicado é esperada na fase de escrow/integração
  Steam, quando surgir necessidade real de processos de longa duração e
  recebimento de webhooks.
- **Postgres + Prisma** como banco de dados e ORM.
- **Docker**: Dockerfile para a aplicação, docker-compose para rodar Postgres
  localmente em desenvolvimento.
- **Deploy no Railway**, com CI/CD via GitHub Actions (lint + testes a cada
  push; deploy automático a partir da branch principal).

## Autenticação

Login via **Steam OpenID**. Ao autenticar, cria ou atualiza o registro do
usuário local (`User`) com os dados básicos do perfil Steam.

## Modelo de dados

- **User** — id, steamId (único), displayName, avatarUrl, criadoEm
- **InventoryItemCache** — cache local dos itens lidos do inventário Steam
  do usuário: assetId, ownerId, marketHashName, iconUrl, faixaDeDesgaste,
  atualizadoEm. Evita bater na API da Steam a cada acesso.
- **Listing** (anúncio) — id, sellerId, itemRef (referência ao item do
  inventário anunciado), preco, descricao, status
  (`ativo` \| `reservado` \| `vendido` \| `cancelado`), criadoEm
- **Order** (pedido) — id, listingId, buyerId, sellerId, preco (congelado no
  momento da compra), status (`aguardando_pagamento` → `pago_simulado` →
  `concluido`, com `cancelado` como saída possível), criadoEm, atualizadoEm

## Fluxos principais

1. **Login** — usuário autentica via Steam OpenID → cria/atualiza `User`.
2. **Anunciar** — usuário autenticado tem seu inventário Steam lido via API
   → escolhe um item que ainda não tenha `Listing` ativo → define preço e
   descrição → cria `Listing`.
3. **Navegar/buscar** — catálogo público com filtros (nome da arma, faixa de
   preço, faixa de desgaste) e paginação.
4. **Ver anúncio** — página de detalhe com dados do item, preço e vendedor.
5. **Comprar (simulado)** — comprador clica em comprar → cria `Order` em
   `aguardando_pagamento`, `Listing` muda para `reservado` → passo simulado
   de "confirmar pagamento" avança o pedido para `pago_simulado` e depois
   `concluido`, e o `Listing` muda para `vendido`.

### Regras de negócio

- Um usuário não pode comprar o próprio anúncio.
- Um item do inventário só pode ter um `Listing` ativo por vez.
- Cancelar um `Order` devolve o `Listing` para `ativo`.

## Testes e verificação

- Testes de unidade para as transições de estado de `Order` e regras de
  negócio acima.
- Testes de integração para os Route Handlers principais (criar anúncio,
  criar pedido, listar catálogo).
- Pipeline de CI executa lint e testes antes de permitir deploy.

## Infra e deploy

- Dockerfile multi-stage para build da aplicação Next.js.
- docker-compose com serviço Postgres para desenvolvimento local.
- GitHub Actions: workflow de CI (lint + testes) em cada push/PR; deploy
  automático para o Railway a partir da branch principal.

## Estudo (regra de trabalho)

A partir da fase de implementação, sempre que uma tarefa introduzir um
conceito técnico novo e relevante (ex.: fluxo OAuth/OpenID, ORM/migrations,
Docker, pipeline de CI/CD, máquina de estados), o trabalho é pausado, um
roteiro de estudo é apresentado, e a implementação só continua depois que o
usuário explicar corretamente os pontos levantados.
