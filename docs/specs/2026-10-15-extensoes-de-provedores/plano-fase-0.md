# Pontos de extensão para provedores (fase 0) — Plano de implementação

> **Para agentes:** SUB-SKILL OBRIGATÓRIA: use superpowers:subagent-driven-development (recomendado) ou superpowers:executing-plans para implementar este plano tarefa por tarefa. Os passos usam checkbox (`- [ ]`) para acompanhamento.

**Objetivo:** permitir que um provedor novo declare categoria e capacidades e implemente estorno, validação de credenciais e PIX OUT por **interfaces**, sem nenhum `match`/`if` novo com o nome dele e sem alterar os provedores existentes.

**Arquitetura:** cada capacidade ganha uma interface em `app/Gateways/Contracts`. Os pontos do core que hoje fazem `match ($slug)` ganham, **só no braço `default`** (ou como `|| …` numa condição que hoje cai em "não suportado"), uma consulta `instanceof <Interface>`. Os metadados (`category`, `capabilities`) são chaves opcionais da definição do provedor, lidas pelo backend e repassadas ao frontend com fallback para as listas fixas atuais.

**Stack:** PHP 8.3, Laravel 12, PHPUnit 11 (SQLite em memória), Inertia + Vue 3, Vite.

**Spec:** [spec.md](spec.md). Leia antes de começar.

## Restrições globais

- **Não alterar provedores existentes:** nada em `app/Gateways/<Provedor>/`, `app/Services/<Provedor>/`, `app/Jobs/Reconcile<Provedor>*`, `app/Console/Commands/Reconcile<Provedor>*` nem `app/Http/Controllers/Webhooks/*`.
- **Não alterar o comportamento dos braços existentes** de `match`/`if`. A única exceção autorizada é o parâmetro opcional `bool $auto = true` em `WithdrawalAutoPayoutService::attemptVersell` (tarefa 1).
- **Não alterar testes existentes.** A suíte inteira tem que passar como está.
- Os valores válidos de `capabilities` são exatamente `pix_in`, `pix_out` e `refund`.
- Os IDs de categoria já conhecidos pelo frontend são `psp`, `acquirer` e `open_finance`.
- O modo de destino PIX dos provedores por contrato é sempre `pix_key_only`.
- Nome do comando novo: `withdrawals:reconcile-contract-payouts`.
- Código, nomes e mensagens de commit em inglês, no estilo do upstream (uma frase explicando o *porquê*). Mensagens exibidas ao usuário em português, como no resto do sistema.
- Cada PR nasce de `upstream/main` numa branch `contrib/<nome>`; veja o fluxo 3 no `CLAUDE.md`. Este plano **não** faz push nem abre PR: isso só acontece com a confirmação do usuário.

## Como rodar os testes

```bash
php artisan test --filter=NomeDoTeste            # PHP local
# ou dentro do Docker de dev:
docker compose -f docker-compose.local-db.yml -f docker-compose.dev.yml exec app php artisan test --filter=NomeDoTeste
composer test                                    # suíte completa
npm run build                                    # valida o frontend
```

## Foco de revisão

Estas são as condições que a spec implica e que mais provavelmente quebrariam em uso real. Cada uma tem um teste na tarefa indicada.

1. **Provedores existentes inalterados.** A suíte completa passa sem mudanças no fim de cada PR (último passo das tarefas 1, 3, 6 e 12).
2. **Provedor com `SupportsPayout` mas fora de `platform_acquirer_slugs`** não pode virar provedor de payout (tarefa 7, `test_contract_gateway_outside_platform_acquirer_slugs_is_ignored`).
3. **Credencial do provedor por contrato desconectada** (`is_connected = false`): ele não fica ativo, e o saque não é enviado (tarefa 7, `test_disconnected_contract_gateway_is_not_active`).
4. **Saque sem chave PIX** (nem na solicitação, nem no cadastro): o resultado é `skipped/no_pix_key`, sem chamar o driver (tarefa 8, `test_missing_pix_key_skips_without_calling_driver`).
5. **Driver lança exceção** no envio do saque ou no estorno: o erro é registrado, e a requisição não quebra (tarefa 8, `test_driver_exception_is_recorded_as_error`; tarefa 4, `test_refund_exception_is_mapped_to_failed`).

## Estrutura de arquivos

| Arquivo | PR | Responsabilidade |
|---|---|---|
| `app/Services/WithdrawalAutoPayoutService.php` (modificar) | 1, 4 | `attemptVersell($w, $auto)`, `attemptViaContract()` e o braço `default` |
| `app/Http/Controllers/Platform/FinancialController.php` (modificar) | 1, 4 | Bloco Versell, bloco por contrato e helper de redirect em `approveWithdrawal`; validação e props de preferência |
| `tests/Feature/VersellAdminApproveWithdrawalTest.php` (criar) | 1 | Teste da correção da Versell |
| `app/Support/GatewayCapabilities.php` (criar) | 2 | Normaliza `capabilities` da definição |
| `app/Support/GatewayCategories.php` (criar) | 2 | Normaliza `config('gateways.categories')` |
| `config/gateways.php` (modificar) | 2 | Chave `categories` vazia, documentada |
| `app/Http/Controllers/Concerns/ProvidesPlatformGatewayProps.php` (modificar) | 2 | Expõe `category` e `capabilities` na lista |
| `app/Http/Controllers/GatewaysController.php` (modificar) | 2 | Expõe `category` e `capabilities` no `show` |
| `resources/js/Pages/Platform/Financial/Index.vue` (modificar) | 2, 4 | Agrupamento por categoria; opções extras de payout |
| `resources/js/components/settings/PixInOutBadges.vue` (modificar) | 2 | Badges a partir das `capabilities` |
| `resources/js/components/settings/GatewayCard.vue`, `GatewayConfigSidebar.vue` (modificar) | 2 | Repassam `capabilities` |
| `tests/Concerns/RegistersFakeGateways.php` (criar) | 2 | Helper de teste para registrar um provedor falso |
| `tests/Unit/Gateways/GatewayMetadataTest.php` (criar) | 2 | Testes de metadados |
| `app/Gateways/Contracts/SupportsRefund.php` (criar) | 3 | Interface de estorno |
| `app/Gateways/Contracts/ValidatesCredentials.php` (criar) | 3 | Interface de credenciais mínimas |
| `app/Services/OrderRefundGatewayBridge.php` (modificar) | 3 | `default` → `tryContractRefund()` |
| `app/Support/GatewayApiCredentials.php` (modificar) | 3 | `default` → consulta o driver |
| `app/Models/Order.php` (modificar) | 3 | Fallback por `methods` no relatório |
| `tests/Fixtures/Gateways/FakeGatewayDriver.php` (criar) | 3 | Driver falso (cobrança, estorno, credenciais) |
| `tests/Feature/ContractRefundTest.php`, `tests/Unit/Gateways/ContractCredentialsAndReportTest.php` (criar) | 3 | Testes |
| `app/Gateways/Contracts/SupportsPayout.php` (criar) | 4 | Interface de PIX OUT |
| `app/Services/Payout/PlatformPayoutGateway.php` (modificar) | 4 | `contractSlugs()`, `supportsPayoutContract()`, `payoutOrder()`, `contractOptions()` |
| `app/Services/Payout/ContractPayoutService.php` (criar) | 4 | Economia, destino e envio por contrato |
| `app/Jobs/ReconcileContractPayoutWithdrawalJob.php` (criar) | 4 | Reconciliação de um saque |
| `app/Console/Commands/ReconcileContractPayoutWithdrawalsCommand.php` (criar) | 4 | Reconciliação em lote |
| `bootstrap/app.php` (modificar) | 4 | Agendamento do comando |
| `app/Services/Payout/PayoutDestinationValidator.php`, `app/Http/Controllers/SellerFinancialController.php` (modificar) | 4 | `pix_key_only` para provedores por contrato |
| `tests/Fixtures/Gateways/FakePayoutGatewayDriver.php` (criar) | 4 | Driver falso com payout |
| `tests/Unit/Gateways/PlatformPayoutGatewayContractTest.php`, `tests/Feature/ContractPayoutFlowTest.php`, `tests/Feature/ContractPayoutAdminTest.php`, `tests/Feature/ContractPayoutDestinationTest.php` (criar) | 4 | Testes |
| `app/Gateways/README.md` (modificar) | 3, 4 | Documenta as interfaces novas |

---

## PR 1 — `contrib/fix-versell-admin-payout`

### Tarefa 1: Aprovação de saque pelo admin com a Versell

**Arquivos:**
- Modificar: `app/Services/WithdrawalAutoPayoutService.php` (método `attemptVersell`)
- Modificar: `app/Http/Controllers/Platform/FinancialController.php` (`approveWithdrawal` e helpers novos no fim da classe)
- Teste: `tests/Feature/VersellAdminApproveWithdrawalTest.php`

**Interfaces:**
- Consome: `WithdrawalAutoPayoutService::attemptVersell(Withdrawal $withdrawal): array{ok: bool, skipped?: bool, reason?: string, error?: string, pending?: bool}` (já existe).
- Produz:
  - `WithdrawalAutoPayoutService::attemptVersell(Withdrawal $withdrawal, bool $auto = true): array` (mesmo retorno);
  - `FinancialController::redirectAfterAdminPayoutAttempt(Request $request, Withdrawal $withdrawal, string $slug, string $label, array $result): RedirectResponse` (private), reutilizado na tarefa 10;
  - `FinancialController::adminPayoutSkipMessage(?string $reason): string` (private).

- [ ] **Passo 0: Criar a branch a partir do upstream**

```bash
git fetch upstream
git switch -c contrib/fix-versell-admin-payout upstream/main
```

- [ ] **Passo 1: Escrever o teste que falha**

`tests/Feature/VersellAdminApproveWithdrawalTest.php`:

```php
<?php

namespace Tests\Feature;

use App\Http\Middleware\EnsureInstalled;
use App\Http\Middleware\EnsureStackerLicense;
use App\Jobs\ReconcileVersellWithdrawalJob;
use App\Models\GatewayCredential;
use App\Models\Setting;
use App\Models\User;
use App\Models\Withdrawal;
use App\Services\Withdrawal\WithdrawalPolicyService;
use Illuminate\Foundation\Http\Middleware\ValidateCsrfToken;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Queue;
use Illuminate\Support\Facades\Schema;
use Tests\TestCase;

class VersellAdminApproveWithdrawalTest extends TestCase
{
    private string $tmpDir;

    protected function setUp(): void
    {
        parent::setUp();
        $this->withoutMiddleware([EnsureInstalled::class, EnsureStackerLicense::class, ValidateCsrfToken::class]);

        $this->tmpDir = sys_get_temp_dir().DIRECTORY_SEPARATOR.'versell_admin_'.uniqid('', true);
        mkdir($this->tmpDir, 0700, true);
        foreach (['cash_in.crt', 'cash_in.key', 'cash_out.crt', 'cash_out.key'] as $file) {
            file_put_contents($this->tmpDir.DIRECTORY_SEPARATOR.$file, "-----BEGIN X-----\nT\n-----END X-----\n");
        }

        Cache::flush();
        Queue::fake();
    }

    protected function tearDown(): void
    {
        GatewayCredential::query()->where('gateway_slug', 'versell')->delete();
        Setting::set('platform_payout_gateway', null, null);
        foreach (glob($this->tmpDir.DIRECTORY_SEPARATOR.'*') ?: [] as $file) {
            @unlink($file);
        }
        @rmdir($this->tmpDir);
        parent::tearDown();
    }

    public function test_admin_approval_sends_payout_through_versell(): void
    {
        if (! Schema::hasTable('withdrawals')) {
            $this->markTestSkipped('withdrawals table');
        }

        $cred = GatewayCredential::query()->firstOrNew(['tenant_id' => null, 'gateway_slug' => 'versell']);
        $cred->is_connected = true;
        $cred->setEncryptedCredentials([
            'cash_in' => [
                'client_id' => 'ci_client',
                'client_secret' => 'ci_secret',
                'certificate_path' => $this->tmpDir.DIRECTORY_SEPARATOR.'cash_in.crt',
                'private_key_path' => $this->tmpDir.DIRECTORY_SEPARATOR.'cash_in.key',
                'pix_key' => 'pix@versell.test',
            ],
            'cash_out' => [
                'client_id' => 'co_client',
                'client_secret' => 'co_secret',
                'certificate_path' => $this->tmpDir.DIRECTORY_SEPARATOR.'cash_out.crt',
                'private_key_path' => $this->tmpDir.DIRECTORY_SEPARATOR.'cash_out.key',
            ],
        ]);
        $cred->save();
        Setting::set('platform_payout_gateway', 'versell', null);

        Http::fake([
            'https://pagamentos.basspago.com.br/api/v2/oauth/token' => Http::response(['access_token' => 'co-token', 'expires_in' => 3600], 200),
            'https://pagamentos.basspago.com.br/api/v2/pix/payments/dict' => Http::response([
                'endToEndId' => 'EADMIN12345678901234567890123456',
                'status' => 'ON_QUEUE',
            ], 202),
        ]);

        WithdrawalPolicyService::setManualApprovalPin('1234');
        $admin = User::factory()->create(['role' => User::ROLE_PLATFORM_ADMIN, 'tenant_id' => null]);

        $seller = User::factory()->create(['role' => User::ROLE_INFOPRODUTOR]);
        $seller->forceFill([
            'tenant_id' => $seller->id,
            'payout_settings' => [
                'cajupay_pix_key' => 'destino@pix.com',
                'cajupay_pix_key_type' => 'email',
                'cajupay_pix_key_owner_document' => '52998224725',
            ],
        ])->save();

        $withdrawal = Withdrawal::query()->create([
            'tenant_id' => $seller->id,
            'user_id' => $seller->id,
            'amount' => 100,
            'fee_amount' => 0,
            'net_amount' => 100,
            'bucket' => 'pix',
            'status' => 'pending',
            'currency' => 'BRL',
        ]);

        $response = $this->actingAs($admin)->post(
            route('plataforma.financeiro.saques.approve', $withdrawal),
            ['manual_approval_pin' => '1234']
        );

        $response->assertRedirect(route('plataforma.saques.index'));
        $response->assertSessionHas('success');

        $fresh = $withdrawal->fresh();
        $this->assertSame('versell', $fresh->payout_provider);
        $this->assertSame('EADMIN12345678901234567890123456', $fresh->payout_external_id);
        $this->assertArrayNotHasKey('auto', is_array($fresh->payout_meta) ? $fresh->payout_meta : []);
        Queue::assertPushed(ReconcileVersellWithdrawalJob::class);
    }
}
```

- [ ] **Passo 2: Rodar e confirmar que falha**

Rode `php artisan test --filter=VersellAdminApproveWithdrawalTest`.
Esperado: FAIL. A sessão tem `error` "Gateway de payout não suportado." e `payout_provider` é `null`.

- [ ] **Passo 3: Adicionar o parâmetro `$auto` em `attemptVersell`**

Em `app/Services/WithdrawalAutoPayoutService.php`, altere a assinatura:

```php
    public function attemptVersell(Withdrawal $withdrawal, bool $auto = true): array
```

Dentro do **mesmo método**, troque as duas ocorrências de `'auto' => true,` por `'auto' => $auto,`. Uma está no bloco de sucesso (dentro do `array_filter`, que descarta `false`, então o saque do admin fica sem a chave) e a outra no bloco de falha. Não mexa em nenhum outro método.

- [ ] **Passo 4: Adicionar o bloco Versell e o helper em `approveWithdrawal`**

Em `app/Http/Controllers/Platform/FinancialController.php`, imediatamente **antes** de `if ($slug === 'onlyup') {` dentro de `approveWithdrawal`, insira:

```php
        if ($slug === 'versell') {
            $result = app(WithdrawalAutoPayoutService::class)->attemptVersell($withdrawal->fresh(), auto: false);

            return $this->redirectAfterAdminPayoutAttempt($request, $withdrawal, 'versell', 'Versell', $result);
        }
```

No fim da classe, antes do último `}`, adicione:

```php
    /**
     * @param  array{ok: bool, skipped?: bool, reason?: string, error?: string, pending?: bool}  $result
     */
    private function redirectAfterAdminPayoutAttempt(Request $request, Withdrawal $withdrawal, string $slug, string $label, array $result): RedirectResponse
    {
        MerchantWithdrawalService::releasePayoutApproval($withdrawal->fresh());

        if (! ($result['ok'] ?? false)) {
            if ($result['skipped'] ?? false) {
                return redirect()->route('plataforma.saques.index')
                    ->with('error', $this->adminPayoutSkipMessage($result['reason'] ?? null));
            }

            $message = $label.': '.($result['error'] ?? 'Falha ao enviar o saque.');
            $this->notifyWithdrawalPayoutError($withdrawal, $message);

            return redirect()->route('plataforma.saques.index')->with('error', $message);
        }

        PlatformAuditService::log('platform.withdrawal.approved', [
            'withdrawal_id' => $withdrawal->id,
            $slug => true,
            'pending' => true,
        ], $request);

        return redirect()->route('plataforma.saques.index')
            ->with('success', 'Saque enviado à '.$label.'. Será marcado como pago após confirmação do PIX.');
    }

    private function adminPayoutSkipMessage(?string $reason): string
    {
        return match ($reason) {
            'no_pix_key' => 'O infoprodutor precisa cadastrar uma chave PIX para saque em Financeiro (painel do vendedor).',
            'no_key_owner_document' => 'O infoprodutor precisa informar o CPF/CNPJ do titular da chave PIX em Financeiro.',
            'no_owner' => 'Titular da conta (infoprodutor) não encontrado.',
            default => 'Não foi possível enviar o saque pelo gateway de payout ativo.',
        };
    }
```

Garanta o import `use App\Services\WithdrawalAutoPayoutService;` no topo do arquivo. `Request`, `RedirectResponse`, `Withdrawal`, `MerchantWithdrawalService` e `PlatformAuditService` já são usados no arquivo.

- [ ] **Passo 5: Rodar o teste e confirmar que passa**

Rode `php artisan test --filter=VersellAdminApproveWithdrawalTest`.
Esperado: PASS.

- [ ] **Passo 6: Rodar a suíte completa**

Rode `composer test`.
Esperado: tudo verde. Nenhum teste existente foi alterado.

- [ ] **Passo 7: Commit**

```bash
git add app/Services/WithdrawalAutoPayoutService.php app/Http/Controllers/Platform/FinancialController.php tests/Feature/VersellAdminApproveWithdrawalTest.php
git commit -m "Let platform admins approve withdrawals when Versell is the active payout gateway instead of failing as unsupported."
```

---

## PR 2 — `contrib/gateway-category-capabilities`

### Tarefa 2: Metadados `category` e `capabilities` no backend

**Arquivos:**
- Criar: `app/Support/GatewayCapabilities.php`
- Criar: `app/Support/GatewayCategories.php`
- Modificar: `config/gateways.php` (chave nova no nível superior)
- Modificar: `app/Http/Controllers/Concerns/ProvidesPlatformGatewayProps.php` (`buildGatewaysListForSettings`)
- Modificar: `app/Http/Controllers/GatewaysController.php` (`show`, array `$payload`)
- Modificar: `app/Http/Controllers/Platform/FinancialController.php` (`index`, prop nova)
- Criar: `tests/Concerns/RegistersFakeGateways.php`
- Teste: `tests/Unit/Gateways/GatewayMetadataTest.php`

**Interfaces:**
- Produz:
  - `GatewayCapabilities::PIX_IN|PIX_OUT|REFUND` (constantes string);
  - `GatewayCapabilities::fromDefinition(array $definition): ?array` (lista ou null);
  - `GatewayCapabilities::has(array $definition, string $capability): bool`;
  - `GatewayCategories::extra(): list<array{id: string, title: string, dot_class: string}>`;
  - nas props de cada gateway: `category: ?string` e `capabilities: ?list<string>`;
  - prop de página `gateway_categories`;
  - trait de teste `RegistersFakeGateways` com `registerFakeGateway(string $slug, ?string $driverClass = null, array $overrides = []): void` e `connectGlobalGatewayCredential(string $slug, array $credentials = ['api_key' => 'test'], bool $connected = true): GatewayCredential`.

- [ ] **Passo 0: Criar a branch**

```bash
git fetch upstream
git switch -c contrib/gateway-category-capabilities upstream/main
```

- [ ] **Passo 1: Criar o helper de teste**

`tests/Concerns/RegistersFakeGateways.php`:

```php
<?php

namespace Tests\Concerns;

use App\Models\GatewayCredential;

trait RegistersFakeGateways
{
    /**
     * Registra um provedor só para o teste (config é reiniciada a cada teste).
     *
     * @param  array<string, mixed>  $overrides
     */
    protected function registerFakeGateway(string $slug, ?string $driverClass = null, array $overrides = []): void
    {
        config()->set("gateways.gateways.{$slug}", array_merge([
            'slug' => $slug,
            'name' => 'Fake '.ucfirst($slug),
            'image' => 'images/gateways/fake.png',
            'methods' => ['pix'],
            'scope' => 'national',
            'driver' => $driverClass,
            'credential_keys' => [
                ['key' => 'api_key', 'label' => 'API key', 'type' => 'password'],
            ],
        ], $overrides));

        $slugs = config('gateways.platform_acquirer_slugs', []);
        if (! in_array($slug, $slugs, true)) {
            $slugs[] = $slug;
            config()->set('gateways.platform_acquirer_slugs', $slugs);
        }
    }

    /**
     * @param  array<string, mixed>  $credentials
     */
    protected function connectGlobalGatewayCredential(string $slug, array $credentials = ['api_key' => 'test'], bool $connected = true): GatewayCredential
    {
        $cred = GatewayCredential::query()->firstOrNew(['tenant_id' => null, 'gateway_slug' => $slug]);
        $cred->is_connected = $connected;
        $cred->is_enabled = true;
        $cred->setEncryptedCredentials($credentials);
        $cred->save();

        return $cred;
    }
}
```

- [ ] **Passo 2: Escrever os testes que falham**

`tests/Unit/Gateways/GatewayMetadataTest.php`:

```php
<?php

namespace Tests\Unit\Gateways;

use App\Http\Controllers\Concerns\ProvidesPlatformGatewayProps;
use App\Support\GatewayCapabilities;
use App\Support\GatewayCategories;
use Tests\Concerns\RegistersFakeGateways;
use Tests\TestCase;

class GatewayMetadataTest extends TestCase
{
    use RegistersFakeGateways;

    /**
     * @return array<string, array<string, mixed>>
     */
    private function gatewaysBySlug(): array
    {
        $host = new class
        {
            use ProvidesPlatformGatewayProps;

            public function list(): array
            {
                return $this->buildGatewaysListForSettings(null);
            }
        };

        return collect($host->list())->keyBy('slug')->all();
    }

    public function test_capabilities_are_normalized_and_unknown_values_dropped(): void
    {
        $this->assertSame(
            ['pix_in', 'pix_out', 'refund'],
            GatewayCapabilities::fromDefinition(['capabilities' => ['REFUND', 'bogus', ' pix_out ', 'pix_in']])
        );
        $this->assertNull(GatewayCapabilities::fromDefinition([]));
        $this->assertTrue(GatewayCapabilities::has(['capabilities' => ['pix_out']], GatewayCapabilities::PIX_OUT));
        $this->assertFalse(GatewayCapabilities::has([], GatewayCapabilities::PIX_OUT));
    }

    public function test_extra_categories_are_normalized(): void
    {
        config()->set('gateways.categories', [
            'banking' => ['title' => 'Banking as a Service', 'dot_class' => 'bg-amber-500'],
            'no_title' => ['dot_class' => 'bg-red-500'],
            'not_array' => 'x',
        ]);

        $this->assertSame(
            [['id' => 'banking', 'title' => 'Banking as a Service', 'dot_class' => 'bg-amber-500']],
            GatewayCategories::extra()
        );
    }

    public function test_declared_category_and_capabilities_reach_admin_props(): void
    {
        $this->registerFakeGateway('fakepay', null, [
            'category' => 'acquirer',
            'capabilities' => ['pix_in', 'pix_out'],
        ]);

        $row = $this->gatewaysBySlug()['fakepay'];

        $this->assertSame('acquirer', $row['category']);
        $this->assertSame(['pix_in', 'pix_out'], $row['capabilities']);
    }

    public function test_existing_gateways_without_declaration_expose_null(): void
    {
        $row = $this->gatewaysBySlug()['okto'];

        $this->assertNull($row['category']);
        $this->assertNull($row['capabilities']);
    }

    public function test_gateway_detail_json_exposes_category_and_capabilities(): void
    {
        $this->withoutMiddleware([EnsureInstalled::class, EnsureStackerLicense::class]);
        $this->registerFakeGateway('fakepay', null, ['category' => 'psp', 'capabilities' => ['pix_in']]);
        $admin = User::factory()->create(['role' => User::ROLE_PLATFORM_ADMIN, 'tenant_id' => null]);

        $this->actingAs($admin)
            ->getJson(route('plataforma.financeiro.gateways.show', 'fakepay'))
            ->assertOk()
            ->assertJsonPath('category', 'psp')
            ->assertJsonPath('capabilities', ['pix_in']);
    }
}
```

Imports adicionais nesse arquivo: `use App\Http\Middleware\EnsureInstalled;`, `use App\Http\Middleware\EnsureStackerLicense;` e `use App\Models\User;`.

- [ ] **Passo 3: Rodar e confirmar que falha**

Rode `php artisan test --filter=GatewayMetadataTest`.
Esperado: FAIL com "Class App\Support\GatewayCapabilities not found".

- [ ] **Passo 4: Implementar as classes de suporte**

`app/Support/GatewayCapabilities.php`:

```php
<?php

namespace App\Support;

/**
 * Capacidades declaradas na definição do gateway (`capabilities` em config/gateways.php ou no plugin).
 * Gateways sem a chave retornam null: a UI usa as listas legadas.
 */
final class GatewayCapabilities
{
    public const PIX_IN = 'pix_in';

    public const PIX_OUT = 'pix_out';

    public const REFUND = 'refund';

    /** @var list<string> */
    public const ALL = [self::PIX_IN, self::PIX_OUT, self::REFUND];

    /**
     * @param  array<string, mixed>  $definition
     * @return list<string>|null
     */
    public static function fromDefinition(array $definition): ?array
    {
        $raw = $definition['capabilities'] ?? null;
        if (! is_array($raw)) {
            return null;
        }

        $given = array_map(
            static fn ($c) => is_string($c) ? strtolower(trim($c)) : '',
            $raw
        );

        return array_values(array_intersect(self::ALL, $given));
    }

    /**
     * @param  array<string, mixed>  $definition
     */
    public static function has(array $definition, string $capability): bool
    {
        return in_array($capability, self::fromDefinition($definition) ?? [], true);
    }
}
```

`app/Support/GatewayCategories.php`:

```php
<?php

namespace App\Support;

/**
 * Categorias extras de gateway (além de psp / acquirer / open_finance, que ficam no frontend).
 */
final class GatewayCategories
{
    /**
     * @return list<array{id: string, title: string, dot_class: string}>
     */
    public static function extra(): array
    {
        $raw = config('gateways.categories', []);
        if (! is_array($raw)) {
            return [];
        }

        $out = [];
        foreach ($raw as $id => $definition) {
            if (! is_string($id) || trim($id) === '' || ! is_array($definition)) {
                continue;
            }
            $title = trim((string) ($definition['title'] ?? ''));
            if ($title === '') {
                continue;
            }
            $out[] = [
                'id' => $id,
                'title' => $title,
                'dot_class' => (string) ($definition['dot_class'] ?? 'bg-zinc-400'),
            ];
        }

        return $out;
    }
}
```

- [ ] **Passo 5: Expor nas props e no `show`**

Em `ProvidesPlatformGatewayProps::buildGatewaysListForSettings`, dentro do array retornado, logo depois de `'methods' => $g['methods'] ?? [],`, adicione:

```php
                'category' => is_string($g['category'] ?? null) && $g['category'] !== '' ? $g['category'] : null,
                'capabilities' => GatewayCapabilities::fromDefinition($g),
```

Adicione `use App\Support\GatewayCapabilities;` no topo do trait.

Em `GatewaysController::show`, no array `$payload`, logo depois de `'methods' => $gateway['methods'] ?? [],`, adicione:

```php
            'category' => is_string($gateway['category'] ?? null) && $gateway['category'] !== '' ? $gateway['category'] : null,
            'capabilities' => GatewayCapabilities::fromDefinition($gateway),
```

Adicione `use App\Support\GatewayCapabilities;` no topo do controller.

Em `FinancialController::index`, no array passado a `Inertia::render('Platform/Financial/Index', [...])`, logo depois de `'gateway_order' => ...`, adicione:

```php
            'gateway_categories' => GatewayCategories::extra(),
```

Adicione `use App\Support\GatewayCategories;` no topo.

Em `config/gateways.php`, logo **antes** de `'platform_acquirer_slugs' => [`, adicione:

```php
    /*
    |--------------------------------------------------------------------------
    | Categorias extras de gateway (tela Financeiro → Adquirentes).
    |--------------------------------------------------------------------------
    | As categorias padrão (psp, acquirer, open_finance) ficam no frontend.
    | Um gateway escolhe a categoria com a chave opcional 'category' na própria
    | definição; sem ela, vale o agrupamento legado por slug.
    | Formato: 'id' => ['title' => 'Título', 'dot_class' => 'bg-amber-500']
    */
    'categories' => [],
```

- [ ] **Passo 6: Rodar os testes e confirmar que passam**

Rode `php artisan test --filter=GatewayMetadataTest`.
Esperado: PASS (5 testes).

- [ ] **Passo 7: Commit**

```bash
git add app/Support/GatewayCapabilities.php app/Support/GatewayCategories.php config/gateways.php app/Http/Controllers/Concerns/ProvidesPlatformGatewayProps.php app/Http/Controllers/GatewaysController.php app/Http/Controllers/Platform/FinancialController.php tests/Concerns/RegistersFakeGateways.php tests/Unit/Gateways/GatewayMetadataTest.php
git commit -m "Let gateway definitions declare their category and capabilities so new providers do not need hardcoded slug lists in the admin."
```

### Tarefa 3: Agrupamento por categoria e badges a partir das capacidades (frontend)

**Arquivos:**
- Modificar: `resources/js/Pages/Platform/Financial/Index.vue` (props, `acquirerGroups`)
- Modificar: `resources/js/components/settings/PixInOutBadges.vue`
- Modificar: `resources/js/components/settings/GatewayCard.vue` (uso de `PixInOutBadges`)
- Modificar: `resources/js/components/settings/GatewayConfigSidebar.vue` (uso de `PixInOutBadges`)

**Interfaces:**
- Consome: das props de cada gateway, `category: ?string` e `capabilities: ?string[]`; da página, a prop `gateway_categories: {id, title, dot_class}[]` (tarefa 2).
- Produz: a prop nova `capabilities` (Array | null) em `PixInOutBadges.vue`.

O repositório não tem test runner de JavaScript, então a validação desta tarefa é `npm run build` mais uma conferência manual.

- [ ] **Passo 1: Badges a partir das capacidades**

Substitua o `<script setup>` de `resources/js/components/settings/PixInOutBadges.vue` por:

```vue
<script setup>
import { computed } from 'vue';

const props = defineProps({
    slug: { type: String, default: '' },
    /** Capacidades declaradas na definição do gateway (null = usa as listas legadas abaixo). */
    capabilities: { type: Array, default: null },
});

/** Cobrança PIX + saque PIX (plataforma). */
const SLUGS_PIX_IN_OUT = new Set(['cajupay', 'spacepag', 'woovi', 'bspay', 'onlyup', 'versell', 'xflow', 'okto']);
/** Só cobrança PIX (sem payout automático destes adquirentes nesta integração). */
const SLUGS_PIX_IN_ONLY = new Set(['efi', 'mercadopago', 'pagarme']);

const s = computed(() => (props.slug || '').toLowerCase());
const declared = computed(() => (Array.isArray(props.capabilities) ? props.capabilities : null));

const showPixInOut = computed(() =>
    declared.value ? declared.value.includes('pix_out') : SLUGS_PIX_IN_OUT.has(s.value)
);
const showPixInOnly = computed(() =>
    declared.value
        ? declared.value.includes('pix_in') && !declared.value.includes('pix_out')
        : SLUGS_PIX_IN_ONLY.has(s.value)
);
const visible = computed(() => showPixInOut.value || showPixInOnly.value);
</script>
```

O `<template>` não muda.

- [ ] **Passo 2: Repassar as capacidades**

Em `resources/js/components/settings/GatewayCard.vue`, troque `<PixInOutBadges :slug="gateway.slug" />` por:

```vue
<PixInOutBadges :slug="gateway.slug" :capabilities="gateway.capabilities ?? null" />
```

Em `resources/js/components/settings/GatewayConfigSidebar.vue`, troque `<PixInOutBadges :slug="gateway?.slug" />` por:

```vue
<PixInOutBadges :slug="gateway?.slug" :capabilities="gateway?.capabilities ?? null" />
```

Não altere `CajuPayAccountSidebar.vue`, que continua só com `slug="cajupay"`.

- [ ] **Passo 3: Agrupamento por categoria**

Em `resources/js/Pages/Platform/Financial/Index.vue`, adicione nas props (perto de `gateway_webhook_security_warnings`):

```js
    /** Categorias extras definidas em config/gateways.php → categories. */
    gateway_categories: { type: Array, default: () => [] },
```

Substitua o `computed` `acquirerGroups` inteiro (de `const acquirerGroups = computed(() => {` até o `});` correspondente) por:

```js
const extraCategoryGroups = computed(() =>
    (props.gateway_categories || []).map((c) => ({
        id: c.id,
        title: c.title,
        slugs: [],
        dotClass: c.dot_class || 'bg-zinc-400',
    }))
);

const acquirerGroups = computed(() => {
    const list = props.gateways || [];
    const baseGroups = [...ACQUIRER_GROUPS, ...extraCategoryGroups.value];
    const byId = Object.fromEntries(baseGroups.map((g) => [g.id, { ...g, items: [] }]));
    const used = new Set();

    // 1) Categoria declarada na definição do gateway.
    for (const g of list) {
        if (g.category && byId[g.category]) {
            byId[g.category].items.push(g);
            used.add(g.slug);
        }
    }

    // 2) Agrupamento legado por slug (gateways sem categoria declarada).
    const bySlug = Object.fromEntries(list.map((g) => [g.slug, g]));
    for (const group of baseGroups) {
        for (const slug of group.slugs) {
            const g = bySlug[slug];
            if (g && !used.has(slug)) {
                byId[group.id].items.push(g);
                used.add(slug);
            }
        }
    }

    // 3) Sem categoria e fora das listas: primeiro grupo (comportamento anterior).
    const groups = baseGroups.map((g) => byId[g.id]);
    const leftover = list.filter((g) => !used.has(g.slug));
    if (leftover.length > 0 && groups[0]) {
        groups[0].items.push(...leftover);
    }

    return groups.filter((g) => g.items.length > 0);
});
```

- [ ] **Passo 4: Build**

Rode `npm run build`.
Esperado: build concluído sem erros.

- [ ] **Passo 5: Conferência manual**

1. Suba o ambiente de dev e faça login como `platform_admin`.
2. Abra `/plataforma/financeiro`. Os grupos e os badges devem estar **idênticos** a antes.
3. Temporariamente, sem commitar, adicione à entrada `okto` em `config/gateways.php` as linhas `'category' => 'acquirer', 'capabilities' => ['pix_in'],`, e em `categories` a entrada `'banking' => ['title' => 'Banking', 'dot_class' => 'bg-amber-500']`. Rode `php artisan config:clear` e recarregue a página.
4. Esperado: o Okto aparece em "Adquirente" só com o badge "PIX in". O grupo "Banking" não aparece, porque está vazio.
5. Reverta as alterações temporárias.

- [ ] **Passo 6: Suíte completa e commit**

Rode `composer test`.
Esperado: tudo verde.

```bash
git add resources/js/Pages/Platform/Financial/Index.vue resources/js/components/settings/PixInOutBadges.vue resources/js/components/settings/GatewayCard.vue resources/js/components/settings/GatewayConfigSidebar.vue
git commit -m "Group gateways and show PIX in/out badges from their declared category and capabilities, keeping the legacy slug lists as fallback."
```

---

## PR 3 — `contrib/gateway-refund-credentials-contracts`

O helper `tests/Concerns/RegistersFakeGateways.php` vem do PR 2. Se o PR 2 ainda não estiver no upstream, traga-o para esta branch com `git cherry-pick -x <sha>` (só o commit da tarefa 2) e cite isso na descrição do PR.

### Tarefa 4: Estorno por contrato (`SupportsRefund`)

**Arquivos:**
- Criar: `app/Gateways/Contracts/SupportsRefund.php`
- Criar: `tests/Fixtures/Gateways/FakeGatewayDriver.php`
- Modificar: `app/Services/OrderRefundGatewayBridge.php` (braço `default` do `match` e método novo)
- Teste: `tests/Feature/ContractRefundTest.php`

**Interfaces:**
- Produz:
  - `SupportsRefund::refundTransaction(array $credentials, string $transactionId, float $amount, string $externalId): array`. O retorno segue o formato já esperado por `mapDriverRefundResult`: `success: bool`, `pending?: bool`, `message?: string`, `refund_id?: string`, `error_code?: string` e `raw?: array`;
  - fixture `Tests\Fixtures\Gateways\FakeGatewayDriver` com `public static array|\Throwable $refundResult`, `public static bool $credentialsReady`, `public static array $calls` e `public static function reset(): void`.

- [ ] **Passo 0: Criar a branch**

```bash
git fetch upstream
git switch -c contrib/gateway-refund-credentials-contracts upstream/main
```

- [ ] **Passo 1: Criar a interface e o fixture**

`app/Gateways/Contracts/SupportsRefund.php`:

```php
<?php

namespace App\Gateways\Contracts;

/**
 * Gateway que estorna uma cobrança via API.
 * Mesma assinatura do refundTransaction já usado pelos drivers do core.
 */
interface SupportsRefund
{
    /**
     * @param  array<string, mixed>  $credentials
     * @return array{success: bool, pending?: bool, message?: string, refund_id?: string, error_code?: string, raw?: array<string, mixed>}
     */
    public function refundTransaction(array $credentials, string $transactionId, float $amount, string $externalId): array;
}
```

`app/Gateways/Contracts/ValidatesCredentials.php` é criado na tarefa 5. Por isso, nesta tarefa o fixture ainda **não** implementa `ValidatesCredentials`.

`tests/Fixtures/Gateways/FakeGatewayDriver.php`:

```php
<?php

namespace Tests\Fixtures\Gateways;

use App\Gateways\Contracts\GatewayDriver;
use App\Gateways\Contracts\SupportsRefund;

/**
 * Driver falso para testar os pontos de extensão por contrato.
 * Estado estático porque o GatewayRegistry instancia um driver novo a cada chamada.
 */
class FakeGatewayDriver implements GatewayDriver, SupportsRefund
{
    /** @var array<string, mixed>|\Throwable */
    public static array|\Throwable $refundResult = ['success' => true];

    public static bool $credentialsReady = true;

    /** @var list<array{method: string, args: array<int, mixed>}> */
    public static array $calls = [];

    public static function reset(): void
    {
        self::$refundResult = ['success' => true];
        self::$credentialsReady = true;
        self::$calls = [];
    }

    public function testConnection(array $credentials): bool
    {
        return true;
    }

    public function createPixPayment(array $credentials, float $amount, array $consumer, string $externalId, string $postbackUrl, array $options = []): array
    {
        return ['transaction_id' => 'fake-'.$externalId, 'copy_paste' => '000201fake'];
    }

    public function getTransactionStatus(string $transactionId, array $credentials): ?string
    {
        return 'pending';
    }

    public function createCardPayment(array $credentials, float $amount, array $consumer, string $externalId, array $card): array
    {
        throw new \RuntimeException('Fake não suporta cartão.');
    }

    public function createBoletoPayment(array $credentials, float $amount, array $consumer, string $externalId, string $notificationUrl): array
    {
        throw new \RuntimeException('Fake não suporta boleto.');
    }

    public function refundTransaction(array $credentials, string $transactionId, float $amount, string $externalId): array
    {
        self::$calls[] = ['method' => 'refundTransaction', 'args' => func_get_args()];
        if (self::$refundResult instanceof \Throwable) {
            throw self::$refundResult;
        }

        return self::$refundResult;
    }
}
```

Antes de seguir, confira que as assinaturas acima batem **exatamente** com `app/Gateways/Contracts/GatewayDriver.php` (tipos e valores padrão). Se divergirem, ajuste o fixture para a interface real.

- [ ] **Passo 2: Escrever os testes que falham**

`tests/Feature/ContractRefundTest.php`:

```php
<?php

namespace Tests\Feature;

use App\Models\Order;
use App\Models\User;
use App\Services\OrderRefundGatewayBridge;
use Tests\Concerns\RegistersFakeGateways;
use Tests\Fixtures\Gateways\FakeGatewayDriver;
use Tests\TestCase;

class ContractRefundTest extends TestCase
{
    use RegistersFakeGateways;

    protected function setUp(): void
    {
        parent::setUp();
        FakeGatewayDriver::reset();
        $this->registerFakeGateway('fakepay', FakeGatewayDriver::class);
        $this->connectGlobalGatewayCredential('fakepay');
    }

    private function makeOrder(?string $gatewayId = 'fp-tx-1'): Order
    {
        $merchant = User::factory()->create(['role' => User::ROLE_INFOPRODUTOR]);
        $merchant->forceFill(['tenant_id' => $merchant->id])->save();
        $product = $this->createTestProduct(['tenant_id' => $merchant->id]);

        return Order::create([
            'tenant_id' => $merchant->id,
            'user_id' => $merchant->id,
            'product_id' => $product->id,
            'status' => 'completed',
            'amount' => 50,
            'gateway' => 'fakepay',
            'gateway_id' => $gatewayId,
            'payment_method' => 'pix',
            'email' => 'buyer@test.com',
        ]);
    }

    public function test_contract_refund_is_executed_and_completed(): void
    {
        FakeGatewayDriver::$refundResult = ['success' => true, 'refund_id' => 'rf-1', 'message' => 'ok'];
        $order = $this->makeOrder();

        $result = app(OrderRefundGatewayBridge::class)->tryRefund($order);

        $this->assertSame('gateway_ok', $result['status']);
        $call = FakeGatewayDriver::$calls[0];
        $this->assertSame('refundTransaction', $call['method']);
        $this->assertSame('fp-tx-1', $call['args'][1]);
        $this->assertSame(50.0, $call['args'][2]);
        $this->assertSame((string) $order->id, $call['args'][3]);
        $this->assertSame('rf-1', $order->fresh()->metadata['fakepay_refund_id'] ?? null);
    }

    public function test_pending_contract_refund_returns_gateway_pending(): void
    {
        FakeGatewayDriver::$refundResult = ['success' => true, 'pending' => true];
        $order = $this->makeOrder();

        $result = app(OrderRefundGatewayBridge::class)->tryRefund($order);

        $this->assertSame('gateway_pending', $result['status']);
        $this->assertTrue($order->fresh()->metadata['fakepay_refund_pending'] ?? false);
    }

    public function test_refund_exception_is_mapped_to_failed(): void
    {
        FakeGatewayDriver::$refundResult = new \RuntimeException('gateway timeout');
        $order = $this->makeOrder();

        $result = app(OrderRefundGatewayBridge::class)->tryRefund($order);

        $this->assertSame('failed', $result['status']);
    }

    public function test_order_without_transaction_id_is_skipped(): void
    {
        $order = $this->makeOrder(null);

        $result = app(OrderRefundGatewayBridge::class)->tryRefund($order);

        $this->assertSame('skipped', $result['status']);
        $this->assertSame([], FakeGatewayDriver::$calls);
    }
}
```

- [ ] **Passo 3: Rodar e confirmar que falha**

Rode `php artisan test --filter=ContractRefundTest`.
Esperado: FAIL. Os três primeiros testes recebem `status = skipped` ("Estorno automático não implementado…").

- [ ] **Passo 4: Implementar no `OrderRefundGatewayBridge`**

No `match ($gatewaySlug)` de `tryRefund`, substitua **apenas** o braço `default`:

```php
            default => $driver instanceof SupportsRefund
                ? $this->tryContractRefund($driver, $order, $credentials, $gatewaySlug)
                : ['status' => 'skipped', 'note' => 'Estorno automático não implementado para este gateway; conclua no adquirente se necessário.'],
```

Adicione o método, logo depois de `tryOktoRefund`:

```php
    /**
     * Estorno para gateways que implementam {@see SupportsRefund} e não têm braço próprio no match.
     *
     * @param  array<string, mixed>  $credentials
     * @return array{status: string, note: ?string, error_code?: string}
     */
    private function tryContractRefund(SupportsRefund $driver, Order $order, array $credentials, string $gatewaySlug): array
    {
        $transactionId = is_string($order->gateway_id) ? trim($order->gateway_id) : '';
        if ($transactionId === '') {
            return ['status' => 'skipped', 'note' => 'Sem ID de transação do gateway no pedido.'];
        }

        try {
            $result = $driver->refundTransaction($credentials, $transactionId, (float) $order->amount, (string) $order->id);
        } catch (\Throwable $e) {
            return $this->mapRefundException($order, $gatewaySlug, $e);
        }

        if (($result['success'] ?? false) === true) {
            $meta = is_array($order->metadata) ? $order->metadata : [];
            if (! empty($result['refund_id'])) {
                $meta[$gatewaySlug.'_refund_id'] = (string) $result['refund_id'];
            }
            $meta[$gatewaySlug.'_refund_pending'] = ! empty($result['pending']);
            $order->update(['metadata' => $meta]);

            if (! empty($result['pending'])) {
                return [
                    'status' => 'gateway_pending',
                    'note' => $result['message'] ?? 'Estorno enviado ao gateway; aguardando confirmação.',
                ];
            }
        }

        return $this->mapDriverRefundResult($order, $result, $gatewaySlug);
    }
```

Adicione `use App\Gateways\Contracts\SupportsRefund;` no topo.

- [ ] **Passo 5: Rodar e confirmar que passa**

Rode `php artisan test --filter=ContractRefundTest`.
Esperado: PASS (4 testes). Se `test_refund_exception_is_mapped_to_failed` vier com outro status, leia `mapRefundException` e ajuste **só o teste** para o status que esse método já produz hoje para exceções genéricas. O comportamento existente não muda.

- [ ] **Passo 6: Commit**

```bash
git add app/Gateways/Contracts/SupportsRefund.php app/Services/OrderRefundGatewayBridge.php tests/Fixtures/Gateways/FakeGatewayDriver.php tests/Feature/ContractRefundTest.php
git commit -m "Refund orders through any gateway that implements SupportsRefund so new providers do not need a slug arm in the refund bridge."
```

### Tarefa 5: Credenciais mínimas por contrato (`ValidatesCredentials`)

**Arquivos:**
- Criar: `app/Gateways/Contracts/ValidatesCredentials.php`
- Modificar: `tests/Fixtures/Gateways/FakeGatewayDriver.php` (implementar a interface)
- Modificar: `app/Support/GatewayApiCredentials.php` (braço `default` e método novo)
- Teste: `tests/Unit/Gateways/ContractCredentialsAndReportTest.php`

**Interfaces:**
- Produz: `ValidatesCredentials::hasRequiredCredentials(array $credentials): bool`.
- Consome: `FakeGatewayDriver::$credentialsReady` (tarefa 4).

- [ ] **Passo 1: Escrever o teste que falha**

`tests/Unit/Gateways/ContractCredentialsAndReportTest.php`:

```php
<?php

namespace Tests\Unit\Gateways;

use App\Support\GatewayApiCredentials;
use Tests\Concerns\RegistersFakeGateways;
use Tests\Fixtures\Gateways\FakeGatewayDriver;
use Tests\TestCase;

class ContractCredentialsAndReportTest extends TestCase
{
    use RegistersFakeGateways;

    protected function setUp(): void
    {
        parent::setUp();
        FakeGatewayDriver::reset();
    }

    public function test_driver_decides_if_credentials_are_ready(): void
    {
        $this->registerFakeGateway('fakepay', FakeGatewayDriver::class);

        FakeGatewayDriver::$credentialsReady = false;
        $this->assertFalse(GatewayApiCredentials::isReadyForGateway('fakepay', ['api_key' => 'x']));

        FakeGatewayDriver::$credentialsReady = true;
        $this->assertTrue(GatewayApiCredentials::isReadyForGateway('fakepay', ['api_key' => 'x']));
    }

    public function test_gateway_without_contract_keeps_default_true(): void
    {
        $this->registerFakeGateway('nodriver', null);

        $this->assertTrue(GatewayApiCredentials::isReadyForGateway('nodriver', ['api_key' => 'x']));
    }

    public function test_empty_credentials_are_still_rejected_before_contract(): void
    {
        $this->registerFakeGateway('fakepay', FakeGatewayDriver::class);

        $this->assertFalse(GatewayApiCredentials::isReadyForGateway('fakepay', []));
    }
}
```

- [ ] **Passo 2: Rodar e confirmar que falha**

Rode `php artisan test --filter=ContractCredentialsAndReportTest`.
Esperado: FAIL em `test_driver_decides_if_credentials_are_ready`, porque o braço `default` hoje sempre retorna `true`.

- [ ] **Passo 3: Implementar**

`app/Gateways/Contracts/ValidatesCredentials.php`:

```php
<?php

namespace App\Gateways\Contracts;

/**
 * Gateway que informa se as credenciais têm o mínimo para chamar a API (sem fazer requisição).
 */
interface ValidatesCredentials
{
    /**
     * @param  array<string, mixed>  $credentials
     */
    public function hasRequiredCredentials(array $credentials): bool;
}
```

Em `tests/Fixtures/Gateways/FakeGatewayDriver.php`:
- mude a declaração para `class FakeGatewayDriver implements GatewayDriver, SupportsRefund, ValidatesCredentials`;
- adicione `use App\Gateways\Contracts\ValidatesCredentials;`;
- adicione o método:

```php
    public function hasRequiredCredentials(array $credentials): bool
    {
        return self::$credentialsReady;
    }
```

Em `app/Support/GatewayApiCredentials.php`, troque o braço `default => true,` por:

```php
            default => self::driverAcceptsCredentials($gatewaySlug, $credentials),
```

E adicione, antes do `}` final da classe:

```php
    /**
     * @param  array<string, mixed>  $credentials
     */
    private static function driverAcceptsCredentials(string $gatewaySlug, array $credentials): bool
    {
        $driver = GatewayRegistry::driver($gatewaySlug);

        return $driver instanceof ValidatesCredentials
            ? $driver->hasRequiredCredentials($credentials)
            : true;
    }
```

Adicione `use App\Gateways\Contracts\ValidatesCredentials;` e `use App\Gateways\GatewayRegistry;` no topo.

- [ ] **Passo 4: Rodar e confirmar que passa**

Rode `php artisan test --filter="ContractCredentialsAndReportTest|ContractRefundTest"`.
Esperado: PASS.

- [ ] **Passo 5: Commit**

```bash
git add app/Gateways/Contracts/ValidatesCredentials.php app/Support/GatewayApiCredentials.php tests/Fixtures/Gateways/FakeGatewayDriver.php tests/Unit/Gateways/ContractCredentialsAndReportTest.php
git commit -m "Ask the gateway driver whether its credentials are complete when it implements ValidatesCredentials, instead of assuming they are."
```

### Tarefa 6: Classificação no relatório pelo `methods` da definição + documentação

**Arquivos:**
- Modificar: `app/Models/Order.php` (`paymentMethodReportKey`, antes do `return 'outro';`)
- Modificar: `tests/Unit/Gateways/ContractCredentialsAndReportTest.php` (testes novos)
- Modificar: `app/Gateways/README.md` (seção nova)

**Interfaces:**
- Consome: `GatewayRegistry::get(string $slug): ?array` (já existe; `Order.php` já importa `GatewayRegistry`).

- [ ] **Passo 1: Escrever os testes que falham**

Adicione em `ContractCredentialsAndReportTest`:

```php
    public function test_report_key_uses_single_method_from_gateway_definition(): void
    {
        $this->registerFakeGateway('fakepay', null, ['methods' => ['pix']]);
        $this->registerFakeGateway('fakecard', null, ['methods' => ['card']]);
        $this->registerFakeGateway('fakemulti', null, ['methods' => ['pix', 'card']]);

        $order = new \App\Models\Order;

        $order->forceFill(['gateway' => 'fakepay', 'payment_method' => null, 'metadata' => null]);
        $this->assertSame('pix', $order->paymentMethodReportKey());

        $order->forceFill(['gateway' => 'fakecard']);
        $this->assertSame('card', $order->paymentMethodReportKey());

        $order->forceFill(['gateway' => 'fakemulti']);
        $this->assertSame('outro', $order->paymentMethodReportKey());
    }
```

- [ ] **Passo 2: Rodar e confirmar que falha**

Rode `php artisan test --filter=test_report_key_uses_single_method_from_gateway_definition`.
Esperado: FAIL ("outro" ≠ "pix").

- [ ] **Passo 3: Implementar**

Em `Order::paymentMethodReportKey()`, imediatamente antes da última linha `return 'outro';`, insira:

```php
        $methods = GatewayRegistry::get($gateway)['methods'] ?? [];
        if ($methods === ['pix']) {
            return 'pix';
        }
        if ($methods === ['card']) {
            return 'card';
        }
```

- [ ] **Passo 4: Rodar e confirmar que passa**

Rode `php artisan test --filter=ContractCredentialsAndReportTest`.
Esperado: PASS.

- [ ] **Passo 5: Documentar as interfaces no README dos gateways**

No fim de `app/Gateways/README.md`, acrescente:

```markdown
## Capacidades por contrato (sem `match` por slug)

Além de `Contracts\GatewayDriver`, um driver pode implementar:

| Interface | Efeito |
|---|---|
| `Contracts\SupportsRefund` | Estorno automático via `OrderRefundGatewayBridge` (metadata `{slug}_refund_id` / `{slug}_refund_pending`) |
| `Contracts\ValidatesCredentials` | `GatewayApiCredentials::isReadyForGateway` consulta `hasRequiredCredentials()` |

Na definição do gateway (config ou plugin), as chaves opcionais:

- `category`: `psp`, `acquirer`, `open_finance` ou um id de `config('gateways.categories')`. Agrupa o card em Financeiro → Adquirentes.
- `capabilities`: lista com `pix_in`, `pix_out` e/ou `refund`. Define os badges PIX in/out.

Gateways com braço próprio nos `match` continuam usando o braço legado.
```

- [ ] **Passo 6: Suíte completa e commit**

Rode `composer test`.
Esperado: tudo verde.

```bash
git add app/Models/Order.php app/Gateways/README.md tests/Unit/Gateways/ContractCredentialsAndReportTest.php
git commit -m "Classify orders from gateways with a single declared method in reports, and document the refund and credential contracts."
```

---

## PR 4 — `contrib/gateway-payout-contract`

Este PR usa `RegistersFakeGateways` (PR 2), `FakeGatewayDriver` (PR 3) e `redirectAfterAdminPayoutAttempt` (PR 1). Se algum deles ainda não estiver no upstream, traga os commits com `git cherry-pick -x` e cite a dependência na descrição do PR.

### Tarefa 7: Interface `SupportsPayout` e seleção do provedor de payout

**Arquivos:**
- Criar: `app/Gateways/Contracts/SupportsPayout.php`
- Criar: `tests/Fixtures/Gateways/FakePayoutGatewayDriver.php`
- Modificar: `app/Services/Payout/PlatformPayoutGateway.php`
- Teste: `tests/Unit/Gateways/PlatformPayoutGatewayContractTest.php`

**Interfaces:**
- Produz:
  - `SupportsPayout::sendPixPayout(array $credentials, int $amountCents, array $destination, string $externalId): array{ok: bool, transaction_id?: string, error?: string}`, onde `$destination` tem o formato `array{pix_key: string, pix_key_type: string, key_owner_document: string}`;
  - `SupportsPayout::getPayoutStatus(string $transactionId, array $credentials): ?string`, que retorna `'paid'`, `'pending'`, `'failed'`, `'cancelled'` ou `null`;
  - `PlatformPayoutGateway::contractSlugs(): list<string>`;
  - `PlatformPayoutGateway::supportsPayoutContract(?string $slug): bool`;
  - `PlatformPayoutGateway::payoutOrder(): list<string>`;
  - `PlatformPayoutGateway::contractOptions(): list<array{slug: string, name: string}>`;
  - fixture `FakePayoutGatewayDriver` com `public static array|\Throwable $payoutResult` e `public static ?string $payoutStatus`.

- [ ] **Passo 0: Criar a branch**

```bash
git fetch upstream
git switch -c contrib/gateway-payout-contract upstream/main
```

- [ ] **Passo 1: Criar a interface e o fixture**

`app/Gateways/Contracts/SupportsPayout.php`:

```php
<?php

namespace App\Gateways\Contracts;

/**
 * Gateway que envia saques (PIX OUT) da plataforma.
 * Taxas, mínimo e destino são resolvidos pelo core (ContractPayoutService); o driver só transfere.
 */
interface SupportsPayout
{
    /**
     * @param  array<string, mixed>  $credentials
     * @param  array{pix_key: string, pix_key_type: string, key_owner_document: string}  $destination
     * @return array{ok: bool, transaction_id?: string, error?: string}
     */
    public function sendPixPayout(array $credentials, int $amountCents, array $destination, string $externalId): array;

    /**
     * @param  array<string, mixed>  $credentials
     * @return 'paid'|'pending'|'failed'|'cancelled'|null
     */
    public function getPayoutStatus(string $transactionId, array $credentials): ?string;
}
```

`tests/Fixtures/Gateways/FakePayoutGatewayDriver.php`:

```php
<?php

namespace Tests\Fixtures\Gateways;

use App\Gateways\Contracts\SupportsPayout;

class FakePayoutGatewayDriver extends FakeGatewayDriver implements SupportsPayout
{
    /** @var array<string, mixed>|\Throwable */
    public static array|\Throwable $payoutResult = ['ok' => true, 'transaction_id' => 'fake-payout-1'];

    public static ?string $payoutStatus = 'pending';

    public static function reset(): void
    {
        parent::reset();
        self::$payoutResult = ['ok' => true, 'transaction_id' => 'fake-payout-1'];
        self::$payoutStatus = 'pending';
    }

    public function sendPixPayout(array $credentials, int $amountCents, array $destination, string $externalId): array
    {
        self::$calls[] = ['method' => 'sendPixPayout', 'args' => func_get_args()];
        if (self::$payoutResult instanceof \Throwable) {
            throw self::$payoutResult;
        }

        return self::$payoutResult;
    }

    public function getPayoutStatus(string $transactionId, array $credentials): ?string
    {
        self::$calls[] = ['method' => 'getPayoutStatus', 'args' => func_get_args()];

        return self::$payoutStatus;
    }
}
```

- [ ] **Passo 2: Escrever os testes que falham**

`tests/Unit/Gateways/PlatformPayoutGatewayContractTest.php`:

```php
<?php

namespace Tests\Unit\Gateways;

use App\Models\Setting;
use App\Services\Payout\PlatformPayoutGateway;
use Tests\Concerns\RegistersFakeGateways;
use Tests\Fixtures\Gateways\FakeGatewayDriver;
use Tests\Fixtures\Gateways\FakePayoutGatewayDriver;
use Tests\TestCase;

class PlatformPayoutGatewayContractTest extends TestCase
{
    use RegistersFakeGateways;

    protected function setUp(): void
    {
        parent::setUp();
        FakePayoutGatewayDriver::reset();
        Setting::set('platform_payout_gateway', null, null);
    }

    public function test_contract_gateway_becomes_active_when_it_is_the_only_one_connected(): void
    {
        $this->registerFakeGateway('fakepay', FakePayoutGatewayDriver::class);
        $this->connectGlobalGatewayCredential('fakepay');

        $this->assertSame(['fakepay'], PlatformPayoutGateway::contractSlugs());
        $this->assertTrue(PlatformPayoutGateway::supportsPayoutContract('fakepay'));
        $this->assertSame('fakepay', PlatformPayoutGateway::activeSlug());
        $this->assertSame([['slug' => 'fakepay', 'name' => 'Fake Fakepay']], PlatformPayoutGateway::contractOptions());
    }

    public function test_legacy_gateways_keep_priority_in_auto_mode(): void
    {
        $this->registerFakeGateway('fakepay', FakePayoutGatewayDriver::class);
        $this->connectGlobalGatewayCredential('fakepay');
        $this->connectGlobalGatewayCredential('woovi', ['app_id' => 'app', 'from_pix_key' => 'pix@test.com']);

        $this->assertSame('woovi', PlatformPayoutGateway::activeSlug());
    }

    public function test_contract_gateway_can_be_forced_as_preference(): void
    {
        $this->registerFakeGateway('fakepay', FakePayoutGatewayDriver::class);
        $this->connectGlobalGatewayCredential('fakepay');
        $this->connectGlobalGatewayCredential('woovi', ['app_id' => 'app', 'from_pix_key' => 'pix@test.com']);
        Setting::set('platform_payout_gateway', 'fakepay', null);

        $this->assertSame('fakepay', PlatformPayoutGateway::preference());
        $this->assertSame('fakepay', PlatformPayoutGateway::activeSlug());
    }

    public function test_gateway_without_payout_contract_is_ignored(): void
    {
        $this->registerFakeGateway('fakein', FakeGatewayDriver::class);
        $this->connectGlobalGatewayCredential('fakein');

        $this->assertSame([], PlatformPayoutGateway::contractSlugs());
        $this->assertNull(PlatformPayoutGateway::activeSlug());
    }

    public function test_contract_gateway_outside_platform_acquirer_slugs_is_ignored(): void
    {
        $this->registerFakeGateway('fakepay', FakePayoutGatewayDriver::class);
        config()->set('gateways.platform_acquirer_slugs', array_values(array_diff(
            config('gateways.platform_acquirer_slugs'),
            ['fakepay']
        )));
        $this->connectGlobalGatewayCredential('fakepay');

        $this->assertSame([], PlatformPayoutGateway::contractSlugs());
        $this->assertNull(PlatformPayoutGateway::activeSlug());
    }

    public function test_disconnected_contract_gateway_is_not_active(): void
    {
        $this->registerFakeGateway('fakepay', FakePayoutGatewayDriver::class);
        $this->connectGlobalGatewayCredential('fakepay', ['api_key' => 'x'], connected: false);

        $this->assertNull(PlatformPayoutGateway::activeSlug());
    }
}
```

- [ ] **Passo 3: Rodar e confirmar que falha**

Rode `php artisan test --filter=PlatformPayoutGatewayContractTest`.
Esperado: FAIL com "Call to undefined method …::contractSlugs()".

- [ ] **Passo 4: Implementar no `PlatformPayoutGateway`**

Adicione os métodos depois da constante `PAYOUT_ORDER`:

```php
    /**
     * Gateways habilitados cujo driver implementa SupportsPayout e que não têm fluxo legado próprio.
     *
     * @return list<string>
     */
    public static function contractSlugs(): array
    {
        $slugs = [];
        foreach (GatewayRegistry::allowedAcquirers() as $definition) {
            $slug = (string) ($definition['slug'] ?? '');
            if ($slug === '' || $slug === 'spacepag' || in_array($slug, self::PAYOUT_ORDER, true)) {
                continue;
            }
            $driver = $definition['driver'] ?? null;
            if (is_string($driver) && class_exists($driver) && is_subclass_of($driver, SupportsPayout::class)) {
                $slugs[] = $slug;
            }
        }

        return $slugs;
    }

    public static function supportsPayoutContract(?string $slug): bool
    {
        return $slug !== null && $slug !== '' && in_array($slug, self::contractSlugs(), true);
    }

    /**
     * Ordem automática: legados primeiro, depois os gateways por contrato.
     *
     * @return list<string>
     */
    public static function payoutOrder(): array
    {
        return array_merge(self::PAYOUT_ORDER, self::contractSlugs());
    }

    /**
     * @return list<array{slug: string, name: string}>
     */
    public static function contractOptions(): array
    {
        return array_map(
            static fn (string $slug) => ['slug' => $slug, 'name' => (string) (GatewayRegistry::get($slug)['name'] ?? $slug)],
            self::contractSlugs()
        );
    }
```

Em `preference()`, troque a condição:

```php
        if (in_array($v, ['cajupay', /* 'spacepag', */ 'woovi', 'bspay', 'versell', 'xflow', 'okto', 'onlyup'], true)) {
```

por:

```php
        if (in_array($v, ['cajupay', /* 'spacepag', */ 'woovi', 'bspay', 'versell', 'xflow', 'okto', 'onlyup'], true)
            || (is_string($v) && self::supportsPayoutContract($v))) {
```

Atualize o docblock de `preference()` para `@return string` ("'auto', um slug legado ou um slug por contrato").

Em `activeSlug()`, troque as **duas** ocorrências de `foreach (self::PAYOUT_ORDER as $slug)` por `foreach (self::payoutOrder() as $slug)`.

Adicione `use App\Gateways\Contracts\SupportsPayout;` e `use App\Gateways\GatewayRegistry;` no topo.

- [ ] **Passo 5: Rodar e confirmar que passa**

Rode `php artisan test --filter="PlatformPayoutGatewayContractTest|PlatformPayoutGatewayTest"`.
Esperado: PASS. O teste antigo `PlatformPayoutGatewayTest` continua verde.

- [ ] **Passo 6: Commit**

```bash
git add app/Gateways/Contracts/SupportsPayout.php app/Services/Payout/PlatformPayoutGateway.php tests/Fixtures/Gateways/FakePayoutGatewayDriver.php tests/Unit/Gateways/PlatformPayoutGatewayContractTest.php
git commit -m "Let gateways that implement SupportsPayout become the platform payout gateway after the legacy ones, without being added to hardcoded lists."
```

### Tarefa 8: Envio do saque e reconciliação por contrato

**Arquivos:**
- Criar: `app/Services/Payout/ContractPayoutService.php`
- Criar: `app/Jobs/ReconcileContractPayoutWithdrawalJob.php`
- Modificar: `app/Services/WithdrawalAutoPayoutService.php` (braço `default` e `attemptViaContract`)
- Teste: `tests/Feature/ContractPayoutFlowTest.php`

**Interfaces:**
- Consome: `PlatformPayoutGateway::supportsPayoutContract()` e `SupportsPayout` (tarefa 7).
- Produz:
  - `ContractPayoutService::send(Withdrawal $withdrawal, User $owner, string $slug): array{ok: bool, skipped?: bool, reason?: string, transaction_id?: ?string, error?: string}`;
  - `WithdrawalAutoPayoutService::attemptViaContract(Withdrawal $withdrawal, string $slug, bool $auto = true): array{ok: bool, skipped?: bool, reason?: string, error?: string, pending?: bool}`;
  - `ReconcileContractPayoutWithdrawalJob(int $withdrawalId)`.

- [ ] **Passo 1: Escrever os testes que falham**

`tests/Feature/ContractPayoutFlowTest.php`:

```php
<?php

namespace Tests\Feature;

use App\Jobs\ReconcileContractPayoutWithdrawalJob;
use App\Models\Setting;
use App\Models\User;
use App\Models\Withdrawal;
use App\Services\WithdrawalAutoPayoutService;
use Illuminate\Support\Facades\Queue;
use Tests\Concerns\RegistersFakeGateways;
use Tests\Fixtures\Gateways\FakePayoutGatewayDriver;
use Tests\TestCase;

class ContractPayoutFlowTest extends TestCase
{
    use RegistersFakeGateways;

    protected function setUp(): void
    {
        parent::setUp();
        FakePayoutGatewayDriver::reset();
        Setting::set('platform_payout_gateway', null, null);
        $this->registerFakeGateway('fakepay', FakePayoutGatewayDriver::class);
        Queue::fake();
    }

    /**
     * @param  array<string, mixed>  $payoutSettings
     * @param  array<string, mixed>  $attributes
     */
    private function makeWithdrawal(float $net, array $payoutSettings, array $attributes = []): Withdrawal
    {
        $seller = User::factory()->create(['role' => User::ROLE_INFOPRODUTOR]);
        $seller->forceFill(['tenant_id' => $seller->id, 'payout_settings' => $payoutSettings])->save();

        return Withdrawal::query()->create(array_merge([
            'tenant_id' => $seller->id,
            'user_id' => $seller->id,
            'amount' => $net,
            'fee_amount' => 0,
            'net_amount' => $net,
            'bucket' => 'pix',
            'status' => 'pending',
            'currency' => 'BRL',
        ], $attributes));
    }

    /**
     * @return list<array{method: string, args: array<int, mixed>}>
     */
    private function payoutCalls(): array
    {
        return array_values(array_filter(
            FakePayoutGatewayDriver::$calls,
            static fn ($c) => $c['method'] === 'sendPixPayout'
        ));
    }

    public function test_auto_payout_sends_net_amount_and_dispatches_reconcile(): void
    {
        $this->connectGlobalGatewayCredential('fakepay');
        $w = $this->makeWithdrawal(100, ['payout_pix_key' => '11119854997', 'payout_pix_key_type' => 'cpf']);

        $result = app(WithdrawalAutoPayoutService::class)->attemptAutoPayout($w->fresh());

        $this->assertTrue($result['ok'] ?? false, json_encode($result));
        $this->assertTrue($result['pending'] ?? false);
        $fresh = $w->fresh();
        $this->assertSame('fakepay', $fresh->payout_provider);
        $this->assertSame('fake-payout-1', $fresh->payout_external_id);
        $this->assertTrue($fresh->payout_meta['auto'] ?? false);

        $call = $this->payoutCalls()[0];
        $this->assertSame(10000, $call['args'][1]);
        $this->assertSame('11119854997', $call['args'][2]['pix_key']);
        $this->assertSame('cpf', $call['args'][2]['pix_key_type']);
        $this->assertSame((string) $w->id, $call['args'][3]);
        Queue::assertPushed(ReconcileContractPayoutWithdrawalJob::class);
    }

    public function test_admin_payout_fee_is_added_to_transfer_amount(): void
    {
        $this->connectGlobalGatewayCredential('fakepay', ['api_key' => 'x', 'fakepay_admin_fee_payout_brl' => '2']);
        $w = $this->makeWithdrawal(16, ['payout_pix_key' => 'destino@pix.com', 'payout_pix_key_type' => 'email']);

        app(WithdrawalAutoPayoutService::class)->attemptAutoPayout($w->fresh());

        $this->assertSame(1800, $this->payoutCalls()[0]['args'][1]);
    }

    public function test_missing_pix_key_skips_without_calling_driver(): void
    {
        $this->connectGlobalGatewayCredential('fakepay');
        $w = $this->makeWithdrawal(100, []);

        $result = app(WithdrawalAutoPayoutService::class)->attemptAutoPayout($w->fresh());

        $this->assertTrue($result['skipped'] ?? false);
        $this->assertSame('no_pix_key', $result['reason'] ?? null);
        $this->assertSame([], $this->payoutCalls());
    }

    public function test_driver_error_is_recorded(): void
    {
        $this->connectGlobalGatewayCredential('fakepay');
        FakePayoutGatewayDriver::$payoutResult = ['ok' => false, 'error' => 'saldo insuficiente'];
        $w = $this->makeWithdrawal(100, ['payout_pix_key' => '11119854997', 'payout_pix_key_type' => 'cpf']);

        $result = app(WithdrawalAutoPayoutService::class)->attemptAutoPayout($w->fresh());

        $this->assertFalse($result['ok']);
        $this->assertSame('saldo insuficiente', $result['error'] ?? null);
        $this->assertSame('saldo insuficiente', $w->fresh()->payout_meta['last_error'] ?? null);
    }

    public function test_driver_exception_is_recorded_as_error(): void
    {
        $this->connectGlobalGatewayCredential('fakepay');
        FakePayoutGatewayDriver::$payoutResult = new \RuntimeException('gateway timeout');
        $w = $this->makeWithdrawal(100, ['payout_pix_key' => '11119854997', 'payout_pix_key_type' => 'cpf']);

        $result = app(WithdrawalAutoPayoutService::class)->attemptAutoPayout($w->fresh());

        $this->assertFalse($result['ok']);
        $this->assertSame('gateway timeout', $result['error'] ?? null);
        $this->assertSame('gateway timeout', $w->fresh()->payout_meta['last_error'] ?? null);
    }

    public function test_reconcile_job_marks_paid(): void
    {
        $this->connectGlobalGatewayCredential('fakepay');
        FakePayoutGatewayDriver::$payoutStatus = 'paid';
        $w = $this->makeWithdrawal(100, [], [
            'status' => 'processing',
            'payout_provider' => 'fakepay',
            'payout_external_id' => 'fp-1',
        ]);

        (new ReconcileContractPayoutWithdrawalJob($w->id))->handle();

        $this->assertSame('paid', $w->fresh()->status);
    }

    public function test_reconcile_job_marks_failed(): void
    {
        $this->connectGlobalGatewayCredential('fakepay');
        FakePayoutGatewayDriver::$payoutStatus = 'failed';
        $w = $this->makeWithdrawal(100, [], [
            'status' => 'processing',
            'payout_provider' => 'fakepay',
            'payout_external_id' => 'fp-1',
        ]);

        (new ReconcileContractPayoutWithdrawalJob($w->id))->handle();

        $this->assertSame('failed', $w->fresh()->status);
    }
}
```

- [ ] **Passo 2: Rodar e confirmar que falha**

Rode `php artisan test --filter=ContractPayoutFlowTest`.
Esperado: FAIL. A classe `ReconcileContractPayoutWithdrawalJob` ainda não existe, e o `attemptAutoPayout` retorna `no_payout_gateway`.

- [ ] **Passo 3: Criar o `ContractPayoutService`**

`app/Services/Payout/ContractPayoutService.php`:

```php
<?php

namespace App\Services\Payout;

use App\Gateways\Contracts\SupportsPayout;
use App\Gateways\GatewayRegistry;
use App\Models\GatewayCredential;
use App\Models\User;
use App\Models\Withdrawal;
use App\Services\EffectiveMerchantFees;
use App\Services\Withdrawal\WithdrawalMinimumService;
use Illuminate\Support\Facades\Log;

/**
 * Envia saque PIX por gateways que implementam {@see SupportsPayout}.
 * Taxas e mínimo seguem as chaves {slug}_payout_min_brl / {slug}_admin_fee_payout_* (GatewayPayoutEconomics).
 */
class ContractPayoutService
{
    /**
     * @return array{ok: bool, skipped?: bool, reason?: string, transaction_id?: ?string, error?: string}
     */
    public function send(Withdrawal $withdrawal, User $owner, string $slug): array
    {
        $driver = GatewayRegistry::driver($slug);
        if (! $driver instanceof SupportsPayout) {
            return ['ok' => false, 'skipped' => true, 'reason' => 'no_payout_contract'];
        }

        $credential = GatewayCredential::resolveForPayment(null, $slug);
        if ($credential === null || ! $credential->is_connected) {
            return ['ok' => false, 'skipped' => true, 'reason' => $slug.'_not_configured'];
        }
        $credentials = $credential->getDecryptedCredentials();

        $net = (float) $withdrawal->net_amount;
        if ($net <= 0) {
            return ['ok' => false, 'error' => 'Valor líquido do saque inválido.'];
        }

        $economics = GatewayPayoutEconomics::fromCredentialsArray($slug, $credentials);
        $requiredNet = WithdrawalMinimumService::effectiveRequiredMinNet($economics);
        if ((int) round($net * 100) < (int) max(1, (int) round($requiredNet * 100))) {
            $minGross = EffectiveMerchantFees::minimumWithdrawalGrossForTargetNet((int) $withdrawal->tenant_id, $requiredNet);

            return [
                'ok' => false,
                'error' => $minGross !== null
                    ? 'O valor mínimo do saque é R$ '.number_format($minGross, 2, ',', '.').' (valor total a solicitar).'
                    : 'O valor solicitado é inferior ao mínimo permitido.',
            ];
        }

        $settings = is_array($owner->payout_settings) ? $owner->payout_settings : [];
        $fromWithdrawal = WithdrawalPayoutDestination::fromWithdrawal($withdrawal);
        $destination = [
            'pix_key' => (string) ($fromWithdrawal['pix_key'] ?? PayoutUserSettings::pixKey($settings)),
            'pix_key_type' => (string) ($fromWithdrawal['pix_key_type'] ?? PayoutUserSettings::pixKeyType($settings)),
            'key_owner_document' => (string) ($fromWithdrawal['key_owner_document'] ?? ''),
        ];
        if ($destination['pix_key'] === '') {
            return ['ok' => false, 'skipped' => true, 'reason' => 'no_pix_key'];
        }

        $amountBrl = GatewayPayoutEconomics::transferAmountBrlForApi(
            $net,
            $economics['admin_fee_payout_brl'],
            $economics['admin_fee_payout_percent'] ?? 0.0,
        );
        $amountCents = (int) max(1, (int) round($amountBrl * 100));

        try {
            $result = $driver->sendPixPayout($credentials, $amountCents, $destination, (string) $withdrawal->id);
        } catch (\Throwable $e) {
            Log::warning('ContractPayoutService: falha ao enviar payout', [
                'gateway' => $slug,
                'withdrawal_id' => $withdrawal->id,
                'message' => $e->getMessage(),
            ]);

            return ['ok' => false, 'error' => $e->getMessage() !== '' ? $e->getMessage() : 'Falha ao enviar o saque via PIX.'];
        }

        if (! ($result['ok'] ?? false)) {
            return ['ok' => false, 'error' => (string) ($result['error'] ?? 'Falha ao enviar o saque via PIX.')];
        }

        return [
            'ok' => true,
            'transaction_id' => isset($result['transaction_id']) ? (string) $result['transaction_id'] : null,
        ];
    }
}
```

- [ ] **Passo 4: Criar o job de reconciliação**

`app/Jobs/ReconcileContractPayoutWithdrawalJob.php`:

```php
<?php

namespace App\Jobs;

use App\Gateways\Contracts\SupportsPayout;
use App\Gateways\GatewayRegistry;
use App\Models\GatewayCredential;
use App\Models\Withdrawal;
use App\Services\MerchantWithdrawalService;
use App\Services\Payout\PlatformPayoutGateway;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;

/**
 * Reconcilia um saque enviado por gateway que implementa {@see SupportsPayout}.
 */
class ReconcileContractPayoutWithdrawalJob implements ShouldQueue
{
    use Dispatchable;
    use InteractsWithQueue;
    use Queueable;
    use SerializesModels;

    public int $timeout = 90;

    public function __construct(public int $withdrawalId) {}

    public function handle(): void
    {
        $withdrawal = Withdrawal::query()->find($this->withdrawalId);
        if ($withdrawal === null || ! in_array($withdrawal->status, ['pending', 'processing'], true)) {
            return;
        }

        $slug = (string) $withdrawal->payout_provider;
        if (! PlatformPayoutGateway::supportsPayoutContract($slug)) {
            return;
        }

        $driver = GatewayRegistry::driver($slug);
        if (! $driver instanceof SupportsPayout) {
            return;
        }

        $tx = trim((string) $withdrawal->payout_external_id);
        if ($tx === '') {
            $tx = (string) $withdrawal->id;
        }

        $cred = GatewayCredential::resolveForPayment(null, $slug);
        if ($cred === null || ! $cred->is_connected) {
            $this->maybeReleaseForRetry($slug);

            return;
        }

        $credentials = $cred->getDecryptedCredentials();
        if ($credentials === []) {
            return;
        }

        try {
            $status = $driver->getPayoutStatus($tx, $credentials);
        } catch (\Throwable $e) {
            Log::warning('ReconcileContractPayoutWithdrawalJob: falha na consulta', [
                'gateway' => $slug,
                'withdrawal_id' => $this->withdrawalId,
                'message' => $e->getMessage(),
            ]);
            $this->maybeReleaseForRetry($slug);

            return;
        }

        if ($status === 'paid') {
            MerchantWithdrawalService::markPaid($withdrawal->fresh());

            return;
        }

        if (in_array($status, ['failed', 'cancelled'], true)) {
            MerchantWithdrawalService::markFailed(
                $withdrawal->fresh(),
                'Payout '.$slug.' falhou (reconciliação): '.$status
            );

            return;
        }

        $this->maybeReleaseForRetry($slug);
    }

    private function maybeReleaseForRetry(string $slug): void
    {
        if (config('queue.default') === 'sync') {
            return;
        }

        if ($this->attempts() >= 36) {
            $withdrawal = Withdrawal::query()->find($this->withdrawalId);
            if ($withdrawal !== null && in_array($withdrawal->status, ['pending', 'processing'], true)) {
                MerchantWithdrawalService::markFailed(
                    $withdrawal->fresh(),
                    'Payout '.$slug.' não confirmado após esgotar tentativas de reconciliação.'
                );
            }

            return;
        }

        $this->release(120);
    }
}
```

- [ ] **Passo 5: Ligar no `WithdrawalAutoPayoutService`**

No `match ($slug)` de `attemptAutoPayout`, substitua **apenas** o braço `default`:

```php
            default => PlatformPayoutGateway::supportsPayoutContract($slug)
                ? $this->attemptViaContract($withdrawal, (string) $slug)
                : ['ok' => false, 'skipped' => true, 'reason' => 'no_payout_gateway'],
```

Adicione o método público, logo depois de `attemptOnlyUp`:

```php
    /**
     * Saque por gateway que implementa SupportsPayout (sem fluxo legado próprio).
     *
     * @return array{ok: bool, skipped?: bool, reason?: string, error?: string, pending?: bool}
     */
    public function attemptViaContract(Withdrawal $withdrawal, string $slug, bool $auto = true): array
    {
        if ($withdrawal->status !== MerchantWithdrawalService::STATUS_PROCESSING) {
            return ['ok' => false, 'skipped' => true, 'reason' => 'not_processing'];
        }

        $tenantId = (int) $withdrawal->tenant_id;
        $owner = User::query()
            ->where('tenant_id', $tenantId)
            ->where('role', User::ROLE_INFOPRODUTOR)
            ->first();
        if ($owner === null) {
            $owner = User::query()->where('id', $tenantId)->where('role', User::ROLE_INFOPRODUTOR)->first();
        }
        if ($owner === null) {
            return ['ok' => false, 'skipped' => true, 'reason' => 'no_owner'];
        }

        $result = app(ContractPayoutService::class)->send($withdrawal->fresh(), $owner, $slug);

        if ($result['skipped'] ?? false) {
            return ['ok' => false, 'skipped' => true, 'reason' => (string) ($result['reason'] ?? 'skipped')];
        }

        $prev = is_array($withdrawal->payout_meta) ? $withdrawal->payout_meta : [];

        if ($result['ok'] ?? false) {
            $withdrawal->update([
                'payout_manual' => false,
                'payout_provider' => $slug,
                'payout_external_id' => $result['transaction_id'] ?? null,
                'payout_meta' => array_merge($prev, array_filter([
                    'api_status' => 'pending',
                    'requested_at' => now()->toIso8601String(),
                    'auto' => $auto,
                ])),
            ]);

            ReconcileContractPayoutWithdrawalJob::dispatch($withdrawal->fresh()->id)
                ->delay(now()->addSeconds(90));

            return ['ok' => true, 'pending' => true];
        }

        $withdrawal->update([
            'payout_provider' => $slug,
            'payout_meta' => array_merge($prev, [
                'last_error' => $result['error'] ?? 'Erro desconhecido',
                'last_attempt_at' => now()->toIso8601String(),
                'auto' => $auto,
            ]),
        ]);

        return [
            'ok' => false,
            'skipped' => false,
            'error' => $result['error'] ?? 'Falha ao enviar o saque via PIX.',
        ];
    }
```

Adicione no topo `use App\Jobs\ReconcileContractPayoutWithdrawalJob;` e `use App\Services\Payout\ContractPayoutService;`. O arquivo já importa `PlatformPayoutGateway` e `User`.

- [ ] **Passo 6: Rodar e confirmar que passa**

Rode `php artisan test --filter=ContractPayoutFlowTest`.
Esperado: PASS (7 testes).

- [ ] **Passo 7: Commit**

```bash
git add app/Services/Payout/ContractPayoutService.php app/Jobs/ReconcileContractPayoutWithdrawalJob.php app/Services/WithdrawalAutoPayoutService.php tests/Feature/ContractPayoutFlowTest.php
git commit -m "Send and reconcile automatic withdrawals through SupportsPayout gateways with one shared service and job instead of one per provider."
```

### Tarefa 9: Comando de reconciliação em lote e agendamento

**Arquivos:**
- Criar: `app/Console/Commands/ReconcileContractPayoutWithdrawalsCommand.php`
- Modificar: `bootstrap/app.php` (agendamento)
- Modificar: `tests/Feature/ContractPayoutFlowTest.php` (teste novo)

**Interfaces:**
- Consome: `PlatformPayoutGateway::contractSlugs()` e `ReconcileContractPayoutWithdrawalJob` (tarefas 7 e 8).
- Produz: o comando `withdrawals:reconcile-contract-payouts {--limit=80} {--min-age-minutes=1}`.

- [ ] **Passo 1: Escrever o teste que falha**

Adicione em `ContractPayoutFlowTest`:

```php
    public function test_reconcile_command_marks_contract_withdrawals_paid(): void
    {
        $this->connectGlobalGatewayCredential('fakepay');
        FakePayoutGatewayDriver::$payoutStatus = 'paid';
        $w = $this->makeWithdrawal(100, [], [
            'status' => 'processing',
            'payout_provider' => 'fakepay',
            'payout_external_id' => 'fp-1',
        ]);

        $this->artisan('withdrawals:reconcile-contract-payouts', ['--min-age-minutes' => 0])->assertSuccessful();

        $this->assertSame('paid', $w->fresh()->status);
    }
```

- [ ] **Passo 2: Rodar e confirmar que falha**

Rode `php artisan test --filter=test_reconcile_command_marks_contract_withdrawals_paid`.
Esperado: FAIL ("The command "withdrawals:reconcile-contract-payouts" does not exist").

- [ ] **Passo 3: Implementar o comando**

`app/Console/Commands/ReconcileContractPayoutWithdrawalsCommand.php`:

```php
<?php

namespace App\Console\Commands;

use App\Jobs\ReconcileContractPayoutWithdrawalJob;
use App\Models\Withdrawal;
use App\Services\Payout\PlatformPayoutGateway;
use Illuminate\Console\Command;
use Illuminate\Support\Facades\Schema;

class ReconcileContractPayoutWithdrawalsCommand extends Command
{
    protected $signature = 'withdrawals:reconcile-contract-payouts
                            {--limit=80 : Máximo de saques para checar por execução}
                            {--min-age-minutes=1 : Ignorar registros atualizados há menos de X minutos}';

    protected $description = 'Reconcilia saques PIX de gateways que implementam SupportsPayout (sem comando próprio).';

    public function handle(): int
    {
        if (! Schema::hasTable('withdrawals')) {
            return self::SUCCESS;
        }

        $slugs = PlatformPayoutGateway::contractSlugs();
        if ($slugs === []) {
            return self::SUCCESS;
        }

        $limit = max(1, (int) $this->option('limit'));
        $minAge = max(0, (int) $this->option('min-age-minutes'));

        $query = Withdrawal::query()
            ->whereIn('payout_provider', $slugs)
            ->whereIn('status', ['pending', 'processing'])
            ->orderBy('id');

        if ($minAge > 0) {
            $query->where('updated_at', '<=', now()->subMinutes($minAge));
        }

        foreach ($query->limit($limit)->pluck('id') as $id) {
            ReconcileContractPayoutWithdrawalJob::dispatchSync((int) $id);
        }

        return self::SUCCESS;
    }
}
```

Em `bootstrap/app.php`, logo depois da linha `$schedule->command('withdrawals:reconcile-okto --limit=80 --min-age-minutes=0')->everyMinute();`, adicione:

```php
        $schedule->command('withdrawals:reconcile-contract-payouts --limit=80 --min-age-minutes=0')->everyMinute()->withoutOverlapping(10);
```

- [ ] **Passo 4: Rodar e confirmar que passa**

Rode `php artisan test --filter=ContractPayoutFlowTest`.
Esperado: PASS (8 testes).

- [ ] **Passo 5: Commit**

```bash
git add app/Console/Commands/ReconcileContractPayoutWithdrawalsCommand.php bootstrap/app.php tests/Feature/ContractPayoutFlowTest.php
git commit -m "Reconcile pending SupportsPayout withdrawals every minute with a single scheduled command."
```

### Tarefa 10: Aprovação pelo admin e preferência de payout

**Arquivos:**
- Modificar: `app/Http/Controllers/Platform/FinancialController.php` (`approveWithdrawal`, `updatePayoutGatewayPreference`, `index`)
- Modificar: `resources/js/Pages/Platform/Financial/Index.vue` (prop, `watch` e `<select>`)
- Teste: `tests/Feature/ContractPayoutAdminTest.php`

**Interfaces:**
- Consome:
  - `WithdrawalAutoPayoutService::attemptViaContract()` (tarefa 8);
  - `redirectAfterAdminPayoutAttempt()` (tarefa 1);
  - `PlatformPayoutGateway::supportsPayoutContract()`, `contractSlugs()` e `contractOptions()` (tarefa 7).
- Produz: a prop de página `payout_gateway_contract_options: {slug, name}[]`.

- [ ] **Passo 1: Escrever os testes que falham**

`tests/Feature/ContractPayoutAdminTest.php`:

```php
<?php

namespace Tests\Feature;

use App\Http\Middleware\EnsureInstalled;
use App\Http\Middleware\EnsureStackerLicense;
use App\Jobs\ReconcileContractPayoutWithdrawalJob;
use App\Models\Setting;
use App\Models\User;
use App\Models\Withdrawal;
use App\Services\Withdrawal\WithdrawalPolicyService;
use Illuminate\Foundation\Http\Middleware\ValidateCsrfToken;
use Illuminate\Support\Facades\Queue;
use Tests\Concerns\RegistersFakeGateways;
use Tests\Fixtures\Gateways\FakePayoutGatewayDriver;
use Tests\TestCase;

class ContractPayoutAdminTest extends TestCase
{
    use RegistersFakeGateways;

    private User $admin;

    protected function setUp(): void
    {
        parent::setUp();
        $this->withoutMiddleware([EnsureInstalled::class, EnsureStackerLicense::class, ValidateCsrfToken::class]);
        FakePayoutGatewayDriver::reset();
        Setting::set('platform_payout_gateway', null, null);
        $this->registerFakeGateway('fakepay', FakePayoutGatewayDriver::class);
        $this->connectGlobalGatewayCredential('fakepay');
        Queue::fake();

        WithdrawalPolicyService::setManualApprovalPin('1234');
        $this->admin = User::factory()->create(['role' => User::ROLE_PLATFORM_ADMIN, 'tenant_id' => null]);
    }

    public function test_admin_approval_sends_payout_through_contract_gateway(): void
    {
        $seller = User::factory()->create(['role' => User::ROLE_INFOPRODUTOR]);
        $seller->forceFill([
            'tenant_id' => $seller->id,
            'payout_settings' => ['payout_pix_key' => '11119854997', 'payout_pix_key_type' => 'cpf'],
        ])->save();
        $withdrawal = Withdrawal::query()->create([
            'tenant_id' => $seller->id,
            'user_id' => $seller->id,
            'amount' => 100,
            'fee_amount' => 0,
            'net_amount' => 100,
            'bucket' => 'pix',
            'status' => 'pending',
            'currency' => 'BRL',
        ]);

        $response = $this->actingAs($this->admin)->post(
            route('plataforma.financeiro.saques.approve', $withdrawal),
            ['manual_approval_pin' => '1234']
        );

        $response->assertRedirect(route('plataforma.saques.index'));
        $response->assertSessionHas('success');
        $fresh = $withdrawal->fresh();
        $this->assertSame('fakepay', $fresh->payout_provider);
        $this->assertSame('fake-payout-1', $fresh->payout_external_id);
        $this->assertArrayNotHasKey('auto', is_array($fresh->payout_meta) ? $fresh->payout_meta : []);
        Queue::assertPushed(ReconcileContractPayoutWithdrawalJob::class);
    }

    public function test_contract_gateway_is_accepted_as_payout_preference(): void
    {
        $this->actingAs($this->admin)
            ->putJson(route('plataforma.financeiro.payout-gateway.update'), ['preference' => 'fakepay'])
            ->assertOk()
            ->assertJsonPath('payout_gateway_preference', 'fakepay');
    }

    public function test_unknown_payout_preference_is_rejected(): void
    {
        $this->actingAs($this->admin)
            ->putJson(route('plataforma.financeiro.payout-gateway.update'), ['preference' => 'nope'])
            ->assertStatus(422);
    }
}
```

- [ ] **Passo 2: Rodar e confirmar que falha**

Rode `php artisan test --filter=ContractPayoutAdminTest`.
Esperado: FAIL. A aprovação cai em "Gateway de payout não suportado.", e a preferência `fakepay` é recusada com 422.

- [ ] **Passo 3: Implementar no controller**

Em `approveWithdrawal`, imediatamente **antes** do bloco final:

```php
        MerchantWithdrawalService::releasePayoutApproval($withdrawal);

        return redirect()->route('plataforma.saques.index')
            ->with('error', 'Gateway de payout não suportado.');
```

insira:

```php
        if (PlatformPayoutGateway::supportsPayoutContract($slug)) {
            $label = (string) (GatewayRegistry::get($slug)['name'] ?? $slug);
            $result = app(WithdrawalAutoPayoutService::class)->attemptViaContract($withdrawal->fresh(), $slug, auto: false);

            return $this->redirectAfterAdminPayoutAttempt($request, $withdrawal, $slug, $label, $result);
        }
```

Em `updatePayoutGatewayPreference`, troque a regra de validação por:

```php
            'preference' => ['required', 'string', Rule::in(array_merge(
                ['auto', 'cajupay', 'woovi', 'bspay', 'versell', 'xflow', 'okto', 'onlyup'],
                PlatformPayoutGateway::contractSlugs()
            ))],
```

Em `index`, no array do `Inertia::render`, logo depois de `'payout_gateway_active' => PlatformPayoutGateway::activeSlug(),`, adicione:

```php
            'payout_gateway_contract_options' => PlatformPayoutGateway::contractOptions(),
```

Garanta os imports `use Illuminate\Validation\Rule;` e `use App\Services\WithdrawalAutoPayoutService;`. O segundo vem da tarefa 1; se o PR 1 não estiver na base, adicione-o aqui. `GatewayRegistry` e `PlatformPayoutGateway` já são importados.

- [ ] **Passo 4: Rodar e confirmar que passa**

Rode `php artisan test --filter=ContractPayoutAdminTest`.
Esperado: PASS (3 testes).

- [ ] **Passo 5: Frontend — opções extras no seletor**

Em `resources/js/Pages/Platform/Financial/Index.vue`:

Nas props, logo depois de `payout_gateway_active`, adicione:

```js
    /** Gateways com payout por contrato (SupportsPayout), além dos legados. */
    payout_gateway_contract_options: { type: Array, default: () => [] },
```

No `watch` de `props.payout_gateway_preference`, troque o corpo por:

```js
    (v) => {
        const contractSlugs = (props.payout_gateway_contract_options || []).map((o) => o.slug);
        payoutPref.value =
            v === 'cajupay' || v === 'woovi' || v === 'bspay' || v === 'versell' || v === 'xflow' || v === 'okto' || v === 'onlyup' || contractSlugs.includes(v)
                ? v
                : 'auto';
    },
```

No `<select id="payout-pref-select">`, logo depois de `<option value="onlyup">Forçar OnlyUp</option>`, adicione:

```vue
                                <option
                                    v-for="opt in payout_gateway_contract_options"
                                    :key="opt.slug"
                                    :value="opt.slug"
                                >
                                    Forçar {{ opt.name }}
                                </option>
```

- [ ] **Passo 6: Build**

Rode `npm run build`.
Esperado: sem erros.

- [ ] **Passo 7: Commit**

```bash
git add app/Http/Controllers/Platform/FinancialController.php resources/js/Pages/Platform/Financial/Index.vue tests/Feature/ContractPayoutAdminTest.php
git commit -m "Let admins approve withdrawals and choose SupportsPayout gateways as the payout preference."
```

### Tarefa 11: Destino PIX `pix_key_only` para provedores por contrato

**Arquivos:**
- Modificar: `app/Services/Payout/PayoutDestinationValidator.php` (duas condições)
- Modificar: `app/Http/Controllers/SellerFinancialController.php` (`match` de `$payoutPixSetup` e duas condições)
- Teste: `tests/Feature/ContractPayoutDestinationTest.php`

**Interfaces:**
- Consome: `PlatformPayoutGateway::supportsPayoutContract()` (tarefa 7).

- [ ] **Passo 1: Escrever os testes que falham**

`tests/Feature/ContractPayoutDestinationTest.php`:

```php
<?php

namespace Tests\Feature;

use App\Http\Middleware\EnsureInstalled;
use App\Http\Middleware\EnsureStackerLicense;
use App\Models\Setting;
use App\Models\User;
use App\Services\Payout\PayoutDestinationValidator;
use Illuminate\Foundation\Http\Middleware\ValidateCsrfToken;
use Tests\Concerns\RegistersFakeGateways;
use Tests\Fixtures\Gateways\FakePayoutGatewayDriver;
use Tests\TestCase;

class ContractPayoutDestinationTest extends TestCase
{
    use RegistersFakeGateways;

    protected function setUp(): void
    {
        parent::setUp();
        $this->withoutMiddleware([EnsureInstalled::class, EnsureStackerLicense::class, ValidateCsrfToken::class]);
        FakePayoutGatewayDriver::reset();
        Setting::set('platform_payout_gateway', null, null);
        $this->registerFakeGateway('fakepay', FakePayoutGatewayDriver::class);
        $this->connectGlobalGatewayCredential('fakepay');
    }

    private function approvedSeller(array $payoutSettings = []): User
    {
        $seller = User::factory()->create(['role' => User::ROLE_INFOPRODUTOR]);
        $seller->forceFill([
            'tenant_id' => $seller->id,
            'kyc_status' => User::KYC_APPROVED,
            'account_status' => 'approved',
            'payout_settings' => $payoutSettings,
        ])->save();

        return $seller;
    }

    public function test_validator_accepts_pix_key_only_for_contract_gateway(): void
    {
        $result = PayoutDestinationValidator::validateForUpdate(
            ['pix_key' => 'destino@pix.com', 'pix_key_type' => 'email'],
            'fakepay'
        );

        $this->assertTrue($result['ok']);
        $this->assertSame('email', $result['pix_key_type']);
    }

    public function test_seller_ready_for_withdrawal_with_pix_key(): void
    {
        $this->assertFalse(PayoutDestinationValidator::assertReadyForWithdrawal($this->approvedSeller(), 'fakepay')['ok']);

        $ready = $this->approvedSeller(['payout_pix_key' => 'destino@pix.com', 'payout_pix_key_type' => 'email']);
        $this->assertTrue(PayoutDestinationValidator::assertReadyForWithdrawal($ready, 'fakepay')['ok']);
    }

    public function test_seller_can_save_pix_key_when_contract_gateway_is_active(): void
    {
        $seller = $this->approvedSeller();

        $response = $this->actingAs($seller)->post(route('financeiro.seller.pix-saque'), [
            'pix_key' => 'destino@pix.com',
            'pix_key_type' => 'email',
        ]);

        $response->assertRedirect(route('financeiro.seller.index'));
        $response->assertSessionHas('success');
        $this->assertSame('destino@pix.com', $seller->fresh()->payout_settings['payout_pix_key'] ?? null);
    }
}
```

- [ ] **Passo 2: Rodar e confirmar que falha**

Rode `php artisan test --filter=ContractPayoutDestinationTest`.
Esperado: FAIL. O validador responde "Saque automático não configurado na plataforma.", e o controller responde "Gateway de payout não suportado.".

- [ ] **Passo 3: Implementar no validador**

Em `PayoutDestinationValidator::validateForUpdate`, troque:

```php
        if (in_array($slug, ['spacepag', 'woovi', 'bspay', 'onlyup', 'xflow', 'okto'], true)) {
```

por:

```php
        if (in_array($slug, ['spacepag', 'woovi', 'bspay', 'onlyup', 'xflow', 'okto'], true)
            || PlatformPayoutGateway::supportsPayoutContract($slug)) {
```

Faça **a mesma troca** em `assertReadyForWithdrawal`, que tem a mesma linha. `PlatformPayoutGateway` está no mesmo namespace (`App\Services\Payout`), então não precisa de import.

- [ ] **Passo 4: Implementar no `SellerFinancialController`**

No `match ($payoutGateway)` que calcula `$payoutPixSetup`, troque `default => null,` por:

```php
            default => PlatformPayoutGateway::supportsPayoutContract($payoutGateway) ? 'pix_key_only' : null,
```

A linha `if (in_array($slug, ['woovi', 'bspay', 'onlyup', 'xflow', 'okto'], true)) {` aparece **duas vezes**: em `storePayoutPixKey` e em `storeWithdrawal`. Troque as duas por:

```php
        if (in_array($slug, ['woovi', 'bspay', 'onlyup', 'xflow', 'okto'], true)
            || PlatformPayoutGateway::supportsPayoutContract($slug)) {
```

`PlatformPayoutGateway` já é importado no controller.

- [ ] **Passo 5: Rodar e confirmar que passa**

Rode `php artisan test --filter=ContractPayoutDestinationTest`.
Esperado: PASS (3 testes). Se `test_seller_can_save_pix_key_when_contract_gateway_is_active` falhar por causa de um middleware do painel do vendedor (KYC, equipe, termos), use o mesmo setup de vendedor dos testes existentes que postam em rotas do painel (por exemplo `tests/Feature/CustomerPanelAccessTest.php`). Ajuste **só o teste**.

- [ ] **Passo 6: Commit**

```bash
git add app/Services/Payout/PayoutDestinationValidator.php app/Http/Controllers/SellerFinancialController.php tests/Feature/ContractPayoutDestinationTest.php
git commit -m "Accept PIX-key-only payout destinations for SupportsPayout gateways in the seller panel, API and withdrawal checks."
```

### Tarefa 12: Documentação do payout por contrato e verificação final do PR 4

**Arquivos:**
- Modificar: `app/Gateways/README.md`

- [ ] **Passo 1: Documentar**

Na seção "Capacidades por contrato" de `app/Gateways/README.md` (criada na tarefa 6; se o PR 3 não estiver na base, crie a seção com o mesmo título), acrescente esta linha à tabela:

```markdown
| `Contracts\SupportsPayout` | PIX OUT: entra depois dos gateways legados na ordem automática de payout, pode ser forçado como preferência, aprovado pelo admin e reconciliado por `withdrawals:reconcile-contract-payouts`. Destino sempre "só chave PIX". Taxas e mínimo: `{slug}_payout_min_brl`, `{slug}_admin_fee_payout_brl` / `_percent` nas credenciais |
```

E depois da tabela:

```markdown
Webhook de transferência: o controller do gateway localiza o saque por `payout_provider = <slug>` + `payout_external_id` e chama `MerchantWithdrawalService::markPaid()` / `markFailed()`. Sem webhook, a reconciliação agendada confirma em até ~1 minuto.
```

- [ ] **Passo 2: Verificação final**

```bash
composer test
npm run build
git diff upstream/main...HEAD --stat
```

Esperado:
- tudo verde;
- build sem erro;
- o `--stat` não lista nenhum arquivo em `app/Gateways/<Provedor>/`, `app/Services/<Provedor>/`, `app/Jobs/Reconcile<Provedor>*` ou `app/Http/Controllers/Webhooks/`.

- [ ] **Passo 3: Commit**

```bash
git add app/Gateways/README.md
git commit -m "Document the SupportsPayout contract for gateway authors."
```

---

## Depois de cada PR (fora deste plano, com a confirmação do usuário)

1. `git push -u origin contrib/<nome>` e abrir o PR para `stacker-builders/stacker-gateway:main`. A descrição deve citar a spec, o princípio "aditivo, nenhum provedor existente alterado" e a Delfinance como primeiro consumidor.
2. Trazer para o fork: `git switch -c integration/<nome> origin/main` e depois `git cherry-pick -x <shas>`. Rodar `composer test` e abrir o PR para a `main` do fork.
