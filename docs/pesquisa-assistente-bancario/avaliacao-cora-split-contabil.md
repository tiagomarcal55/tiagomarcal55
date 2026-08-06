# Avaliação: Assistente bancário Cora + split de pagamentos (uso interno)

**Escopo:** empresa pequena, solução **não comercial** (não vender split a terceiros), sem abrir fintech.  
**Objetivo:** cobrar clientes PJ (boleto/PIX), auxiliar financeiro, e-mails com modelos, e avaliar se dá para ter “split” com parceiro sem dor de cabeça contábil para o cliente (especialmente Lucro Real).  
**Aviso:** isto é pesquisa técnico-operacional e de viabilidade. **Não substitui parecer contábil/jurídico** do seu contador e advogado para o seu CNAE, município e contratos reais.

---

## 1. Veredito em uma página

| Pergunta | Resposta curta |
|---|---|
| A Cora serve para um assistente bancário próprio? | **Sim.** Integração Direta cobre boleto, boleto+PIX, carnê, QR Pix, extrato, pagamentos, transferências, webhooks e NFS-e. |
| A Cora tem split nativo? | **Não.** Não há endpoint/recurso de split marketplace na documentação pública. |
| Posso “criar meu próprio split” recebendo tudo e repassando ao parceiro? | **Tecnicamente sim; regulatoriamente e contabilmente é o pior caminho** se o dinheiro do parceiro passa pela sua conta como se fosse intermediário. |
| Cliente Lucro Real aceita bem 1 boleto + 2 NFS-e (sua + parceiro)? | **Na prática, não.** Contabilidade exige amarrar despesa × documento fiscal × pagamento. Esse desenho gera fricção e o cliente tende a pedir **uma única nota**. |
| Melhor modelo para sua empresa pequena? | **Modelo A — Prestador principal + subcontratação** (1 NF + 1 cobrança para o cliente; parceiro nota para você). Ou **Modelo B — duas cobranças** se ambos forem contratados pelo cliente. |
| Quando usar gateway com split (Asaas/Iugu/Pagar.me)? | Só se a operação for recorrente com vários recebedores **e** o cliente aceitar a documentação (ainda assim a dor contábil do tomador não some sozinha). |

**Recomendação prática:** use a **Cora para o assistente financeiro/cobrança da sua empresa**; resolva o “parceiro” por **modelo contratual/fiscal (A ou B)**, não inventando um arranjo de pagamento próprio.

---

## 2. O que a API Cora oferece hoje

Fontes principais:
- [Instruções iniciais](https://developers.cora.com.br/docs/instrucoes-iniciais)
- [Índice llms.txt](https://developers.cora.com.br/llms.txt)
- [Integração Direta](https://developers.cora.com.br/docs/integracao-direta)
- [Termos das APIs](https://www.cora.com.br/termos-e-condicoes-de-apis/)
- [Página de integrações / taxas](https://www.cora.com.br/integracoes/)

### 2.1 Modalidades

1. **Integração Direta** — você integra as APIs na **sua** conta PJ (seu caso). Autenticação: certificado + private key (mTLS) + token OAuth client-credentials.  
2. **Parceria Cora** — pensada para ERP/CRM/BPO que oferecem Cora aos **clientes finais** deles (mais burocracia de parceria; não é necessário se o sistema é só seu).

URLs (Integração Direta):
- Stage: `https://matls-clients.api.stage.cora.com.br/`
- Produção: `https://matls-clients.api.cora.com.br/`

### 2.2 Capacidades úteis para o assistente

| Capacidade | API / recurso | Uso no assistente |
|---|---|---|
| Emitir boleto / boleto+PIX | `POST /v2/invoices` | Cobrança única ao cliente |
| Carnê (parcelado) | Emissão de carnê v2 | Parcelamento |
| QR Code Pix | QR Code Pix v2 | Pix avulso |
| Consultar / cancelar cobrança | Consulta e cancelamento v2 | Conciliação e régua |
| Notificações nativas | Objeto `notification` (EMAIL / SMS) + regras de datas | Lembretes de vencimento/pagamento |
| Webhooks | Endpoints + eventos (`invoice.paid`, etc.) | Disparar e-mail customizado, split interno, NFS-e |
| Extrato / saldo / conta | APIs de conta | Assistente financeiro |
| Pagar boletos / DARF / GPS | Iniciação de pagamento | Contas a pagar |
| Transferir | Iniciação de transferência | Repasse ao parceiro (com aprovação no app) |
| NFS-e | APIs de nota fiscal de serviço | Automatizar emissão (onde habilitado) |
| Stage “pagar boleto” | API de stage | Testar fluxo de liquidação |

### 2.3 O que **não** existe na Cora (para o seu caso)

- Split / marketplace / múltiplos recebedores na liquidação  
- Contas de sellers / subcontas  
- Liquidação automática “X% para CNPJ A, Y% para CNPJ B” no mesmo boleto  

Ou seja: a Cora é um **banco digital + API de cobrança/pagamento da sua conta**, não um PSP de marketplace.

### 2.4 Custos (referência pública; confirme no app)

- **Cora Pro** (necessário para Integração Direta): cerca de **R$ 44,90/mês**  
- Por transação compensada via API (conforme termos públicos):
  - Boleto código de barras: **R$ 1,70**
  - Boleto/QR Pix: **1% até o teto de R$ 0,50** (a partir de ~R$ 50, fica R$ 0,50)
  - TED: **R$ 2,00**
  - Cancelamento de boleto/QR gerado via API: **R$ 0,30**

Para empresa pequena com volume moderado, o custo é **baixo** e compatível com uso interno.

### 2.5 E-mail com modelos pré-configurados

A Cora envia notificações de cobrança por **EMAIL/SMS** com regras de data (`NOTIFY_ON_DUE_DATE`, `NOTIFY_WHEN_PAID`, etc.), mas o layout é o da Cora.

Para **modelos seus** (marca, texto jurídico, anexar NFS-e, discriminar serviços, PDF customizado):

1. Seu sistema gera a cobrança na Cora.  
2. Recebe PDF/linha digitável/QR da API.  
3. Envia e-mail com template próprio (Resend, Amazon SES, Postmark, Nodemailer + React Email, etc.).  
4. Usa webhook `invoice.paid` para e-mail de confirmação / gatilho de NFS-e / repasse.

Isso é totalmente viável e é o desenho recomendado do “assistente”.

---

## 3. Atenção: dois “splits” diferentes (não misturar)

| Conceito | O que é | Relevância para você |
|---|---|---|
| **Split comercial / marketplace** | Dividir o valor pago pelo cliente entre 2+ recebedores (você + parceiro) | É o que você quer operacionalmente |
| **Split payment da Reforma Tributária (IBS/CBS)** | Segregação automática de tributo na liquidação do pagamento, com vínculo DFe ↔ pagamento (NT 2026.001 etc.) | Vai afetar **todo mundo** no futuro; **não resolve** o split com parceiro |

O “split payment” da RFB/CGIBS trata de **imposto**, não de repartir receita entre dois prestadores. Mesmo com DF agregador e vinculação boleto↔nota, o problema contábil do tomador com **duas NFS-e e um pagamento a um CNPJ** continua sendo outro assunto.

---

## 4. Dá para criar legalmente um “split próprio” sem burocracia?

### 4.1 O risco regulatório (BACEN / arranjos de pagamento)

Quando uma empresa:

1. Emite cobrança no valor total,  
2. Recebe o dinheiro do cliente na **própria conta**,  
3. Depois **repassa** parte ao parceiro,

ela está, na prática, **tocando fluxo financeiro de terceiro**.

No mercado brasileiro, a distinção crítica é:

- **Split na liquidação (origem):** o PSP/adquirente/IP autorizado divide o valor **antes** de cair na sua conta operacional como “caixa de terceiros”. Você não “segura” o dinheiro do parceiro.  
- **Receive-and-forward (recebe e repassa):** o valor do parceiro entra na sua conta e sai depois. Isso é o modelo que mais se aproxima de **intermediação/facilitação de pagamento** e, em escala ou se mal estruturado, atrai leitura de atividade típica de **instituição de pagamento** / facilitador.

Para o seu perfil (“só a minha empresa, pequena, não comercial”):

- Volume baixo **reduz** a probabilidade de o BACEN olhar para você como IP.  
- Mesmo assim, **não elimina** risco fiscal/contábil/contratual.  
- “Não é produto comercial” **não é** escudo regulatório se o fluxo for de intermediação de recursos de terceiros.

**Conclusão regulatória prática:**  
Não construa um “sistema de split” caseiro que receba dinheiro de cliente em nome de dois CNPJs. Se precisar de split real na liquidação, use **IP já autorizada** (Asaas, Iugu, Pagar.me, Mercado Pago Marketplace, etc.). Se não quiser essa burocracia/custo, mude o **modelo contratual** (seção 5).

### 4.2 O risco que mais importa para você: o cliente Lucro Real

Mesmo que “passe” regulatoriamente no baixo volume, o cliente PJ (principalmente Lucro Real) precisa:

1. **Despesa necessária, usual e comprovada** (IRPJ/CSLL).  
2. **Documento fiscal hábil** (NFS-e) do prestador.  
3. **Pagamento conciliável** com esse documento (extrato, boleto, Pix).  
4. Em muitos casos, **retenções** (IRRF, PIS/COFINS/CSLL, INSS, ISS) aplicadas sobre o pagamento/nota correta.

#### Cenário problemático (o que você descreveu)

- NFS-e sua: R$ 6.000 (CNPJ A)  
- NFS-e do parceiro: R$ 4.000 (CNPJ B)  
- Cliente paga **um** boleto/Pix de R$ 10.000 para **CNPJ A**  
- Você “splita” e manda R$ 4.000 ao parceiro  

Para a contabilidade do cliente:

- Há pagamento de R$ 10.000 a **A**, mas nota de A só R$ 6.000.  
- Há nota de **B** de R$ 4.000 **sem** pagamento a B (ou com pagamento “indireto” sem vínculo bancário claro).  
- Resultado típico: glosa de despesa, pergunta do auditor interno, ou o financeiro pede: **“emite uma nota só”**.

Isso é exatamente a dor de cabeça que você quer evitar.

#### Como o Lucro Real “justifica” despesa

Em regra, a justificativa é: **serviço tomado + NFS-e do prestador + pagamento a esse prestador (ou a intermediário formal com documentação robusta)**.  
Não existe atalho confortável do tipo “paguei o CNPJ A, mas a nota é do CNPJ B” sem estrutura contratual clara (e mesmo assim costuma ser rejeitada pelo contador do cliente).

---

## 5. Modelos viáveis (do melhor ao pior para o seu caso)

### Modelo A — Prestador principal + subcontratação (**recomendado**)

**Como funciona**

1. Contrato comercial: **você** é o contratado do cliente pelo serviço completo.  
2. Você emite **uma NFS-e** no valor total ao cliente.  
3. Cliente paga **um** boleto/Pix (Cora) para o **seu** CNPJ.  
4. Parceiro é **seu** subcontratado: emite NFS-e **para você**, não para o cliente.  
5. Você paga o parceiro (TED/Pix/transferência Cora, preferencialmente automatizada via webhook + transferência).

```text
Cliente  --paga R$10k-->  Você (CNPJ A)
Cliente  <--NFS-e R$10k-- Você
Você     --paga R$4k--->  Parceiro (CNPJ B)
Você     <--NFS-e R$4k--  Parceiro
```

**Para o cliente:** perfeito — 1 nota, 1 pagamento, retenções sobre 1 documento.  
**Para você:** tributa a receita bruta total (ISS/PIS/COFINS/IR conforme regime) e deduz/custa o que paga ao parceiro (conforme seu regime).  
**Para o parceiro:** relação B2B com você.  
**Regulatório:** você não intermediou pagamento de terceiro; recebeu preço do **seu** serviço e pagou **seu** fornecedor.  
**Cora:** encaixa 100% (invoice + webhook + transfer + opcional NFS-e).

**Cuidados**

- Contrato com cliente deve permitir subcontratação (ou ser silente de forma adequada).  
- Contrato com parceiro: escopo, preço, prazo, confidencialidade, responsabilidade técnica.  
- ISS: em geral **não** se “abate” a nota do parceiro da base da sua nota ao cliente (salvo regra municipal específica). Planeje a margem.  
- Retenção previdenciária (quando aplicável): há regras de destaque/dedução com subcontratadas (IN RFB 971/2009 e SC Cosit correlatas) — seu contador deve calibrar.  
- Cosit (ex.: SC Cosit 16/2020): quem executa “em seu nome e por sua conta” tributa o **total**; quem só intermedia pode tributar só a comissão — mas aí o prestador real emite ao cliente. Escolha um enquadramento e seja consistente.

### Modelo B — Dois contratos / duas cobranças (também limpo)

```text
Cliente --paga R$6k--> Você     + NFS-e A R$6k
Cliente --paga R$4k--> Parceiro + NFS-e B R$4k
```

**UX:** um e-mail seu com **dois** PDFs/links, ou um pacote “cobrança conjunta” sem misturar CNPJs.  
**Contábil do cliente:** limpo.  
**Seu sistema:** gera a sua cobrança na Cora; a do parceiro pode ser link/PDF dele ou, no futuro, outro provedor.  
**Quando usar:** cliente exige contratar cada CNPJ, ou vocês não querem subcontratação.

### Modelo C — Split nativo via IP (Asaas / Iugu / Pagar.me / Mercado Pago)

```text
Cliente paga 1 cobrança no PSP
  -> liquidação: R$6k carteira A + R$4k carteira B
Cada um emite sua NFS-e ao cliente
```

**Prós:** você não segura dinheiro do parceiro; automação boa.  
**Contras para o tomador Lucro Real:** ainda há **duas NFS-e** e **um** comprovante de pagamento ao arranjo/emissor da cobrança. Muitos contadores **ainda** pedem pacote documental forte (contrato conjunto, demonstrativo do split, IDs da liquidação) ou preferem Modelo A/B.  
**Contras para você:** onboarding KYC do parceiro em subconta, taxas do gateway, troca parcial ou total da Cora para cobrança, compliance do PSP.  
**Encaixa no seu desejo?** Só se o split operacional for inegociável **e** o cliente aceitar a papelada. Para empresa pequena “não comercial”, costuma ser **overkill**.

### Modelo D — “Split caseiro” Cora (receber tudo + transferir) — **evitar como regra**

```text
Cliente paga R$10k na sua Cora
Você transfere R$4k ao parceiro
Parceiro emite NFS-e ao cliente
```

**Problemas**

- Descompasso pagamento × NF no cliente.  
- Aparência de intermediação de recursos.  
- Se você “não registra receita dos R$4k”, e o parceiro nota ao cliente, o fluxo bancário não bate.  
- Se você registra os R$10k como receita e o parceiro também nota ao cliente, risco de **bitributação / inconsistência / conflito de narrativa fiscal**.  
- Transferência Cora exige **aprovação no app** — ruim para automação de split.

Use no máximo como **exceção pontual** (erro operacional), não como produto.

### Modelo E — Intermediação pura (você só “cobra comissão”)

Só faz sentido se o contrato disser que você **intermedeia** e o prestador real é o parceiro. Aí:

- Sua receita = comissão.  
- Quem presta o serviço principal emite a NFS-e principal ao cliente.  
- Em geral **não** resolve o desejo de “uma cobrança única com duas notas suas+parceiro”.

---

## 6. Matriz de decisão (sua empresa pequena)

| Critério | A Subcontratação | B Duas cobranças | C Split IP | D Split caseiro Cora |
|---|---|---|---|---|
| Dor de cabeça do cliente Lucro Real | Baixa | Baixa | Média/Alta | Alta |
| Chance do cliente pedir “1 nota só” | Baixa (já é 1) | Média | Alta | Muito alta |
| Burocracia BACEN / IP | Nenhuma extra | Nenhuma | Usa IP de terceiros | Zona cinza |
| Encaixa na Cora | Ótimo | Ótimo (sua parte) | Parcial (migrar cobrança) | Ruim |
| Custo tributário seu | Tributa o bruto | Tributa só sua parte | Tributa só sua parte | Ambíguo / perigoso |
| Complexidade de software | Média | Baixa/Média | Alta | Falsa simplicidade |
| Adequação “não sou fintech” | Excelente | Excelente | Aceitável | Ruim |

**Escolha sugerida:**  
- Default comercial/fiscal: **Modelo A**.  
- Se o cliente ou o parceiro exigirem vínculo direto: **Modelo B**.  
- Assistente bancário: **Cora Integração Direta** em qualquer dos dois.

---

## 7. Arquitetura sugerida do assistente (sem fintech)

### 7.1 Escopo do produto (uso interno)

1. Cadastro de clientes PJ e parceiros.  
2. Modelos de cobrança (texto, serviços, juros/multa/desconto, vencimento).  
3. Emissão Cora (boleto+PIX).  
4. E-mail com templates (HTML/PDF) + anexos.  
5. Régua de cobrança (Cora nativa + e-mails próprios).  
6. Webhook → status pago / conciliação.  
7. Contas a pagar (boletos/impostos) e visão de extrato.  
8. Módulo “parceiro”:
   - **A:** após `invoice.paid`, criar obrigação a pagar ao parceiro + iniciar transferência.  
   - **B:** gerar pacote com duas cobranças (sua automática; do parceiro assistida).  
9. (Opcional) NFS-e via Cora ou provedor municipal/focus/nfe.io etc.

### 7.2 Stack barata / gratuita o suficiente

| Camada | Opções |
|---|---|
| App | Next.js / Node (ou Python FastAPI) |
| DB | Postgres (Neon free / Supabase) |
| Filas/jobs | Vercel cron, Inngest free, ou worker simples |
| E-mail | Resend free tier, Amazon SES, Mailgun |
| Templates | React Email / MJML |
| Storage PDF | S3/R2/Blob |
| Auth interno | Clerk / Auth.js (só você e equipe) |
| Banco/cobrança | **Cora Pro + API** |
| Observabilidade | logs + alerta de webhook falho |

Não precisa de Open Banking avançado no dia 1: a Cora já cobre o núcleo.

### 7.3 Fluxo feliz (Modelo A)

```text
[Operador] cria proposta
   -> gera NFS-e (sua) + invoice Cora
   -> e-mail template "Cobrança + NF" ao cliente
[Cliente] paga boleto/Pix
   -> webhook invoice.paid
   -> marca título liquidado
   -> cria conta a pagar parceiro (se houver)
   -> inicia transferência Cora (aprovação no app)
   -> e-mail interno + e-mail de quitação ao cliente
```

---

## 8. Alternativas se a Cora não bastar no futuro

| Necessidade | Alternativa |
|---|---|
| Split na liquidação com subcontas | **Asaas** (split por `walletId`), **Iugu**, **Pagar.me**, Mercado Pago Marketplace |
| Banking-as-a-Service mais profundo | Stark Bank, Celcoin, etc. (mais “fintech”; fora do seu objetivo) |
| Só cobrança recorrente simples | Asaas / EFI (Gerencianet) / Inter API |
| NFS-e multi-município | Focus NFe, Enotas, NFe.io, ou API Cora onde disponível |

Para o seu briefing atual (**não comercial, empresa pequena**), migrar só por causa de split costuma ser **pior** do que adotar Modelo A.

---

## 9. Burocracias e dificuldades reais

1. **Cora Pro + certificado mTLS** — setup inicial chato, mas único.  
2. **Saídas (pagamento/transferência) pedem aprovação no app** — automação parcial.  
3. **NFS-e** depende de município/habilitação.  
4. **Parceiro sem contrato** — maior risco do que a API.  
5. **Margem no Modelo A** — você pode pagar ISS/PIS/COFINS sobre o bruto; calcule preço.  
6. **Cliente Lucro Real** — o que ele odeia é divergência NF × pagamento; resolva no desenho, não no “jeitinho de split”.  
7. **Reforma Tributária** — prepare-se para vincular cobrança ↔ DFe no futuro; isso é obrigação fiscal nova, não feature de marketplace.  
8. **PLD/KYC** — se um dia usar subcontas de IP, o parceiro passa por onboarding.

---

## 10. Checklist de validação com seu contador (antes de construir)

Leve estas perguntas objetivas:

1. Nosso serviço com parceiro deve ser **subcontratação (A)** ou **contratos paralelos (B)**?  
2. No nosso regime (Simples/Presumido/Real), o Modelo A **mata a margem**?  
3. Há retenção de INSS/IRRF/ISS típica nos nossos clientes?  
4. O cliente Lucro Real aceita pacote “1 NF + 1 boleto” (A) sem questionar subcontratação?  
5. Se insistirmos em 2 NFs ao cliente, eles aceitam **2 pagamentos**?  
6. Existe alguma regra municipal de ISS que mude a análise?

---

## 11. Plano de implementação sugerido (fases)

### Fase 0 — Decisão de negócio (sem código)
- Fechar Modelo A ou B com contador.  
- Modelo de contrato cliente + parceiro.  
- Tabela de preços já embutindo impostos do Modelo A (se for o caso).

### Fase 1 — Assistente Cora (alto valor, baixo risco)
- Integração Direta stage → produção.  
- Emitir boleto+PIX, webhooks, templates de e-mail, painel simples.  
- Contas a receber/pagar + extrato.

### Fase 2 — Parceiros
- Cadastro de parceiros e regras de rateio **interno** (só memória de cálculo).  
- No Modelo A: após pagamento, gerar AP + transferência.  
- Relatórios para contabilidade (CSV/PDF).

### Fase 3 — Só se necessário
- Avaliar Asaas/Iugu **apenas** se surgir volume real de splits e Modelo A/B não servir.  
- Nunca vender isso como produto externo sem reassessorar regulatório.

---

## 12. Fontes consultadas

### Cora
- https://developers.cora.com.br/docs/instrucoes-iniciais  
- https://developers.cora.com.br/llms.txt  
- https://developers.cora.com.br/docs/integracao-direta  
- https://developers.cora.com.br/docs/utilizacao-das-apis  
- https://developers.cora.com.br/reference/emissao-de-boleto-registrado-v2  
- https://developers.cora.com.br/reference/iniciacao-de-transferencia  
- https://developers.cora.com.br/reference/pagar-boleto-em-stage  
- https://developers.cora.com.br/reference/exemplo-de-post-da-notificacao  
- https://www.cora.com.br/integracoes/  
- https://www.cora.com.br/termos-e-condicoes-de-apis/  

### Split / regulatório (contexto)
- Discussões sobre split na liquidação vs receber-e-repassar (BACEN / marketplaces; Circular 3.682 e evolução de regras de facilitadores; Res. BCB 80/81 sobre IPs).  
- Documentação Asaas Split: https://docs.asaas.com/docs/split-de-pagamentos  
- Documentação Iugu Split: https://dev.iugu.com/docs/split-de-pagamentos  

### Contábil / fiscal (contexto)
- SC Cosit sobre execução em nome próprio vs intermediação (ex.: SC Cosit nº 16/2020).  
- Regras de subcontratação e retenção previdenciária (IN RFB 971/2009; SC Cosit nº 32/2017).  
- Dedutibilidade e comprovação no Lucro Real (despesa necessária + documentação).  
- Notas sobre split payment tributário da Reforma (NT 2026.001 / manuais CGIBS) — **tema distinto**.

---

## 13. Conclusão

Para a sua empresa pequena, o caminho de menor dor é:

1. **Construir o assistente na Cora** (cobrança, Pix, e-mails com seus templates, conciliação, pagamentos).  
2. **Não inventar um arranjo de split caseiro.**  
3. **Tratar o parceiro como subcontratado (Modelo A)** para o cliente ter 1 nota + 1 boleto — isso ataca a raiz do pedido “emite uma nota só”.  
4. Se ambos precisam aparecer para o cliente, use **duas cobranças (Modelo B)**, não um boleto único maquiado.  
5. Gateway com split fica como **plano B operacional**, não como fundação.

Se quiser o próximo passo de engenharia: especificar a Fase 1 (esquema de dados, endpoints Cora a usar, templates de e-mail e fluxo de webhooks) já no repositório.
