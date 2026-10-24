# Arquitetura: Núcleo, Pacotes de Segmento e Dados

> Registro das ideias de arquitetura discutidas em outubro de 2026.
> Status: **proposta em construção**. Decisões consolidadas estão em `decisoes-arquiteturais.md`.

---

## 1. Visão

Uma plataforma de atendimento e relacionamento **agnóstica de canal** (WhatsApp, Instagram, e-mail, chat do site e outros), cuja interface do lado da empresa **se adapta ao segmento do negócio**.

Em vez de uma interface genérica que tenta servir a todos, cada segmento (PPCI, oficina, barbearia, restaurante...) recebe um **pacote** com seus objetos, linguagem, funil, filas, templates, regras de recorrência e indicadores, todos rodando sobre o **mesmo núcleo**.

No mercado, esse padrão é conhecido como **vertical SaaS sobre núcleo horizontal**.

---

## 2. Por que pacotes de segmento

| Benefício | Ligação com a pesquisa |
|---|---|
| Configuração em minutos: o cliente escolhe o segmento e já começa com tudo pronto | Resolve a dor nº 1 (configuração difícil) |
| Linguagem do negócio na interface | Elogio nº 1 da categoria: simplicidade |
| Verticalização sem perder escala | Menos concorrência, maior disposição a pagar |
| IA treinada por segmento | Resolve a partida a frio de clientes novos |
| Recorrência conhecida por segmento | Atendimento passa de reativo a gerador de receita |

### 2.1 Recorrência: o maior valor dos pacotes

Cada segmento tem eventos previsíveis que o sistema pode antecipar:

| Segmento | Objetos do negócio | Recorrência que gera venda |
|---|---|---|
| PPCI | Edificação, extintor, alvará, projeto, vistoria | Recarga anual de extintores, renovação do alvará dos bombeiros, manutenção de hidrantes |
| Oficina | Veículo, serviço, peça | Troca de óleo por quilometragem ou tempo, revisões |
| Barbearia | Cliente, serviço, profissional, horário | Retorno no intervalo habitual do cliente |
| Restaurante | Cardápio, pedido, reserva | Cliente frequente sumido, datas comemorativas |

Lembretes de recorrência são templates de **utilidade** na Meta (R$ 0,035 por mensagem no Brasil), não de marketing (R$ 0,3217): baratos e de alto valor.

---

## 3. Arquitetura orientada a metadados

### 3.1 O risco a evitar

Se cada segmento ganhar código próprio, o resultado é manter N produtos diferentes. Os pacotes devem ser **configuração**, não código.

### 3.2 Núcleo (igual para todos os segmentos)

- Contatos e **resolução de identidade** entre canais
- Conversas, mensagens e adaptadores de canal
- Roteamento, filas e carteira (dono do contato)
- **Motor de recorrência e vencimentos**
- Log de eventos imutável
- Métricas, estatística e IA genéricas
- Usuários, permissões, multi-tenant, auditoria, LGPD

### 3.3 Pacote de segmento (declarativo)

Cada pacote declara:

- **Tipos de objeto** do negócio, com esquema (JSON Schema). Ex.: `Extintor` com tipo, capacidade, agente extintor, localização e data de validade.
- **Mapeamento para o núcleo:** quais campos alimentam entidades genéricas (ex.: a validade do extintor gera um `Vencimento`).
- **Etapas do funil** e **filas / intenções** padrão.
- **Templates de mensagem** (com categoria prevista na Meta).
- **Regras de recorrência** (ex.: recarga a cada 12 meses; aviso 30 dias antes).
- **Indicadores** específicos do segmento.
- **Componentes de interface** a exibir (cartão do extintor, agenda, cardápio).

### 3.4 Interface guiada por esquema

O painel lê o pacote e monta a tela com os componentes declarados (*server-driven UI*). Um segmento novo vira um arquivo de configuração e, eventualmente, alguns componentes visuais novos.

Esse formato bem definido também favorece o desenvolvimento com multiagentes: um agente pode gerar um pacote novo a partir de uma especificação e ser validado contra o esquema de pacotes.

### 3.5 Sequência recomendada

1. **Começar com um único segmento, em profundidade** (proposta: PPCI, pelo conhecimento de domínio e pelo piloto disponível).
2. **Construir o núcleo genérico desde o início**, mas **não construir o motor de pacotes ainda**. Abstrações criadas antes de dois ou três casos concretos tendem a sair erradas.
3. **No segundo segmento, extrair o padrão** do que realmente varia e transformá-lo no formato de pacote.
4. **Priorizar segmentos parecidos com PPCI:** serviços técnicos com recorrência e venda consultiva (manutenção de ar-condicionado, dedetização, elevadores, segurança eletrônica).
5. **Em segmentos com concorrentes verticais fortes** (barbearia, restaurante), integrar com as ferramentas existentes em vez de competir.

**Posicionamento em avaliação:** "atendimento e relacionamento para empresas de serviços técnicos".

---

## 4. Dados

### 4.1 Três naturezas de dado

| Natureza | Exemplos | Necessidade |
|---|---|---|
| Núcleo estável | Contatos, conversas, usuários, filas | Integridade, transações, relacionamentos |
| Específico de segmento | Extintores, cardápios, serviços | Esquema flexível por pacote |
| Analítico / telemetria | Eventos, métricas, séries temporais | Consultas agregadas sobre grande volume |

### 4.2 Decisão: separação lógica agora, física depois

Começar com **um único PostgreSQL**, com as três naturezas separadas por esquema:

- `core` — tabelas relacionais do núcleo
- `segmento` — objetos dos pacotes em colunas `JSONB`
- `analytics` — log de eventos e visões materializadas

**Por que não três bancos desde o início:**

- Consistência entre bancos exigiria transações distribuídas ou padrões como outbox e compensação.
- LGPD: exclusão de titular em três lugares, com comprovação.
- Segurança / ISO 27001: três superfícies de ataque, três políticas de backup e acesso.
- Multi-tenant: isolamento garantido três vezes.
- Operação multiplicada para uma pessoa só.

### 4.3 Recursos do PostgreSQL aproveitados

| Recurso | Uso |
|---|---|
| `JSONB` + índices GIN | Objetos flexíveis dos pacotes |
| `pg_jsonschema` (opcional) | Validação do esquema do pacote no banco |
| `pg_trgm` | Similaridade de texto na resolução de identidade |
| `pgvector` | Embeddings para classificação e busca de respostas |
| Row-Level Security | Isolamento entre clientes (multi-tenant) no próprio banco |
| Particionamento por tempo + índice BRIN | Tabela de eventos de alto volume |
| Visões materializadas | Métricas e painéis |
| TimescaleDB (se necessário) | Séries temporais |

### 4.4 Regra de ouro do modelo flexível

> O que o núcleo precisa consultar em todos os segmentos vira coluna relacional.
> O que só a tela do segmento lê pode ficar no `JSONB`.

Exemplo: a validade do extintor parece específica de PPCI, mas o motor de recorrência precisa consultá-la. Por isso existe a entidade genérica **`Vencimento`** (objeto de origem, data, tipo de recorrência, status), preenchida pelo pacote PPCI a partir do extintor, pelo de oficina a partir da troca de óleo e pelo de barbearia a partir do último atendimento. Atributos que só a tela usa (capacidade, agente extintor, localização na planta) ficam no `JSONB`.

**Evitar** o modelo EAV (entidade-atributo-valor): flexível na aparência, lento e difícil de consultar na prática.

### 4.5 Disciplina de modelagem

- Nenhuma consulta cruza fronteiras de esquema fora dos pontos previstos.
- O código analítico não faz JOIN direto com tabelas do núcleo; consome o log de eventos.
- O log de eventos é **imutável** (append-only) e é o **contrato** entre o operacional e o analítico.

### 4.6 Caminho de evolução

1. **Fase 1:** só PostgreSQL, com os três esquemas e o log de eventos.
2. **Fase 1.5:** réplica de leitura para relatórios, tirando a carga analítica da produção.
3. **Fase 2:** replicar eventos para um banco OLAP (ClickHouse) ou para Parquet + DuckDB, via CDC (Debezium) ou padrão outbox. O analítico vira **cópia derivada**, sem problema de consistência.
4. **Fase 3, só se necessário:** busca dedicada (OpenSearch) ou outro armazenamento especializado.

O gatilho para cada fase é um **problema real medido**, não antecipação.

---

## 5. Arquitetura hexagonal (Ports & Adapters)

### 5.1 Princípios

- **O domínio não sabe** que existem JSONB, SQL, PostgreSQL, WhatsApp ou Instagram. Ele conhece `Contato`, `Conversa`, `Extintor`, `Vencimento`.
- **Portas** são interfaces definidas pelo domínio (ex.: `RepositorioDeContatos`, `CanalDeMensagens`).
- **Adaptadores** implementam as portas para cada tecnologia (PostgreSQL, WhatsApp Cloud API, Instagram, e-mail).

### 5.2 Adaptadores de canal

- Um **modelo de mensagem único** interno.
- Um **adaptador por canal** traduz de e para cada plataforma, incluindo suas regras (janela de 24h do WhatsApp, limitações do Instagram, ausência de janela no e-mail).
- Canal novo (Telegram, RCS, telefone, Reclame Aqui) = adaptador novo, sem mexer no núcleo.
- Mudança de regra da Meta = troca de um adaptador, não do produto.

### 5.3 Proteção contra degradação

O maior risco não é desenhar errado, e sim a arquitetura se degradar com atalhos ao longo do tempo.

- **Testes de arquitetura automatizados** que quebram o build se o domínio importar infraestrutura:
  - Java: ArchUnit
  - Python: `import-linter`
  - TypeScript: `dependency-cruiser`
- Integram-se à estratégia de **multiagentes**: o agente de arquitetura passa a ter uma régua objetiva, e atalhos de agentes geradores de código são barrados automaticamente.

Com essas proteções, separar bancos no futuro vira "escrever um adaptador e migrar os dados", não "refatorar o sistema".

---

## 6. Estratégia de desenvolvimento com multiagentes

Ideia registrada: acelerar o desenvolvimento com agentes de LLM especializados, cada um responsável por um aspecto (segurança, testes, arquitetura, cobertura de critérios de aceite etc.), deixando o foco humano no desenho e na tomada de decisão.

Diretrizes já discutidas:

- Decisões de **segurança e arquitetura** sempre passam pela revisão humana. Agentes seguem critérios bem; não decidem quais critérios importam.
- Fronteiras verificáveis automaticamente (testes de arquitetura, validação de esquema de pacotes) dão aos agentes réguas objetivas.
- Sem pressa: o ritmo permite revisar cada etapa.

---

## 7. Questões em aberto

- Construir do zero ou partir do Chatwoot (open source) e investir na camada de IA, analytics e experiência?
- Modelo de integração com a Meta: Tech Provider (cliente paga a Meta diretamente) ou BSP?
- Modo de chat web (WhatsApp só como porta de entrada) como opção por fila: validar com teste A/B no piloto.
- Confirmar PPCI como primeiro segmento e o posicionamento "serviços técnicos" após a conversa de descoberta com o piloto.
- Linguagem e stack principal (define as ferramentas de teste de arquitetura).
