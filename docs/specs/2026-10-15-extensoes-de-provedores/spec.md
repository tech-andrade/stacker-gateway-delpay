# Spec — Pontos de extensão para provedores (fase 0) e Delfinance PSP (fase 1)

| | |
|---|---|
| **Status** | Aprovada para planejamento. A implementação ainda não começou. |
| **Data** | 2026-10-15 |
| **Plano da fase 0** | [plano-fase-0.md](plano-fase-0.md) |
| **Plano da fase 1** | Será escrito depois que a fase 0 for revisada |
| **Contexto** | [docs/arquitetura/provedores-de-pagamento.md](../../arquitetura/provedores-de-pagamento.md), [docs/arquitetura/proposta-delfinance.md](../../arquitetura/proposta-delfinance.md) |

## 1. Problema

Hoje só a cobrança (pay-in) de um provedor passa por uma interface (`GatewayDriver`). Todo o resto é `match`/`if` por slug espalhado pelo core:
- PIX OUT (saque);
- estorno;
- validação de credenciais mínimas;
- classificação nos relatórios;
- categoria e badges na tela de admin.

Um provedor completo como o Okto mexeu em 33 arquivos. Isso tem três consequências:

1. Cada provedor novo no fork da DelPay gera cerca de 30 pontos de conflito com o upstream.
2. Não dá para distribuir a Delfinance para outras instalações sem alterar o core delas.
3. O upstream acumula duplicação: um `attemptX`, um job e um comando de reconciliação por provedor.

## 2. Objetivo

- **Fase 0:** criar pontos de extensão genéricos, para que um provedor novo declare suas capacidades e implemente interfaces, **sem novos `match` por slug no core**. Ela será proposta ao upstream (`stacker-builders/stacker-gateway`).
- **Fase 1:** implementar a Delfinance como PSP (PIX IN, PIX OUT e estorno) **usando apenas os pontos de extensão da fase 0**.

## 3. Princípio central: a fase 0 é aditiva

Esta é a regra que torna a proposta aceitável para o dono do upstream:

1. **Nenhum provedor existente é alterado.** Drivers, services, jobs, comandos e webhooks de CajuPay, Efí, Woovi, BSPay, Versell, Xflow, Okto e dos demais ficam intocados.
2. **Nenhum braço existente de `match`/`if` muda de comportamento.** O caminho novo entra só no braço `default` ou como um `|| …` em condições que hoje retornam "não suportado".
3. **Precedência:** o braço com o nome do provedor é avaliado antes da interface. Para um provedor que tem os dois, vale o braço legado.
4. **Prova de não regressão:** a suíte de testes atual passa **sem nenhuma alteração**. Os testes novos usam provedores falsos em `tests/Fixtures/Gateways`.
5. **As interfaces copiam as assinaturas que já existem** sempre que possível (ex.: `refundTransaction`). Assim, migrar um provedor depois é só acrescentar `implements` e apagar o braço dele. Essa migração fica **fora do escopo**: é decisão do mantenedor do upstream.

## 4. Fase 0 — Requisitos

A fase 0 é entregue em 4 PRs independentes, na ordem abaixo. Cada um é revisável e aprovável sozinho, com uma exceção: o PR 4 reaproveita os fixtures de teste e o helper de controller criados pelos PRs 1 e 3.

### PR 1 — Correção: aprovação de saque pelo admin com a Versell

- **R1.1** Quando a Versell é o provedor de payout ativo, aprovar o saque em `/plataforma/saques` envia o PIX pela Versell, como já acontece no saque automático. Hoje o resultado é "Gateway de payout não suportado."
- **R1.2** O saque aprovado pelo admin não é marcado como automático (`payout_meta.auto` ausente).
- **R1.3** As mensagens de sucesso e erro seguem o padrão dos outros provedores.

### PR 2 — Categoria e capacidades declaradas

- **R2.1** A definição do provedor aceita `category` (string) opcional. Os valores conhecidos são `psp`, `acquirer` e `open_finance`.
- **R2.2** `config/gateways.php` aceita `categories` opcional (padrão `[]`): categorias extras com `title` e `dot_class`.
- **R2.3** A definição aceita `capabilities` (lista) opcional. Os valores válidos são `pix_in`, `pix_out` e `refund`; os desconhecidos são descartados.
- **R2.4** As props da tela de adquirentes e o JSON de detalhe do provedor incluem `category` e `capabilities`, ambos `null` quando não declarados.
- **R2.5** Um provedor com `category` aparece no grupo correspondente. Os provedores sem `category` continuam exatamente como hoje (listas de slug, e o resto em PSP).
- **R2.6** Um provedor com `capabilities` tem os badges PIX IN/OUT derivados delas. Os que não têm continuam nas listas fixas atuais.

### PR 3 — Estorno, credenciais e relatório por contrato

- **R3.1** Criar a interface `App\Gateways\Contracts\SupportsRefund`, com a mesma assinatura do `refundTransaction` que já existe nos drivers.
- **R3.2** Um provedor sem braço próprio em `OrderRefundGatewayBridge` que implementa `SupportsRefund` tem o estorno executado. O `pending` vira `gateway_pending`, e a metadata fica como `{slug}_refund_id` / `{slug}_refund_pending`. MED e falha são mapeados pelo `mapDriverRefundResult` atual.
- **R3.3** Criar a interface `App\Gateways\Contracts\ValidatesCredentials`. O braço `default` de `GatewayApiCredentials::isReadyForGateway` passa a consultar o driver quando ele implementa essa interface. Senão, continua `true`.
- **R3.4** `Order::paymentMethodReportKey()`: antes de retornar `'outro'`, um provedor cujos `methods` são exatamente `['pix']` é classificado como `pix`, e exatamente `['card']`, como `card`.

### PR 4 — PIX OUT por contrato

- **R4.1** Criar a interface `App\Gateways\Contracts\SupportsPayout`, com `sendPixPayout(credentials, amountCents, destination, externalId)` e `getPayoutStatus(transactionId, credentials)`.
- **R4.2** Um provedor habilitado (em `platform_acquirer_slugs`) cujo driver implementa `SupportsPayout` e que não está em `PAYOUT_ORDER` entra **depois** dos provedores legados na ordem automática de payout. Ele também pode ser escolhido como preferência.
- **R4.3** O saque automático por contrato segue o fluxo do Okto:
  - calcula taxa e mínimo com `GatewayPayoutEconomics` (`{slug}_payout_min_brl`, `{slug}_admin_fee_payout_*`);
  - resolve o destino pela solicitação de saque ou pelo cadastro do vendedor (`payout_pix_key`);
  - grava `payout_provider`, `payout_external_id` e `payout_meta`;
  - agenda a reconciliação.
- **R4.4** A reconciliação é genérica: um único job e um único comando (`withdrawals:reconcile-contract-payouts`, a cada minuto) para todos os provedores por contrato.
- **R4.5** A aprovação manual pelo admin funciona para provedores por contrato (sem `auto`).
- **R4.6** Em todos os pontos de destino PIX (validador, cadastro de chave pelo vendedor, solicitação de saque, `payout_pix_setup`), o provedor por contrato usa o modo **`pix_key_only`**. O modo "chave + titular" (CajuPay/Versell) fica fora desta fase.
- **R4.7** A tela de admin lista os provedores por contrato no seletor de preferência de payout.
- **R4.8** A falha de envio (erro do driver ou exceção) é registrada em `payout_meta.last_error` e não derruba a requisição.

### Fora do escopo da fase 0

- Migrar os provedores existentes para as interfaces novas.
- Interface para MED, Pix Automático, saldo no provedor ou tokenização de cartão no checkout.
- Modo "chave + documento do titular" para payout por contrato.
- Categoria "Provedores KYC". Ela será tratada na fase 2, com um registro separado (veja a proposta).

## 5. Fase 1 — Delfinance como PSP (visão geral)

O plano detalhado será escrito depois da revisão da fase 0. Estes são os requisitos de alto nível.

- **R5.1** `app/Gateways/Delfinance/DelfinanceDriver.php` implementa `GatewayDriver` (PIX), `SupportsPayout`, `SupportsRefund` e `ValidatesCredentials`.
- **R5.2** A entrada `delfinance` em `config/gateways.php` tem:
  - `category => 'psp'`;
  - `capabilities => ['pix_in', 'pix_out', 'refund']`;
  - `credential_keys`, incluindo `delfinance_payout_min_brl` e `delfinance_admin_fee_*`.

  O slug entra em `platform_acquirer_slugs` e em `default_order.pix`.
- **R5.3** Webhook próprio (`DelfinanceWebhookController`) com validação de autenticidade:
  - cobrança paga: `PaymentWebhookDispatcher::dispatch('delfinance', …, 'order.paid', 'paid')`;
  - transferência: `MerchantWithdrawalService::markPaid/markFailed`;
  - estorno: `PlatformOrderAdminService::applyGatewayRefund`.
- **R5.4** `getTransactionStatus` e `getPayoutStatus` confiáveis. São a base da reconfirmação do webhook e das reconciliações.
- **R5.5** Edições no core limitadas a: `config/gateways.php`, `routes/web.php` (rota do webhook) e o logo em `public/images/gateways/`. Opcionalmente, `GatewayWebhookUrl` e `config/webhooks.php`. **Nenhum `match`/`if` novo por slug.**
- **R5.6** Testes unitários do driver (HTTP falso) e testes feature do webhook e do saque.
- **Dependências em aberto:** a documentação da API da Delfinance (autenticação, PIX, transferência, estorno, webhooks e assinatura) e a decisão entre driver no core (proposto ao upstream) ou plugin.

## 6. Estratégia de git e PRs

Segue os fluxos do `CLAUDE.md`:

1. Cada PR da fase 0 nasce em `contrib/<nome>`, a partir de `upstream/main` (fluxo 3), e é proposto para `stacker-builders/stacker-gateway:main`.
2. Enquanto o upstream não aceita, os commits entram no fork por `git cherry-pick -x`, numa branch `integration/<nome>` (fluxo 2).
3. A fase 1 é desenvolvida no fork (`feature/delfinance-psp`, fluxo 1), em cima da fase 0 já trazida.
4. Se o upstream aceitar com mudanças, o próximo sync traz a versão dele. Nos conflitos, prevalece a do upstream.
5. Se o upstream recusar, a fase 0 fica no fork, com cada ponto marcado com `[delpay]`.

## 7. Critérios de aceite (fase 0)

- A suíte completa (`composer test`) passa sem alterar nenhum teste existente.
- Os testes novos cobrem cada requisito R1 a R4. Os de frontend (R2.5, R2.6, R4.7) são validados por `npm run build` e por uma conferência manual descrita no plano.
- Com o fixture de provedor falso registrado, um provedor com `category`, `capabilities`, `SupportsRefund`, `ValidatesCredentials` e `SupportsPayout` funciona de ponta a ponta: aparece no grupo certo, estorna, saca, reconcilia e é aprovado pelo admin, sem nenhum `match` com o nome dele.
- Os `diff`s dos PRs não mostram nenhuma linha removida nos braços dos provedores existentes. A única exceção é o parâmetro opcional `bool $auto = true` em `attemptVersell` (PR 1).

## 8. Riscos

| Risco | Mitigação |
|---|---|
| O upstream rejeita "código sem consumidor" | Testes com provedor falso; o PR cita a Delfinance como consumidor; PRs pequenos e aditivos |
| O upstream muda os mesmos arquivos em paralelo | PRs pequenos e rápidos; cada PR rebaseado em `upstream/main` antes de abrir |
| `GatewayRegistry::driver()` instanciado em pontos novos (custo, dependências no construtor) | `PlatformPayoutGateway` usa `is_subclass_of` sobre a classe, sem instanciar; só os caminhos de execução instanciam |
| Provedor por contrato precisa do modo "chave + titular" | Fora da fase 0; vira requisito da fase 1 se a API da Delfinance exigir |
