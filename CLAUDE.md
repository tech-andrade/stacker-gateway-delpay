# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## O que é este projeto

Stacker Gateway (derivado do open-source "Getfy"): plataforma Laravel 12 + Inertia/Vue 3 para checkout de infoprodutos, orquestração de gateways de pagamento, carteira e saques de vendedores, áreas de membros (cursos), afiliados/coprodução e uma API pública de pagamentos (`/api/v1`). Textos da UI, nomes de rotas e boa parte do código/comentários estão em português (`produtos`, `vendas`, `plataforma`, `alunos`, `financeiro`…).

## Este repositório é um FORK

Este repo (`origin` = `tech-andrade/stacker-gateway-delpay`) é um fork do projeto open-source **`stacker-builders/stacker-gateway`** (remote `upstream`, branch principal `main`). Ele recebe **customizações próprias da DelPay**, mas continua puxando correções do fornecedor e pode **devolver melhorias para a comunidade**.

Toda mudança se encaixa em um de três fluxos. **Identifique o fluxo antes de criar a branch.** Se o pedido for ambíguo (ex.: uma correção que serve tanto para a DelPay quanto para o upstream), pergunte.

| Fluxo | Quando o usuário diz… | Branch base | Nome da branch | PR vai para |
|---|---|---|---|---|
| **1. Customizar** | "customizar", "criar feature", "ajustar para a DelPay" | `origin/main` | `feature/<nome>` ou `fix/<nome>` | `main` do fork |
| **2. Trazer do fornecedor** | "trazer", "sincronizar", "atualizar com o upstream", "puxar a correção X" | `origin/main` | `integration/upstream-AAAA-MM-DD` ou `integration/<nome-da-correção>` | `main` do fork |
| **3. Contribuir** | "contribuir", "mandar para a comunidade", "abrir PR no stacker-gateway" | `upstream/main` | `contrib/<nome>` | `stacker-builders/stacker-gateway:main` |

Regras gerais:
- Nunca commitar direto na `main`, nunca fazer rebase da `main` e nunca usar `push --force` na `main`. Tudo entra por PR.
- Antes de começar qualquer fluxo, garantir que os remotes existem (`git remote -v`). Se faltar o `upstream`, rodar `git remote add upstream https://github.com/stacker-builders/stacker-gateway.git`.
- Push e abertura de PR são ações externas. Confirme com o usuário antes, principalmente no fluxo 3, que publica código num repositório público de terceiros.

### Fluxo 1 — Customizar (código só da DelPay)

```bash
git fetch origin
git switch main && git pull origin main
git switch -c feature/<nome>
# ...desenvolver, commitar...
composer test && npm run build
git push -u origin feature/<nome>       # PR: feature/<nome> → main (fork)
```

Como escrever a customização para que os merges futuros do upstream não virem dor de cabeça:
- **Prefira arquivos novos** a editar arquivos do upstream: novos services, um listener novo para `OrderCompleted` (registrado numa linha), controllers e páginas Vue novos, um gateway novo via `GatewayRegistry`, ou um plugin em `plugins/`.
- **Arquivos centrais do upstream** que mudam com frequência (`routes/web.php`, `bootstrap/app.php`, `app/Providers/AppServiceProvider.php`, `app/Services/PaymentService.php`, `app/Http/Controllers/CheckoutController.php`, `config/gateways.php`) recebem só a menor alteração possível. Marque a alteração com um comentário `// [delpay] <motivo>` (`{{-- [delpay] --}}` / `<!-- [delpay] -->` em templates). Para listar todas as customizações espalhadas: `git grep -n "\[delpay\]"`.
- Migrations próprias recebem nomes que deixem a origem clara (ex.: `..._delpay_add_x_to_orders.php`) e nunca alteram migrations que vieram do upstream.
- Se, no meio do trabalho, aparecer uma correção que **também vale para o upstream**, separe-a: faça essa parte pelo fluxo 3 e traga para o fork por `cherry-pick` (ou espere o merge do upstream), em vez de misturá-la com o commit da customização.

### Fluxo 2 — Trazer atualizações do fornecedor

**2a. Sincronizar tudo** (recomendado periodicamente ou quando sair uma release nova do upstream):

```bash
git fetch upstream && git fetch origin
git switch main && git pull origin main
git log --oneline main..upstream/main           # o que vai entrar (vazio = já está em dia)
git switch -c integration/upstream-AAAA-MM-DD
git merge upstream/main                         # merge, NÃO rebase: preserva o histórico do upstream
```

**2b. Trazer só uma correção específica** (ex.: um PR do fornecedor que ainda não entrou na `main` dele, ou um hotfix urgente):

```bash
git fetch upstream
git log --oneline upstream/main -20                       # ou: git branch -r | grep upstream/fix/
git log --oneline main..upstream/fix/<nome>               # commits da branch do fornecedor
git switch main && git pull origin main
git switch -c integration/<nome-da-correção>
git cherry-pick -x <sha>                                  # -x registra o commit de origem na mensagem
```

Resolução de conflitos (2a e 2b):
- Regra padrão: **adote a versão do upstream e reaplique por cima a customização marcada com `[delpay]`**. Não descarte uma mudança do upstream sem entender o motivo dela (leia o commit com `git show <sha>`).
- `VERSION` e `docs/releases/`: sempre aceitar os do upstream.
- `composer.lock` / `package-lock.json`: aceite a versão do upstream e rode `composer install` / `npm install` para regenerar se o fork tiver dependências extras.
- Depois do merge, confira se o upstream trouxe migrations novas (`git diff --name-only main...HEAD -- database/migrations`) e se elas não colidem com migrations `delpay`.

Validação antes do PR:

```bash
composer test
npm run build
git grep -n "\[delpay\]"        # confirme que as customizações continuam presentes
```

PR: `integration/...` → `main` do fork. Na descrição, liste o intervalo trazido (`git log --oneline main..HEAD`) e os conflitos resolvidos.

### Fluxo 3 — Contribuir de volta para a comunidade

A branch de contribuição **nasce do `upstream/main`** e não pode conter nada da DelPay.

```bash
git fetch upstream
git switch -c contrib/<nome> upstream/main
# Opção A: implementar direto aqui.
# Opção B: portar uma melhoria que já existe no fork (só o commit genérico):
git cherry-pick -x <sha-do-fork>
```

Checklist antes do push:
- `git diff upstream/main...HEAD` não contém `[delpay]`, domínios, credenciais, branding, textos ou regras de negócio da DelPay. Também não contém migrations próprias do fork.
- Commits no estilo do upstream: uma frase em inglês que explique o *porquê* (ex.: `Charge PayPal orders in BRL so Brazilian payee accounts can capture.`). Uma mudança por PR.
- `composer test` e `npm run build` passando. Não altere `VERSION` nem `docs/releases/` (isso é papel do mantenedor).

Publicar e abrir o PR (confirmar com o usuário antes):

```bash
git push -u origin contrib/<nome>
# Abrir: https://github.com/stacker-builders/stacker-gateway/compare/main...<owner>:<repo>:contrib/<nome>
```

O GitHub reconhece `tech-andrade/stacker-gateway-delpay` como fork oficial de `stacker-builders/stacker-gateway`, então o push para `origin` basta para abrir PR no upstream.

**Atenção ao repositório base dos PRs.** Como o repo é um fork, o GitHub sugere por padrão `stacker-builders/stacker-gateway` como base. Isso vale para o link "Compare & pull request", para o banner "had recent pushes" que aparece na página do upstream e para o `gh pr create` sem `--repo`.
- Fluxos 1 e 2: o PR precisa ter como base **`tech-andrade/stacker-gateway-delpay:main`**. Com o `gh`, use `gh pr create --repo tech-andrade/stacker-gateway-delpay --base main`.
- Fluxo 3: só aqui a base é `stacker-builders/stacker-gateway:main`. Com o `gh`, use `gh pr create --repo stacker-builders/stacker-gateway --base main --head tech-andrade:contrib/<nome>`.

Depois que o fornecedor aceitar, a mudança volta para o fork naturalmente pelo fluxo 2. Se o fork já tinha a versão própria, resolva o conflito ficando com a versão do upstream e remova a customização duplicada.

### Observação sobre CI

O `.github/workflows/gateway-release.yml` publica releases na API Stacker usando secrets do fornecedor. No fork ele falha a cada push na `main` que mexa em arquivos de código ou no `VERSION`. Ele precisa ser desativado ou adaptado. Se for editado, marque a edição com `[delpay]`, porque esse arquivo também vem do upstream.

## Comandos

```bash
# Stack local (app + stacker-agent + Postgres + Redis), app em http://localhost:8085
docker compose -f docker-compose.local-db.yml -f docker-compose.dev.yml up --build -d

# Sem Docker (serve + queue:listen + pail + vite em paralelo)
composer dev
npm run dev            # só o Vite
npm run build          # assets de produção em public/build

# Testes (PHPUnit 11; suítes: Unit, Feature)
composer test                                   # config:clear + artisan test
php artisan test --filter=CajuPayPixWebhookTest # uma classe
php artisan test --filter=test_method_name      # um método
php artisan test tests/Feature/Meta             # um diretório

# Formatação
vendor/bin/pint
```

Os testes sempre rodam em SQLite em memória (forçado no `phpunit.xml`). O `tests/TestCase.php` lança exceção de propósito se o `RefreshDatabase` for rodar em qualquer outro banco — nunca aponte os testes para o Postgres local. Produtos usam id UUID no MySQL/Postgres, mas id inteiro no SQLite — crie produtos nos testes com `$this->createTestProduct()`. Os helpers de teste de webhook de gateway ficam em `tests/Concerns/InteractsWithGatewayWebhooks.php`.

## Arquitetura

### Usuários, multi-tenant e painéis
- Banco único, multi-tenant por linha: o `tenant_id` da maioria dos models é o `users.id` do vendedor (infoprodutor) dono do registro; muitos models expõem `scopeForTenant($q, $tenantId)`. Tenant `null` normalmente significa nível plataforma.
- Papéis (`App\Models\User::ROLE_*`): `platform_admin`, `admin`, `infoprodutor` (vendedor), `team` (equipe do vendedor, controlada por `team.permission:<area>.<acao>`), `cliente` (comprador; `aluno` é alias depreciado).
- Painéis distintos, todos em `routes/web.php` (~1350 linhas), separados por grupos de middleware:
  - `/plataforma/*` — admin da plataforma (`platform.admin`, `stacker.license`), controllers em `app/Http/Controllers/Platform/`.
  - Painel do vendedor — `auth, admin.tenant, seller.panel, stacker.license, role:…, audit.log`.
  - Painel do cliente — `customer.panel`.
  - Área de membros — `/m/{slug}` (`member.area.resolve`) ou domínio próprio (`member.area.resolve.by.host`).
  - Checkout público, webhooks de entrada (`/webhooks/...`, isentos de CSRF em `bootstrap/app.php`), instalador (`/install`, `/docker-setup`, `/criar-admin`).
- `routes/api.php` é a API pública para parceiros (middleware `api.application` + `api.scope`, headers `X-Public-Key`/`X-Secret-Key`, `Idempotency-Key`). Controllers em `app/Http/Controllers/Api/V1/`. Templates da documentação/pacote LLM para parceiros ficam em `docs/llm/`.
- O middleware global web (checagens de instalação, branding, idioma, bloqueio de mutações em modo demo, Inertia, headers de segurança, fallback do scheduler) é configurado em `bootstrap/app.php`, que também define o **scheduler** (jobs de reconciliação, liberação de saldo, recuperação de carrinho, backups, limpezas).

### Gateways de pagamento
- Cada gateway é um driver em `app/Gateways/<Nome>/<Nome>Driver.php` implementando `Contracts\GatewayDriver` (`testConnection`, `createPixPayment`, `createCardPayment`, `createBoletoPayment`, `getTransactionStatus`). Gateways são registrados em `config/gateways.php` e/ou em runtime por plugins via `GatewayRegistry::register()`. O passo a passo completo está em `app/Gateways/README.md`.
- Helpers específicos de cada gateway (saques, estornos, resolução de credenciais) ficam em `app/Services/<Gateway>/`; controllers de webhook em `app/Http/Controllers/Webhooks/`. Gateways de plugin usam a rota genérica `POST /webhooks/gateways/{slug}`.
- `App\Services\PaymentService` escolhe o gateway por método de pagamento usando uma lista ordenada de fallback por tenant/produto (`getGatewayOrderForMethod`), então uma cobrança pode cair para o próximo gateway.
- Notas longas de integração de alguns provedores estão em arquivos na raiz (`cajupay-integracao-llm-full.md`, `bspay-…`, `Xflow-…`).

### Ciclo de vida do pedido (transversal)
1. O checkout (`CheckoutController` / `ApiCheckoutController`) cria um `Order` e a cobrança via `PaymentService`.
2. Webhooks dos gateways são registrados (`InboundGatewayWebhook`) e despachados para `App\Jobs\ProcessPaymentWebhook` na fila `webhooks-inbound`. Os comandos agendados `payments:reconcile-*` / `withdrawals:reconcile-*` consultam os gateways como fallback para webhooks perdidos.
3. Na aprovação, dispara o evento `OrderCompleted`. Os efeitos colaterais são listeners registrados no `AppServiceProvider` (crédito na carteira, comissões de afiliado/indicação/coprodutor, acesso a módulos da área de membros, e-mail de acesso, cupons, conquistas, push) mais subscribers de integrações (`app/Listeners/*EventSubscriber.php`: Utmify, Meta CAPI, Webhooks, WhatsApp Uazapi/Evolution, Spedy, Cademi, métricas). Comportamentos novos pós-pagamento devem virar listener, não código inline no fluxo do webhook.
4. Créditos na carteira ficam retidos e são liberados por `settlement:release` (`SettlementReleaseService`); saques dos vendedores passam por `MerchantWithdrawalService` e pela fila `payouts`.

### Filas
Em produção há workers separados por fila (ver `docker-compose.yml`): `payments,default`, `webhooks-outbound,webhooks`, `webhooks-inbound`, `payouts`, `meta-tracking`, `utmify-tracking,metrics-tracking`, `integrax-sms`, `uazapi`, … Ao criar um job, coloque-o na fila nomeada correspondente, senão ele pode nunca ser consumido em produção.

### Plugins
`plugins/<slug>/` (ignorado pelo git, instalado em runtime) com `plugin.json` + `bootstrap.php`, carregado por `App\Providers\PluginServiceProvider` / `App\Plugins\PluginRegistry`. Plugins se conectam pelos eventos de domínio em `app/Events/` (ex.: `CheckoutPageLoading`, `ProductBeforeSave`, `DashboardLoading`) e podem registrar gateways e páginas Vue (`resources/js/PluginPages`).

### Frontend
Inertia + Vue 3 + Pinia + Tailwind 4, build com Vite. Páginas em `resources/js/Pages/<Area>/` espelham os controllers; UI compartilhada em `resources/js/components`, lógica em `resources/js/composables` e `resources/js/lib`. As props compartilhadas do Inertia vêm de `HandleInertiaRequests` (com cache via `InertiaSharedPropsCache`).

### Deploy / releases
- Instalações self-hosted usam Docker (`docker-compose*.yml`, scripts em `docker/`, `install*.sh`/`update*.sh`). `agent/` é um sidecar TypeScript ("stacker-agent") que aplica updates remotos (baixa o zip da release, rebuilda o `app`, roda migrate) e envia métricas para a API Stacker.
- Release: crie `docs/releases/X.Y.Z.md` (formato em `docs/releases/README.md`), atualize `VERSION` e faça push na `main` — o `.github/workflows/gateway-release.yml` gera o zip e publica na API Stacker.
- A checagem de licença (middleware `stacker.license`) fica desativada em dev via `STACKER_LICENSE_DISABLED=1`.
