# Pesquisa: Assistente bancário Cora + split

Documento completo:

- [`avaliacao-cora-split-contabil.md`](./avaliacao-cora-split-contabil.md)

## Resumo executivo

- **Cora** é adequada para um assistente interno de cobranças/pagamentos (boleto, PIX, webhooks, e-mail próprio, NFS-e).
- **Cora não tem split nativo.**
- **Split caseiro** (receber tudo e repassar) é o pior desenho para cliente Lucro Real e cria zona cinza regulatória.
- **Melhor modelo:** prestador principal + subcontratação (1 NFS-e + 1 boleto ao cliente; parceiro nota para você).
- Alternativa limpa: duas cobranças / dois contratos.
- Gateway com split (Asaas/Iugu/etc.) só se o Modelo A/B não servir.
