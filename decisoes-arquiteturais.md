# Registro de Decisões (ADR)

> Formato: Architecture Decision Records simplificados.
> Cada decisão tem contexto, decisão e consequências. Decisões não se apagam: quando mudam, uma nova ADR substitui a anterior e o status desta passa a "Substituída por ADR-XXX".
>
> **Status possíveis:** Proposta · Aceita · Substituída · Descartada

---

## Índice

| ADR | Título | Status |
|---|---|---|
| 001 | Usar apenas a API oficial do WhatsApp | Aceita |
| 002 | Plataforma agnóstica de canal | Aceita |
| 003 | Resolução de identidade estatística como base do produto | Aceita |
| 004 | Log de eventos imutável desde o primeiro dia | Aceita |
| 005 | PostgreSQL único com separação lógica por esquema | Aceita |
| 006 | Arquitetura hexagonal com testes de arquitetura | Aceita |
| 007 | IA em cascata: regra → estatística → modelo pequeno → LLM | Aceita |
| 008 | Pacotes de segmento orientados a metadados | Aceita |
| 009 | Adiar o motor de pacotes até o segundo segmento | Proposta |
| 010 | PPCI como primeiro segmento | Proposta |
| 011 | Integração com a Meta via Tech Provider | Proposta |
| 012 | Chat web como modo opcional por fila | Proposta |
| 013 | Confiança operacional como diferencial comercial | Proposta |

---

## ADR-001 — Usar apenas a API oficial do WhatsApp

**Status:** Aceita

**Contexto:** muitas ferramentas baratas no Brasil usam APIs não oficiais, que simulam o WhatsApp Web e não pagam a Meta, mas expõem o número a banimento.

**Decisão:** usar exclusivamente a WhatsApp Cloud API oficial.

**Consequências:**
- (+) Compatível com o posicionamento de segurança, ISO e confiabilidade.
- (+) Sem risco de banimento por uso de API não oficial.
- (−) Custo por mensagem da Meta (marketing R$ 0,3217; utilidade, autenticação e serviço R$ 0,035; 1.000 mensagens de serviço gratuitas por número/mês, valores de outubro de 2026).
- (−) Dependência de regras da Meta, que podem mudar até quatro vezes por ano.

---

## ADR-002 — Plataforma agnóstica de canal

**Status:** Aceita

**Contexto:** construir o negócio em torno de "conectar no WhatsApp" é construir em terreno alugado. A Meta muda preços e regras e lançou um agente de IA próprio que concorre com quem constrói sobre ela.

**Decisão:** WhatsApp, Instagram, e-mail, chat do site e futuros canais entram como adaptadores de um núcleo único. O valor do produto fica na camada acima dos canais (identidade, roteamento, IA, estatística).

**Consequências:**
- (+) Resiliência a mudanças de uma única plataforma.
- (+) Promessa de venda clara: nenhum contato se perde, venha de onde vier.
- (−) Cada canal exige um adaptador com suas regras próprias (janelas, limites, formatos).

---

## ADR-003 — Resolução de identidade estatística como base do produto

**Status:** Aceita

**Contexto:** a pesquisa mostrou que contatos duplicados, cliente se repetindo e perda de contexto entre canais são o mesmo problema: saber quem é o cliente. Concorrentes resolvem mal (mescla manual, duplicatas por formato de telefone).

**Decisão:** normalização determinística (incluindo o 9º dígito brasileiro) seguida de vinculação probabilística (Fellegi-Sunter, estimado por EM), com três zonas de decisão: mescla automática, sugestão ao atendente e separação.

**Consequências:**
- (+) Diferencial técnico alinhado à formação em estatística.
- (+) Explicável ao usuário ("unidos porque: mesmo telefone, nome 92% similar").
- (−) Mesclas erradas são graves: exigem desfazer mescla e trilha de auditoria.

---

## ADR-004 — Log de eventos imutável desde o primeiro dia

**Status:** Aceita

**Contexto:** métricas, modelos estatísticos e IA dependem de histórico. Decidir o que medir depois é impossível sem dados brutos.

**Decisão:** registrar todas as ações relevantes como eventos append-only (`mensagem_recebida`, `atribuida`, `reatribuida`, `contatos_mesclados`, `resolvida`, `reaberta`...). O log é o contrato entre o operacional e o analítico.

**Consequências:**
- (+) Métricas novas calculáveis retroativamente.
- (+) Correções dos atendentes viram rótulos gratuitos para modelos.
- (+) Facilita separar o analítico em outro banco no futuro.
- (−) Volume de armazenamento; exige particionamento e política de retenção compatível com a LGPD.

---

## ADR-005 — PostgreSQL único com separação lógica por esquema

**Status:** Aceita

**Contexto:** há três naturezas de dado (núcleo relacional, objetos flexíveis de segmento, analítico). Usar três tecnologias desde o início multiplicaria consistência, LGPD, segurança, multi-tenant e operação para uma pessoa só.

**Decisão:** um único PostgreSQL com esquemas `core`, `segmento` (JSONB) e `analytics`. Extensões: `pg_trgm`, `pgvector`, Row-Level Security; opcionalmente `pg_jsonschema` e TimescaleDB. Separação física só diante de problema real medido.

**Regra:** o que o núcleo consulta em todos os segmentos vira coluna relacional; o que só a tela do segmento lê pode ficar no JSONB. Evitar EAV.

**Consequências:**
- (+) Transações únicas, integridade referencial, operação simples.
- (+) Caminho de evolução claro: réplica de leitura → OLAP via CDC/outbox → armazenamentos especializados.
- (−) Exige disciplina para não cruzar fronteiras entre esquemas.

---

## ADR-006 — Arquitetura hexagonal com testes de arquitetura

**Status:** Aceita

**Contexto:** a evolução de banco único para vários bancos, e de poucos para muitos canais, só é barata se o domínio estiver isolado da infraestrutura. O maior risco é a degradação gradual por atalhos, inclusive de agentes geradores de código.

**Decisão:** Ports & Adapters. O domínio não conhece tecnologias. Fronteiras verificadas por testes automatizados que quebram o build (ArchUnit, `import-linter` ou `dependency-cruiser`, conforme a stack).

**Consequências:**
- (+) Trocar ou adicionar banco/canal = novo adaptador.
- (+) Régua objetiva para o agente de arquitetura na estratégia multiagentes.
- (−) Mais código de mapeamento entre domínio e infraestrutura.

---

## ADR-007 — IA em cascata

**Status:** Aceita

**Contexto:** LLM para tudo é caro, lento e pouco explicável. Muitas tarefas (identidade, previsão, dimensionamento, rankings, controle de qualidade) são resolvidas melhor por estatística clássica.

**Decisão:** preferência na ordem regra → estatística → modelo pequeno → LLM. LLM reservado para linguagem aberta (respostas com RAG, resumos, rascunhos, nomear temas, redigir relatórios a partir de números calculados) e para casos de baixa confiança, cujas respostas viram rótulos para o modelo pequeno.

**Consequências:**
- (+) Custo de IA por cliente muito menor, melhor margem.
- (+) Decisões explicáveis (LGPD art. 20, auditorias).
- (−) Mais componentes para manter e monitorar.

Detalhamento: `ia-estatistica-bi-plataforma.md`.

---

## ADR-008 — Pacotes de segmento orientados a metadados

**Status:** Aceita

**Contexto:** a interface deve se adaptar ao segmento (PPCI, oficina, barbearia...), mas código próprio por segmento levaria a manter N produtos.

**Decisão:** núcleo único; cada segmento é um pacote declarativo (tipos de objeto com JSON Schema, mapeamento para entidades do núcleo, funil, filas, templates, regras de recorrência, indicadores, componentes de tela), renderizado por interface guiada por esquema.

**Consequências:**
- (+) Segmento novo = configuração + poucos componentes.
- (+) Pacotes geráveis e validáveis por agentes.
- (−) O núcleo precisa de entidades genéricas bem pensadas (ex.: `Vencimento`).

---

## ADR-009 — Adiar o motor de pacotes até o segundo segmento

**Status:** Proposta

**Contexto:** abstrações criadas antes de dois ou três casos concretos tendem a sair erradas.

**Decisão proposta:** construir o núcleo genérico desde o início, mas implementar o primeiro segmento sem o motor de pacotes. Extrair o formato de pacote ao implementar o segundo.

---

## ADR-010 — PPCI como primeiro segmento

**Status:** Proposta (validar na conversa de descoberta com o piloto)

**Contexto:** conhecimento de domínio prévio em PPCI, cliente piloto disponível, forte recorrência (recarga de extintores, renovação de alvarás), pouca atenção de grandes fornecedores.

**Decisão proposta:** PPCI primeiro; depois, serviços técnicos semelhantes (ar-condicionado, dedetização, elevadores, segurança eletrônica). Posicionamento em avaliação: "atendimento e relacionamento para empresas de serviços técnicos".

---

## ADR-011 — Integração com a Meta via Tech Provider

**Status:** Proposta

**Contexto:** BSPs revendem mensagens com margem e carregam o risco de crédito. Reclamações brasileiras mostram faturas surpresa e números presos ao fornecedor após o cancelamento.

**Decisão proposta:** modelo Tech Provider: a conta da Meta fica no nome do cliente, que paga a Meta diretamente (faturamento em reais disponível desde julho de 2026). A receita do produto é a assinatura do software.

**Consequências esperadas:**
- (+) Sem risco de crédito nem revenda de mensagens.
- (+) Portabilidade do número garantida: argumento comercial.
- (−) O cliente precisa cadastrar forma de pagamento na Meta (obrigatório para mensagens de serviço desde outubro de 2026): incluir no onboarding.

---

## ADR-012 — Chat web como modo opcional por fila

**Status:** Proposta

**Contexto:** usar o WhatsApp só como porta de entrada e levar o atendimento a um chat web reduz o custo da Meta, mas pode aumentar o abandono e exige solução para notificar o cliente.

**Decisão proposta:** suportar os dois modos, configuráveis por fila (ex.: suporte técnico → chat web; vendas → WhatsApp). Medir abandono e conversão com teste A/B no piloto antes de recomendar um padrão.

---

## ADR-013 — Confiança operacional como diferencial comercial

**Status:** Proposta

**Contexto:** no Brasil, as reclamações sobre concorrentes são majoritariamente de relação comercial e operação: suporte por bot em loop, ativação travada na Meta, faturas surpresa, cancelamento difícil, número preso, operação bloqueada por inadimplência, falhas silenciosas.

**Decisão proposta:** tratar como requisitos de produto, não só de atendimento:
- suporte humano acessível em todos os planos;
- assistente de ativação na Meta com diagnóstico em linguagem simples;
- teste gratuito real;
- custo em tempo real, previsão de fatura e alerta de orçamento;
- cancelamento em um clique e portabilidade do número;
- carência e modo somente leitura antes de qualquer bloqueio;
- página de status, alertas de falha, backup e versionamento da base de conhecimento.
