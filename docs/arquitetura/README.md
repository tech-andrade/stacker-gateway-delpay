# Arquitetura — documentação do fork DelPay

Documentação técnica de como o Stacker Gateway funciona por dentro, escrita a partir da leitura do código. Ela serve para o time da Delfinance e para quem for estender a plataforma.

| Documento | Conteúdo |
|---|---|
| [provedores-de-pagamento.md](provedores-de-pagamento.md) | Como os provedores (gateways) funcionam: contrato, registro, escolha com fallback, credenciais, webhooks, PIX OUT, estorno, MED, Pix Automático, "categorias" (PSP / Adquirente / Open Finance) e o checklist completo para criar um provedor novo |
| [kyc.md](kyc.md) | Como o KYC de vendedores funciona hoje (100% manual) e onde ele bloqueia a operação |
| [proposta-delfinance.md](proposta-delfinance.md) | **Proposta**, ainda não implementada: Delfinance como PSP, a adquirência depois e uma nova categoria "Provedores KYC" |

Specs e planos de implementação ficam em [docs/specs/](../specs/):

| Spec | Conteúdo |
|---|---|
| [2026-10-15-extensoes-de-provedores](../specs/2026-10-15-extensoes-de-provedores/spec.md) | Fase 0 (pontos de extensão aditivos, em 4 PRs para o upstream) e fase 1 (Delfinance PSP). Inclui o [plano detalhado da fase 0](../specs/2026-10-15-extensoes-de-provedores/plano-fase-0.md) |

As referências a arquivos e linhas valem para o código de outubro de 2026 (versão 2.3.x do upstream). Depois de merges do upstream os números de linha podem mudar; nesse caso, procure pelo nome da classe ou do método.
