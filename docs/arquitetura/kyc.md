# KYC de vendedores — como funciona hoje

**Resumo:** o KYC é **100% manual**. O vendedor envia documentos, um admin da plataforma analisa e aprova ou rejeita (com TOTP). Nenhum serviço externo valida CPF, documento ou selfie. A única consulta externa é a de CNPJ na BrasilAPI, que é apenas informativa. **Não existe abstração de "provedor de KYC".**

## 1. Dados

**`users`** ([app/Models/User.php](../../app/Models/User.php)):

- **Status do KYC (`kyc_status`):** `not_submitted` → `pending_review` → `approved` | `rejected`.
- **Status da conta (`account_status`):** é separado, com os valores `pending`, `approved`, `suspended`, `blocked` e `rejected`. Para operar, o vendedor precisa de **`kyc_status = approved` E `account_status = approved`** (`isMerchantOperationallyApproved()`).
- **Dados de cadastro:**
  - `person_type` (`pf`/`pj`);
  - `document` (CPF ou CNPJ);
  - `birth_date`;
  - `company_name`, `trade_name`, `legal_representative_cpf`;
  - endereço completo;
  - `monthly_revenue_range`.
- **Controle do KYC:**
  - `kyc_rejection_reason`, `kyc_reviewed_at`, `kyc_reviewed_by`, `kyc_needs_document_review`;
  - `identity_document_type` (`rg`/`cnh`/`passport`);
  - `company_legal_nature` (MEI ou outra);
  - `kyc_requirements_version`.
- **JSON:** `cnpj_lookup` (snapshot da BrasilAPI) e `pj_conversion` (migração de PF para PJ).

**`kyc_documents`** ([app/Models/KycDocument.php](../../app/Models/KycDocument.php)):

- Tipos (`kind`): `rg_front`, `rg_back`, `selfie_with_document`, `address_proof`, `company_address_proof`, `social_contract` e `ccmei`.
- Um reenvio não apaga o documento anterior: o antigo é marcado com `superseded_at`.
- Os arquivos ficam no disco `local`, em `kyc/{userId}/{uuid}.{ext}`, e são acessados pelo `public_token`.

**Documentos exigidos** ([app/Support/KycRequiredDocuments.php](../../app/Support/KycRequiredDocuments.php)):

- **PF:** documento de identidade (RG frente e verso, ou CNH/passaporte só frente), comprovante de endereço e selfie com o documento.
- **PJ:** documento de identidade, comprovante de endereço da empresa e CCMEI (para MEI) ou contrato social (para os demais).
- Cada exigência pode ser ligada ou desligada no setting `kyc_document_requirements` (tela Configurações → aba KYC).

## 2. Fluxo do vendedor

1. **Cadastro** (`InfoprodutorRegistrationController`):
   - valida CPF/CNPJ só pelo dígito verificador e exige idade de 18 anos ou mais;
   - para PJ, consulta a BrasilAPI;
   - cria a conta com `kyc_status = not_submitted` e `account_status = pending`;
   - dispara `SellerRegistered`.
2. **O painel fica bloqueado:** o middleware `EnsureSellerPanel` redireciona tudo para `/financeiro?tab=seus-dados` até o KYC e a conta serem aprovados.
3. **Envio dos documentos** (`SellerKycController` + `resources/js/components/kyc/KycDocumentsForm.vue`):
   - `POST /kyc/preferences` grava o tipo de documento e a natureza jurídica;
   - `POST /kyc/document` envia um arquivo por vez;
   - `POST /kyc/finalize` envia para análise.
4. **Finalização:**
   - `markPendingReview()` coloca o status em `pending_review`;
   - manda e-mail para os admins (`kyc_notification_emails`) e grava no log de atividade.
   - **Não existe evento `KycSubmitted`.**

## 3. Fluxo do admin

`/plataforma/verificacoes-kyc` ([Platform/KycVerificationsController](../../app/Http/Controllers/Platform/KycVerificationsController.php) + `resources/js/Pages/Platform/Kyc/`):

- Lista com filtros (pendentes, rejeitados, não enviados, precisam revisar e todos).
- Detalhe com:
  - os dados do vendedor;
  - todos os documentos, inclusive os substituídos;
  - o painel de comparação com o CNPJ oficial, com alertas de situação irregular e de razão social diferente.
- **Aprovar** (exige `pending_review`, TOTP e todos os documentos):
  - grava `kyc_status = approved` **e `account_status = approved`**;
  - registra na auditoria;
  - manda o e-mail `KycApprovedMail`;
  - dispara o evento **`KycApproved`**.
- **Rejeitar** (motivo obrigatório):
  - grava `kyc_status = rejected`;
  - manda o e-mail `KycRejectedMail`;
  - dispara o evento **`KycRejected`**.
- Hoje o único listener desses eventos é o `PlatformWhatsappEventSubscriber`, que manda os templates de WhatsApp.

> A lógica de aprovação e rejeição está **dentro do controller**. Para automatizar o KYC, o primeiro passo é extraí-la para um service reutilizável.

## 4. O que o KYC bloqueia

O guard central fica em [app/Services/MerchantOperationalGuard.php](../../app/Services/MerchantOperationalGuard.php).

| Onde | O que exige |
|---|---|
| Painel do vendedor inteiro (`EnsureSellerPanel`) | KYC e conta aprovados |
| API de pagamentos (`/api/v1` payments, PIX, checkout sessions) e PixGO | `assertCanAcceptPayments` |
| Saques: painel, API e `BalanceController` | `assertCanRequestWithdrawal` |
| Cadastro de chave PIX de saque | KYC aprovado |
| Admin marcar a conta como aprovada (`UsersController`) | KYC já aprovado |
| Conversão PF → PJ | KYC aprovado |

> O **checkout web público** (`CheckoutController`) **não** checa o KYC. Hoje, a proteção ali é indireta: o vendedor sem KYC aprovado não consegue entrar no painel para criar produtos.

## 5. Verificação externa existente

- **CNPJ:**
  - `app/Services/Cnpj/BrasilApiCnpjClient.php` é uma classe concreta, sem interface;
  - os resultados ficam em cache (24 h para sucesso, 60 s para falha);
  - `app/Support/CnpjLookup.php` gera os alertas mostrados ao admin.
- **CPF:** só o dígito verificador. Não há consulta à Receita, OCR nem face match.
- **Provedores de pagamento:** nenhum faz KYC do vendedor na plataforma.

## 6. Settings relacionados

| Setting | Onde edita | O que controla |
|---|---|---|
| `kyc_document_requirements` | Configurações → KYC | Quais documentos são exigidos |
| `kyc_notification_emails` | Configurações | Quem recebe o aviso de KYC enviado |
| `KYC_UPLOAD_MAX_KB` (env) | `config/getfy_uploads.php` | Tamanho máximo do arquivo (padrão 20 MB) |

**Não existe** auto-aprovação nem escolha de provedor.
