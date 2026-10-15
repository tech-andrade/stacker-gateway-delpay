# Proposta — Delfinance como PSP, adquirência e Provedores KYC

> **Status: proposta para discussão.** Nada aqui foi implementado. O documento se baseia no que está descrito em [provedores-de-pagamento.md](provedores-de-pagamento.md) e [kyc.md](kyc.md).

## Objetivos

1. Colocar a **Delfinance como PSP** (PIX IN e PIX OUT, estorno e, de preferência, MED).
2. Quando houver adquirência, **habilitar cartão** pela Delfinance.
3. Criar uma nova categoria, **Provedores KYC**, e plugar nela a solução de basic KYC da Delfinance.
4. Usar tudo isso na DelPay **e** deixar pronto para que outras instalações do Stacker Gateway possam usar a Delfinance.

## O problema a resolver primeiro

Hoje, um provedor completo exige mudanças em **cerca de 30 arquivos centrais do upstream** (o Okto mexeu em 33), porque PIX OUT, estorno, MED, badges e categoria são `match`/`if` por slug. Isso dá dois problemas:

- **Para o fork:** cada um desses arquivos vira um ponto de conflito em todo merge do upstream.
- **Para a comunidade:** ninguém consegue instalar a Delfinance sem receber todas essas alterações no core.

Por isso a recomendação é começar pelos **pontos de extensão genéricos** (fase 0). Eles são úteis para qualquer provedor, então são bons candidatos a PR para o upstream (fluxo 3 do `CLAUDE.md`).

## Fase 0 — Pontos de extensão genéricos (candidatos a contribuição upstream)

Todas as mudanças abaixo são **aditivas**: os provedores atuais continuam nos seus `match`, e só o braço `default` passa a consultar uma interface.

> A spec formal e o plano de implementação tarefa por tarefa estão em [docs/specs/2026-10-15-extensoes-de-provedores/](../specs/2026-10-15-extensoes-de-provedores/spec.md). A ordem definida lá é: PR 1 (bug da Versell) → PR 2 (categoria e capacidades) → PR 3 (estorno, credenciais e relatório) → PR 4 (PIX OUT).

| # | Mudança | Onde | Efeito |
|---|---|---|---|
| 0.1 | Chave `category` na definição do provedor (`'psp'`, `'acquirer'`, `'open_finance'`…) e o frontend agrupando por ela, com a constante atual como fallback | `config/gateways.php`, `ProvidesPlatformGatewayProps`, `Platform/Financial/Index.vue` | A categoria passa a ser declarada pelo próprio provedor, inclusive por plugin. Categorias novas ganham título e cor por config |
| 0.2 | Capacidades declaradas na definição (ex.: `'capabilities' => ['pix_out', 'refund', 'balance']`) e `PixInOutBadges.vue` lendo da prop | `config/gateways.php`, `ProvidesPlatformGatewayProps`, `PixInOutBadges.vue` | Os badges deixam de ser listas fixas |
| 0.3 | Interface `Contracts\SupportsPayout` (`sendPixPayout(Withdrawal, User): array`, `getPayoutStatus(string, array): ?string`), com o braço `default` de `WithdrawalAutoPayoutService` e de `approveWithdrawal` usando `instanceof` | `app/Gateways/Contracts`, `WithdrawalAutoPayoutService`, `Platform\FinancialController`, `PlatformPayoutGateway` (lista derivada das capacidades) | PIX OUT sem `match` novo |
| 0.4 | Comando genérico `withdrawals:reconcile-generic`, que reconcilia saques de qualquer provedor que implemente `SupportsPayout` | `app/Console/Commands`, `bootstrap/app.php` | Evita um comando e um job por provedor |
| 0.5 | Interface `Contracts\SupportsRefund` (formaliza o `refundTransaction()` que já existe) no braço `default` de `OrderRefundGatewayBridge` | `OrderRefundGatewayBridge` | Estorno sem `match` novo |
| 0.6 | `isReadyForGateway` delegando para um método opcional do driver no braço `default` | `GatewayApiCredentials` | Valida as credenciais mínimas sem `match` novo |

Também vale corrigir, via upstream, o **bug da Versell** em `approveWithdrawal`: falta o bloco `versell`.

**Se o upstream não aceitar ou demorar:** dá para aplicar a fase 0 no fork, marcada com `[delpay]`. Ainda assim são cerca de 6 arquivos com mudanças pequenas, contra cerca de 30 por provedor.

## Fase 1 — Delfinance como PSP (PIX IN + PIX OUT)

Com a fase 0 pronta, o provedor fica quase todo em arquivos novos:

```
app/Gateways/Delfinance/DelfinanceDriver.php        implements GatewayDriver, SupportsPayout, SupportsRefund
app/Services/Delfinance/DelfinanceHttpClient.php
app/Http/Controllers/Webhooks/DelfinanceWebhookController.php
config/gateways.php                                  entrada 'delfinance' (category 'psp', capabilities, credential_keys)
                                                     + platform_acquirer_slugs + default_order.pix
routes/web.php                                       rota do webhook
public/images/gateways/delfinance.svg
tests/Unit/Delfinance/…, tests/Feature/DelfinanceWebhookTest.php
```

Ainda é preciso tocar no core em `Order::paymentMethodReportKey` e `config/webhooks.php`. O MED continua exigindo os `elseif` dos dois controllers de MED, a não ser que entre uma interface `SupportsMed` na fase 0.

**Como distribuir para a comunidade.** Há três opções, que não se excluem:

| Opção | Prós | Contras |
|---|---|---|
| **A. Contribuir o driver Delfinance para o upstream**, como foi feito com Okto e Xflow | Aparece para todas as instalações e é mantido junto com o core | Depende de o mantenedor aceitar |
| **B. Plugin `plugins/delfinance`** | Zero conflito com o upstream; pode ser distribuído pela loja de plugins | Só funciona por completo se a fase 0 existir no upstream |
| **C. Só no fork** | Rápido | Ninguém de fora usa, e cada merge do upstream fica mais caro |

**Recomendação:** desenvolver no fork já no formato da opção A (driver no core, seguindo os padrões do upstream) e propor ao upstream depois da fase 0.

## Fase 2 — Categoria "Provedores KYC"

KYC **não é provedor de pagamento**. Se a Delfinance KYC entrasse em `config/gateways.php`, apareceria na ordem de cobrança e nas listas de payout. Por isso a proposta é usar um **registro separado**, que segue o mesmo padrão dos gateways:

```
config/kyc_providers.php                       definição: slug, name, image, driver, credential_keys, checks (cpf, documento, selfie/liveness, cnpj…)
app/Kyc/Contracts/KycProviderDriver.php        interface (Strategy)
app/Kyc/KycProviderRegistry.php                config + register() para plugins, igual ao GatewayRegistry
app/Kyc/Delfinance/DelfinanceKycDriver.php
app/Models/KycProviderCredential.php           credenciais criptografadas (ou gateway_credentials com o slug "kyc:<slug>" — decidir)
app/Models/KycVerification.php                 resultado de cada verificação: user, provider, status, score, motivos, external_id, raw
```

Esboço da interface:

```php
interface KycProviderDriver
{
    public function testConnection(array $credentials): bool;

    /** Envia os dados e documentos do vendedor para verificação. */
    public function verify(User $subject, array $documents, array $credentials): KycVerificationResult;

    /** Para provedores assíncronos: consulta o resultado depois. */
    public function getStatus(string $externalId, array $credentials): KycVerificationResult;
}

// KycVerificationResult: status (approved|rejected|pending|manual_review), score, reasons[], externalId, raw
```

**Onde o fluxo atual se encaixa:**

1. Extrair a aprovação e a rejeição de `KycVerificationsController` para um `KycReviewService`. A tela de admin passa a usar o service, e a automação também.
2. Criar o evento **`KycSubmitted`**, que hoje não existe, e dispará-lo em `SellerKycController::markPendingReview()`.
3. Criar o listener `RunKycProviderOnSubmission`, em fila: chama o provedor ativo e grava um `KycVerification`.
4. Agir de acordo com um **modo** configurável:
   - **`advisory`** (recomendado para começar): só mostra o resultado e o score para o admin, que continua decidindo;
   - **`auto_approve`**: aprova sozinho quando o provedor aprova e manda para o admin os demais casos;
   - **`auto`**: aprova e rejeita sozinho.
5. Webhook do provedor KYC, para resultados assíncronos: `POST /webhooks/kyc/{slug}`.

**Tela.** Para mostrar os Provedores KYC como uma categoria nova, há duas opções:
- **(a)** um grupo novo na tela `/plataforma/financeiro`, reaproveitando o `GatewayCard`, mas alimentado pelo `KycProviderRegistry`;
- **(b)** uma seção em Configurações → KYC, junto com as exigências de documentos que já ficam ali.

Nos dois casos, o registro continua separado do de pagamento.

**Para a comunidade:** como tudo isso é infraestrutura nova e genérica, a fase 2 inteira (sem o driver Delfinance) é candidata a PR para o upstream. O driver Delfinance KYC pode ir junto ou ser um plugin.

## Fase 3 — Adquirência (cartão)

Quando a Delfinance tiver adquirência:

- `'card'` em `methods` e `createCardPayment` no driver;
- **tokenização no navegador**: um branch novo em `CheckoutForm.vue`, com um composable em `resources/js/composables`, e chaves públicas via `checkout_payload_keys`. Isso também precisa ser replicado em `Renewal/Show.vue` e `ApiCheckout/Show.vue`;
- `CardInstallments` (parcelamento) e estorno de cartão;
- trocar ou adicionar a categoria (`'acquirer'`). Com a fase 0.1, basta uma linha na definição.

Uma melhoria candidata ao upstream: mover a tokenização de cartão para o `registry.js` por slug. Isso acabaria com a armadilha do fallback para a Efí.

## Decisões em aberto

1. Propor a fase 0 ao upstream antes, ou implementar direto no fork e propor depois?
2. Delfinance PSP: driver no core (opção A) ou plugin (opção B)?
3. KYC: onde ficam as credenciais (tabela nova ou `gateway_credentials` com prefixo) e onde fica a tela (Financeiro ou Configurações → KYC)?
4. KYC: com qual modo começar (`advisory`, `auto_approve` ou `auto`)?
5. Quais verificações o basic KYC da Delfinance faz (CPF na Receita, OCR, liveness, CNPJ…)? Isso define a lista de `checks` e quais documentos o fluxo atual ainda precisa pedir.
