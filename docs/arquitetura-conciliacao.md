# Arquitetura e Requisitos — Sistema de Conciliação Automática MetaMundi

## 1) Contexto e objetivos

A MetaMundi opera com alto volume de ordens de serviço (OS), faturamento consolidado, múltiplos meios de recebimento/pagamento, emissão de NF e pagamento de comissões. Isso exige um sistema de conciliação automática auditável, escalável e aderente a boas práticas contábeis.

### Objetivos principais

- Importar dados de múltiplas fontes (planilhas, OFX/CSV, XML de NF, comprovantes).
- Automatizar conciliação de recebimentos e pagamentos, incluindo estornos e ajustes.
- Relacionar OS, faturas, NF e comissões de ponta a ponta.
- Oferecer interface web + exportações Excel/CSV e consultas em linguagem natural.
- Garantir fechamento mensal, segurança, segregação de funções e trilha de auditoria.

---

## 2) Requisitos funcionais

### 2.1 Ingestão de dados

- Upload de OFX/CSV/XML/planilhas com **mapeamento de colunas** reutilizável.
- Validação automática (datas, valores, status, duplicidade e integridade mínima).
- Integrações API com bancos/open finance, gateways de pagamento, ERP e NF.
- Registro em banco relacional como **fonte única da verdade**.

### 2.2 Módulos de conciliação

#### A) Conciliação bancária (Livro Caixa × Banco)

- Match por data, valor e referência com tolerância (data ±N dias, valor ±X%).
- Itens sem correspondência vão para conta suspense.
- Relatório de prova de caixa (4 colunas): saldo inicial, receitas, despesas e saldo final.
- Visão por conta bancária e visão consolidada multi-conta.

#### B) Conciliação de receitas (AR)

- Vincular recebimentos a OS/faturas por referência + valor + data + documento fiscal.
- Suportar pagamento parcial e baixa parcial.
- Suportar faturas consolidadas com vínculo N:N entre fatura e OS.
- Aging de recebíveis com alertas de vencimento.
- Lembretes automáticos a clientes (e-mail/WhatsApp).

#### C) Conciliação de despesas (AP)

- Cadastro de faturas de fornecedores com vencimento, condições e vínculo opcional à OS.
- Two-way/Three-way match (PO/OS, recebimento, invoice).
- Regras de tolerância configuráveis (ex.: 1–2%).
- Conciliação de pagamentos por múltiplos bancos/canais + taxas bancárias.
- Registro de créditos, abatimentos, estornos e bonificações.

#### D) Conciliação de OS

Cada OS deve manter cliente, consultor, serviço, valor de venda, custo, status, vínculo com fatura/NF.

Verificações automáticas por OS:

- Receita total vendida × recebida.
- Custos previstos × pagos × conciliados.
- Diferenças pendentes (câmbio, taxas, ajustes).
- Situação fiscal (NF emitida e conciliada com recebimento).

#### E) Notas fiscais

- Vínculo obrigatório da NF com OS ou fatura.
- Verificação de consistência entre valores faturados, serviço prestado e impostos.

#### F) Comissões

- Regras configuráveis (percentual, faixas progressivas, fixo).
- Liberação da comissão somente após receita conciliada.
- Aprovação e conciliação do pagamento de comissão.
- Relatórios por período, consultor, cliente e OS.

#### G) Consultas e relatórios

- Dashboards com KPIs: AR/AP, aging, fluxo de caixa, OS abertas, comissões.
- Pivot-style grid para análise ad hoc.
- Consulta em linguagem natural (PT/EN) com retorno em texto + tabela.
- Exportação de relatórios em CSV/Excel/PDF.

#### H) Fechamento mensal

- Checklist de fechamento por período.
- Conferência de saldos iniciais/finais e reconciliações pendentes.
- Geração de diário contábil e exportação para ERP.
- Registro de ajustes/reclassificações com trilha completa.

---

## 3) Requisitos não funcionais

### Segurança e governança

- RBAC com segregação de funções (SoD).
- MFA para perfis críticos.
- Criptografia em repouso para dados sensíveis.
- Trilhas de auditoria imutáveis para importações, edições, aprovações e conciliações.

### Escalabilidade e desempenho

- Processamento assíncrono de tarefas pesadas.
- Suporte a milhares de transações/mês com crescimento horizontal.
- Índices e partições para consultas analíticas e reconciliações em lote.

### Usabilidade

- UI responsiva, acessível (WCAG), com experiência estilo planilha.
- Upload drag-and-drop e feedback de validação em tempo real.

### Regionalização

- Padrão pt-BR (moeda, datas, impostos), com suporte multilíngue.

---

## 4) Arquitetura proposta

## 4.1 Camadas

1. **Frontend (React/Vue)**
   - Upload, dashboard, workspace de conciliação, chat de consultas.
2. **Backend (FastAPI/Django/Node.js)**
   - Regras de negócio, APIs, integrações e autorização.
3. **Banco relacional (PostgreSQL)**
   - Entidades operacionais e histórico auditável.
4. **Workers assíncronos (Celery/RQ/queue)**
   - Parsing de arquivos, matching massivo, cálculo de comissões e jobs de fechamento.
5. **Módulo de inteligência**
   - Fuzzy matching + busca semântica (embeddings + índice vetorial).

## 4.2 Entidades de dados (simplificado)

- Clientes
- Fornecedores
- Consultores
- Ordens de Serviço (OS)
- Itens/Detalhes da OS
- Notas Fiscais
- Faturas
- Fatura_OS (vínculo N:N)
- Recebimentos
- Pagamentos
- Movimentos_Bancarios
- Comissoes
- Usuarios
- Audit_Log

---

## 5) Algoritmo-base de conciliação

```text
1. Carregar transações internas (AR/AP) e extrato bancário.
2. Para cada transação não conciliada:
   a. Buscar candidatos por janela de data e tolerância de valor.
   b. Se houver match único: conciliar e vincular IDs.
   c. Se houver múltiplos matches: ranquear por referência (OS/fatura/NF/CNPJ).
   d. Persistindo ambiguidade: mover para fila de pendência.
   e. Sem candidatos: enviar para conta suspense.
3. Recalcular status da OS/fatura: Pago, Parcial, Em aberto, Cancelado.
4. Para AP: exigir two/three-way match antes de autorizar pagamento.
```

### Heurísticas recomendadas

- Normalização textual de descrição (remoção de acentos, tokens, pontuação).
- Distância de Levenshtein para referência aproximada.
- Peso por histórico de correspondência por contraparte.
- Regras de exclusão para prevenir dupla conciliação.

---

## 6) Segurança, auditoria e conformidade

- Separar perfis de cadastro, aprovação, pagamento e conciliação.
- Registrar log de alterações em todas as tabelas críticas.
- Garantir backup diário e testes de restauração.
- Implementar políticas LGPD (minimização, retenção, consentimento, acesso).

---

## 7) Estratégia de testes e QA

- **Unitários:** matching, comissão, aging, validações de upload.
- **Integração:** fluxo completo ingestão → conciliação → relatório → exportação.
- **Performance:** cenário com 100k+ transações.
- **Segurança:** RBAC/SoD, injeção SQL, XSS, trilha de auditoria.
- **Aceitação (UAT):** fechamento mensal real com equipe financeira.

---

## 8) Roadmap de implementação (MVP → Evolução)

### Fase 1 (MVP)

- Modelo de dados base + ingestão CSV/OFX.
- Conciliação bancária e de recebimentos por regras.
- Dashboard inicial, filtros e exportação Excel.
- RBAC essencial + audit log.

### Fase 2

- Faturas consolidadas, AP com three-way match e módulo de comissões.
- Integrações API (bancos/gateways/NF).
- Fechamento mensal assistido.

### Fase 3

- NLP para consultas em linguagem natural.
- Matching inteligente com ML e aprendizagem por feedback.
- Data warehouse e BI avançado.

---

## 9) KPIs recomendados

- Taxa de conciliação automática (%).
- Tempo médio de conciliação por período.
- Pendências em conta suspense (quantidade e valor).
- DSO (dias de vendas em aberto) e aging vencido.
- AP vencido vs. total a pagar.
- % comissão paga com receita efetivamente conciliada.
