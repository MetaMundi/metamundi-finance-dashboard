# Sistema de Conciliação Automática MetaMundi

Este repositório contém a proposta funcional e técnica para o **Sistema de Conciliação Automática MetaMundi**, com foco em:

- Conciliação bancária (livro caixa × extratos).
- Conciliação de contas a receber (OS e faturas consolidadas).
- Conciliação de contas a pagar (fornecedores, matching documental 2-way/3-way).
- Gestão de notas fiscais e comissões.
- Fechamento mensal com trilha de auditoria e segregação de funções.

## Estrutura

- `docs/arquitetura-conciliacao.md`: documento detalhado com requisitos, modelo de dados, fluxos e roadmap de implementação.
- `docs/metamundi-conciliacao-blueprint.md`: blueprint técnico com esquema de banco, pipeline de processamento, dashboard e lógica da IA conversacional.

## Próximos passos sugeridos

1. Implementar modelo de dados relacional inicial no PostgreSQL.
2. Criar pipeline de ingestão (CSV/OFX/XML NF-e) com validação e padronização.
3. Implementar motor de matching com tolerâncias e fila de pendências.
4. Construir painel web com workspace de conciliação e dashboards.
5. Habilitar trilha de auditoria completa e RBAC com segregação de funções.
