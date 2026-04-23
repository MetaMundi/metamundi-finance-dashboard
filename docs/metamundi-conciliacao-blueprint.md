# MetaMundi Conciliação — Blueprint de Arquitetura e Engenharia de Dados

## 1) Modelo de dados proposto (PostgreSQL)

> Objetivo: suportar conciliação multi-origem, rastreabilidade total e regras financeiras por competência/caixa.

### 1.1 Princípios de modelagem

- **Ledger-first**: toda entrada de arquivo gera registros imutáveis em staging + eventos de processamento.
- **Separação de domínio**: operacional (OS/faturas/comissões), financeiro (transações/caixa), e governança (auditoria).
- **Conciliação explicável**: cada match guarda score, regra aplicada e usuário/robô responsável.
- **Bitemporalidade leve**: `event_date` (competência) e `posted_at` (caixa), quando aplicável.

### 1.2 Entidades centrais

#### Cadastro e operação

- `customers` (clientes)
- `passengers` (passageiros)
- `suppliers` (fornecedores)
- `sales_agents` (vendedores)
- `service_orders` (OS)
- `service_order_items` (itens por OS: aéreo, hotel, seguro etc.)

#### Financeiro

- `accounts` (contas: banco/carteira/cartão)
- `bank_statements` (arquivos de extrato)
- `bank_transactions` (linhas normalizadas de extrato)
- `receivable_titles` (títulos a receber por OS/fatura)
- `payable_titles` (títulos a pagar por OS/fornecedor)
- `invoices` (boletos/faturas recebidas ou pagas)
- `invoice_allocations` (N:N entre invoice e OS/títulos)
- `credit_card_owners` (donos de cartão)
- `credit_card_statements` (faturas por dono do cartão)
- `credit_card_items` (lançamentos de fatura)
- `commissions_in` (RAV/DU/comissão fornecedor - entrada)
- `commissions_out` (comissão vendedor - saída)
- `operational_expenses` (despesas sem vínculo direto com OS)

#### Conciliação e auditoria

- `reconciliation_runs` (execuções do motor)
- `reconciliation_matches` (resultado do match)
- `reconciliation_match_links` (match 1:N e N:N)
- `orphan_transactions` (sem correlação com OS/título)
- `file_uploads` (metadados de upload)
- `file_parse_events` (eventos ETL por arquivo)
- `audit_log` (histórico de criação/edição/exclusão/aprovação)

### 1.3 DDL de referência (resumido)

```sql
create table service_orders (
  id uuid primary key,
  os_number text unique not null,
  customer_id uuid not null,
  passenger_id uuid,
  sales_agent_id uuid,
  status text not null check (status in ('OPEN','PARTIAL','CLOSED','CANCELED')),
  sale_amount numeric(14,2) not null,
  supplier_cost_amount numeric(14,2) not null,
  competence_date date not null,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table invoices (
  id uuid primary key,
  invoice_number text,
  invoice_type text not null check (invoice_type in ('RECEIVABLE','PAYABLE','CARD_STATEMENT','BOLETO')),
  counterparty_type text not null check (counterparty_type in ('CUSTOMER','SUPPLIER','BANK','CARD_BRAND')),
  counterparty_id uuid,
  total_amount numeric(14,2) not null,
  due_date date,
  issue_date date,
  status text not null check (status in ('OPEN','PARTIAL','PAID','CANCELED')),
  owner_card_id uuid,
  created_at timestamptz not null default now()
);

create table invoice_allocations (
  id uuid primary key,
  invoice_id uuid not null references invoices(id),
  service_order_id uuid references service_orders(id),
  receivable_title_id uuid,
  payable_title_id uuid,
  allocated_amount numeric(14,2) not null,
  allocation_type text not null check (allocation_type in ('REVENUE','SUPPLIER_COST','FEE','COMMISSION')),
  unique(invoice_id, service_order_id, receivable_title_id, payable_title_id, allocation_type)
);

create table bank_transactions (
  id uuid primary key,
  account_id uuid not null,
  statement_id uuid,
  txn_hash text not null,
  txn_date date not null,
  posted_at timestamptz,
  description text,
  amount numeric(14,2) not null,
  direction text not null check (direction in ('IN','OUT')),
  source_type text not null check (source_type in ('OFX','CSV','XLSX','PDF_OCR','MANUAL')),
  raw_payload jsonb,
  is_reconciled boolean not null default false,
  created_at timestamptz not null default now(),
  unique(account_id, txn_hash)
);

create table reconciliation_matches (
  id uuid primary key,
  run_id uuid not null,
  match_type text not null check (match_type in ('RECEIVABLE','PAYABLE','COMMISSION_IN','COMMISSION_OUT','OPEX')),
  status text not null check (status in ('AUTO_MATCHED','SUGGESTED','MANUAL_MATCHED','REJECTED')),
  confidence_score numeric(5,4) not null,
  rule_code text not null,
  explained_by jsonb not null,
  created_by text not null default 'SYSTEM',
  created_at timestamptz not null default now()
);

create table reconciliation_match_links (
  id uuid primary key,
  match_id uuid not null references reconciliation_matches(id),
  bank_txn_id uuid references bank_transactions(id),
  receivable_title_id uuid,
  payable_title_id uuid,
  invoice_id uuid,
  service_order_id uuid,
  linked_amount numeric(14,2) not null
);

create table audit_log (
  id bigserial primary key,
  entity_name text not null,
  entity_id text not null,
  action text not null check (action in ('INSERT','UPDATE','DELETE','APPROVE','RECONCILE','UNDO_RECONCILE','UPLOAD')),
  diff jsonb,
  actor_id text not null,
  actor_type text not null check (actor_type in ('USER','SYSTEM')),
  created_at timestamptz not null default now()
);
```

### 1.4 Relacionamentos críticos

- `service_orders 1:N receivable_titles` e `service_orders 1:N payable_titles`.
- `invoices N:N service_orders` via `invoice_allocations` (regra de boleto quitando múltiplas OS).
- `credit_card_owners 1:N credit_card_statements` para segregação por dono e consolidação no caixa.
- `bank_transactions N:N títulos/faturas/OS` via `reconciliation_match_links`.
- `reconciliation_matches` armazena explicabilidade e versionamento de decisão.

### 1.5 Índices recomendados

- `bank_transactions (txn_date, amount, direction, is_reconciled)`.
- `service_orders (competence_date, status)`.
- `receivable_titles (due_date, status)` / `payable_titles (due_date, status)`.
- GIN em `bank_transactions.raw_payload` e `reconciliation_matches.explained_by`.
- Particionamento mensal para `bank_transactions`, `audit_log`, `reconciliation_matches`.

---

## 2) Fluxo de processamento de arquivo (upload → conciliação)

## 2.1 Pipeline macro

1. **Upload**
   - Usuário envia XLSX/CSV/PDF/Imagem/TXT-OFX.
   - Sistema grava arquivo bruto (object storage) e cria `file_uploads`.

2. **Detecção + parsing**
   - Identifica tipo MIME/extensão.
   - Executa parser específico:
     - XLSX/CSV → parser tabular com mapeamento de colunas.
     - TXT/OFX → parser financeiro estruturado.
     - PDF/Imagem → OCR + extração semântica de campos.
   - Resultado cai em `staging_*` + `file_parse_events`.

3. **Normalização**
   - Padroniza data (`YYYY-MM-DD`), moeda (BRL), sinal (`IN`/`OUT`), descrição canônica.
   - Gera `txn_hash` para deduplicação.
   - Carrega em `bank_transactions`, `invoices` ou títulos.

4. **Enriquecimento**
   - Tokenização de descrição (PIX, CNPJ, NSU, últimos 4 dígitos cartão, agência/conta).
   - Busca entidades candidatas (OS, cliente, fornecedor, vendedor, fatura).

5. **Motor de conciliação**
   - Executa regras determinísticas + score probabilístico.
   - Gera `AUTO_MATCHED`, `SUGGESTED` ou `orphan_transactions`.

6. **Pós-processamento financeiro**
   - Atualiza status de títulos/OS/faturas (`OPEN/PARTIAL/PAID`).
   - Detecta discrepâncias (pagamento > custo fornecedor previsto, recebimento < venda etc.).

7. **Workspace humano + auditoria**
   - Analista aprova/rejeita sugestões.
   - Toda ação registra `audit_log`.

8. **Reprocessamento controlado**
   - Edição/exclusão de base gera nova versão lógica e replay do run de conciliação.

## 2.2 Regras de matching (ordem de prioridade)

1. **Match exato**: valor exato + referência forte (OS, NSU, Nosso Número, CNPJ).
2. **Match por composição**: 1 transação quitando N OS (subset sum com tolerância de centavos).
3. **Match temporal**: janela de datas por tipo (PIX: D0/D+1, boleto: D+1..D+3).
4. **Match por contraparte**: histórico cliente/fornecedor recorrente.
5. **Match semântico**: similaridade textual da descrição vs. entidades.

### 2.3 Tratamento de exceções

- **Duplicidade**: mesmo `txn_hash` + conta → bloqueio automático.
- **Estorno**: transação reversa cria vínculo de reversão no match original.
- **Ambiguidade alta**: fica em `SUGGESTED` com ranking top-3.
- **Sem vínculo**: vai para `orphan_transactions` com classificação preliminar.

---

## 3) Dashboard principal (KPIs essenciais)

## 3.1 Bloco executivo (visão mensal)

- **Receita conciliada** (R$ e % da receita de venda).
- **Custo conciliado fornecedor** (R$ e % do custo previsto).
- **Lucro líquido real** = Receita conciliada + Comissões de entrada − Custos fornecedores − Comissões pagas − OPEX.
- **Taxa de automação da conciliação** = `AUTO_MATCHED / total transações`.
- **Valor órfão** (R$ sem correlação).

## 3.2 Bloco operacional

- **OS com saldo devedor fornecedor** (qtd, valor e aging).
- **Recebíveis vencidos** (D+1, D+7, D+30).
- **Discrepâncias críticas**:
  - pagamento fornecedor > custo OS;
  - recebimento cliente < valor venda;
  - comissão paga sem receita conciliada.
- **Faturas de cartão por dono** (aberto, pago, próximo vencimento).

## 3.3 Bloco de auditoria

- Uploads processados hoje (sucesso/erro).
- Ajustes manuais de conciliação (qtd e valor movimentado).
- Top usuários por intervenções manuais.
- Linha do tempo de eventos críticos.

## 3.4 Filtros obrigatórios

- Período por **competência** e por **caixa**.
- Conta bancária, dono do cartão, cliente, fornecedor, vendedor.
- Status OS/título/fatura e nível de confiança da conciliação.

---

## 4) Camada de IA conversacional (NLP + raciocínio financeiro)

## 4.1 Arquitetura sugerida

- **NLU/Parser de intenção**: identifica intenção (saldo fornecedor, match PIX, lucro líquido, órfãos).
- **Gerador de plano**: transforma pergunta em plano de consulta com métricas e filtros.
- **Query Builder seguro**: monta SQL parametrizado em cima de *views semânticas*.
- **Executor + verificador**: roda consultas, valida consistência numérica e limites de confiança.
- **Narrador financeiro**: responde em linguagem natural + tabela + justificativa de cálculo.

## 4.2 Views semânticas para IA

- `vw_os_financial_position`
- `vw_cashflow_realized`
- `vw_reconciliation_orphans`
- `vw_commissions_net`
- `vw_card_statements_by_owner`

## 4.3 Pseudocódigo de orquestração

```pseudo
function answer_financial_question(user_id, question_text, context_filters):
    intent = nlu.detect_intent(question_text)
    entities = nlu.extract_entities(question_text)
    period = time_resolver.resolve(entities, context_filters.default_period)

    plan = planner.build(intent, entities, period)
    # Exemplo de plan:
    # - metric: os_supplier_outstanding
    # - group_by: [os_number, supplier]
    # - filters: competence_month=2026-03

    sql_query = sql_builder.from_plan(plan)
    safe_query = guardrails.enforce(sql_query,
                                    allowed_views=[vw_os_financial_position,
                                                   vw_cashflow_realized,
                                                   vw_reconciliation_orphans,
                                                   vw_commissions_net,
                                                   vw_card_statements_by_owner],
                                    max_rows=5000,
                                    pii_policy='masked')

    result = db.execute(safe_query)
    checks = finance_validator.run(result, plan)
    # checks: soma, sinais, reconciliação cruzada com ledger

    explanation = llm.compose(
        question=question_text,
        result=result,
        checks=checks,
        business_rules=[
          'boleto pode liquidar N OS',
          'cartão segregado por dono e consolidado no caixa',
          'destacar discrepâncias'
        ],
        response_format=['insight_text', 'table', 'next_actions']
    )

    audit.log(user_id=user_id,
              action='NLP_QUERY',
              input=question_text,
              plan=plan,
              sql=safe_query,
              row_count=result.count)

    return explanation
```

## 4.4 Lógica de respostas para perguntas-alvo

1. **"Quais OSs de março ainda possuem saldo devedor de fornecedor?"**
   - Filtra `competence_date` em março.
   - Calcula `supplier_cost_amount - paid_supplier_amount` por OS.
   - Retorna apenas saldo > 0 com aging.

2. **"O Pix de R$ 500 recebido ontem no Banco Cora corresponde a qual cliente ou OS?"**
   - Busca transação por conta+valor+data.
   - Roda rank de candidatos por referência textual, histórico e proximidade de vencimento.
   - Retorna match provável + % de confiança + alternativas.

3. **"Qual o lucro líquido real após descontar as comissões pagas e as faturas de cartão deste mês?"**
   - Lucro líquido real por caixa no mês:
   - `receita_conciliada + comissao_entrada - custo_fornecedor_pago - comissao_saida - despesas_cartao - opex`.

4. **"Liste valores no extrato que não possuem correlação com nenhuma OS cadastrada (Órfãos)."**
   - Consulta `vw_reconciliation_orphans` por período/conta.
   - Classifica provável natureza (taxa, tarifa, despesa administrativa, recebimento não identificado).

---

## 5) Regras de negócio implementáveis como políticas

- `POL-001`: Permitir `invoice_allocations` N:N entre fatura e OS.
- `POL-002`: Fatura de cartão obrigatoriamente vinculada a `credit_card_owner`.
- `POL-003`: Bloquear pagamento final ao fornecedor quando `paid_supplier > supplier_cost_amount * tolerance_limit` sem aprovação.
- `POL-004`: Comissão de vendedor só pode ser `APPROVED` quando receita da OS estiver conciliada acima de limiar configurado.
- `POL-005`: Toda exclusão lógica de lançamento exige motivo e gravação em `audit_log`.

## 6) Entrega incremental recomendada

- **Sprint 1**: ingestão multi-formato + staging/auditoria.
- **Sprint 2**: motor de conciliação recepção/pagamento + órfãos.
- **Sprint 3**: comissões + cartões por dono + dashboard executivo.
- **Sprint 4**: IA conversacional com guardrails SQL + explicações financeiras.
