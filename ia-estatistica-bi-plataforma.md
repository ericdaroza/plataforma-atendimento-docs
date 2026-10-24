# IA, Estatística e BI para a Plataforma de Multiatendimento

> Catálogo de técnicas aplicáveis ao produto, organizado por problema.
> "IA" aqui inclui modelos estatísticos clássicos, aprendizado de máquina pequeno e LLMs, nessa ordem de preferência.
> Documentos relacionados: `dores-plataformas-atendimento.md`, `elogios-plataformas-atendimento.md`.

---

## 0. Princípios de projeto

1. **O modelo mais simples que resolve vence.** Regra, estatística, modelo pequeno e LLM, nessa ordem. Cada degrau a mais custa dinheiro, latência e explicabilidade.
2. **Cascata, não monólito.** Regras resolvem o óbvio, um modelo pequeno resolve o comum e o LLM entra só quando a confiança é baixa. Isso corta o custo de IA em uma ordem de grandeza.
3. **Toda decisão automática precisa ser explicável.** "Por que esta conversa foi para o João?" deve ter resposta legível. Isso também atende a LGPD (art. 20, revisão de decisões automatizadas) e auditorias ISO.
4. **Incerteza é parte da resposta.** Previsões vêm com intervalo; rankings consideram tamanho de amostra; classificações têm limiar de confiança e caminho de fallback.
5. **O uso do produto gera os rótulos.** Quando o atendente reatribui uma conversa, corrige uma etiqueta ou mescla dois contatos, isso é um dado rotulado gratuito. O sistema deve registrar essas ações desde o primeiro dia.
6. **Partida a frio (cold start).** Cada cliente pequeno tem pouco dado. Modelos globais (treinados com dados agregados e anonimizados, com consentimento) calibrados por cliente, e modelos hierárquicos bayesianos, resolvem isso melhor que modelos isolados por cliente.

---

## 1. Resolução de identidade entre canais *(a base do produto)*

**Problema:** a mesma pessoa chega pelo WhatsApp, Instagram, e-mail e chat do site com identificadores diferentes.

### 1.1 Normalização determinística (primeiro, e resolve muito)

- Telefones no padrão E.164 com `libphonenumber`.
- **9º dígito brasileiro:** tratar o número com e sem o 9 após o DDD como candidatos equivalentes. Contas antigas do WhatsApp frequentemente aparecem sem o 9.
- E-mail: minúsculas, remoção de espaços; para Gmail, ignorar pontos e o sufixo `+algo` na parte local.
- Nomes: remover acentos, títulos ("Dr.", "Eng."), espaços duplicados; dicionário de apelidos comuns (Zé → José, Chico → Francisco).
- Documentos (CPF/CNPJ), quando informados pelo próprio cliente: validação por dígito verificador.

### 1.2 Vinculação probabilística de registros (record linkage)

- **Modelo de Fellegi-Sunter:** para cada par de registros, compara campos (telefone, e-mail, nome, @ do Instagram, cidade) e combina as evidências em um peso de vinculação. As probabilidades m (concordância dado que é a mesma pessoa) e u (concordância dado que não é) podem ser estimadas **sem rótulos**, via algoritmo EM.
- **Similaridade de texto:** Jaro-Winkler para nomes, distância de Levenshtein para e-mails com erro de digitação.
- **Blocking:** só comparar pares que compartilham alguma chave (mesmos últimos 8 dígitos do telefone, mesmo domínio de e-mail, mesmas iniciais). Sem isso, a comparação é quadrática.
- **Três zonas de decisão:**
  - confiança alta → mescla automática;
  - confiança média → sugestão ao atendente ("este contato parece ser o mesmo de...");
  - confiança baixa → contatos separados.
- **Grafo de identidades:** registros são nós, vínculos são arestas; componentes conexos formam a pessoa. Restrições impedem mesclas absurdas (ex.: dois CPFs diferentes no mesmo componente).
- **Sinais contextuais:** a mensagem "sou eu, a Maria que mandou e-mail ontem" pode ser capturada por regra ou por um classificador pequeno e vira evidência adicional.
- **Aprendizado contínuo:** cada sugestão aceita ou recusada pelo atendente ajusta os pesos.
- **Bibliotecas:** Splink (Fellegi-Sunter sobre DuckDB/Spark, escala bem), `dedupe`, `recordlinkage`.

**Explicabilidade:** a interface mostra por que dois contatos foram unidos ("mesmo telefone, nome 92% similar, mesma cidade").

---

## 2. Triagem e classificação de intenção

**Problema:** entender na primeira mensagem o que o cliente quer e com que urgência, para mandar à fila certa.

### 2.1 Cascata de classificação

1. **Regras e palavras-chave:** "segunda via", "boleto", "orçamento", "cancelar". Resolve uma fração grande dos casos, custo zero.
2. **Modelo pequeno:** embeddings de sentenças multilíngues compactos (ex.: família `multilingual-e5-small` ou modelos de *sentence-transformers*), com um classificador por cima (regressão logística ou k-vizinhos). Roda em CPU, em milissegundos.
3. **Few-shot:** **SetFit** permite treinar um classificador bom com 8 a 16 exemplos por classe. Ideal para cada cliente definir as próprias filas sem montar uma base grande.
4. **LLM como último recurso:** só quando a confiança do modelo pequeno ficar abaixo do limiar. A resposta do LLM vira rótulo de treino para o modelo pequeno (destilação).

### 2.2 Complementos

- **Detecção de idioma:** fastText (`lid.176`), leve e preciso.
- **Urgência:** classificador binário treinado com sinais como palavras de urgência, repetição de mensagens e intervalos curtos entre mensagens.
- **Aprendizado ativo:** o sistema pede ao gestor que rotule apenas os exemplos em que o modelo está mais incerto. Maximiza o ganho por rótulo.
- **Rótulos gratuitos:** reatribuição de fila pelo atendente = rótulo de erro de roteamento.

---

## 3. Roteamento e atribuição

**Problema:** decidir quem atende cada conversa.

### 3.1 Regras explícitas (MVP)

- **Carteira (dono do contato):** cliente que volta cai com o mesmo vendedor, independentemente do canal. Depende da resolução de identidade.
- **Por habilidade:** idioma, produto, nível técnico.
- **Balanceamento:** menor carga atual (conversas abertas ponderadas por complexidade), não simples rodízio.
- **Disponibilidade:** horário, pausa, férias, com transbordo para o reserva definido.

### 3.2 Otimização (fases posteriores)

- **Problema de atribuição:** com várias conversas e atendentes ao mesmo tempo, o algoritmo húngaro ou programação linear inteira encontram a melhor combinação global.
- **Bandits contextuais (ex.: Thompson Sampling):** aprender qual atendente converte melhor cada tipo de lead, equilibrando exploração e aproveitamento. Exige restrições de **justiça** (todos os vendedores recebem oportunidades) para não concentrar leads e gerar conflito na equipe.
- **Previsão de tempo de atendimento** (regressão simples) para uma carga mais realista.

**Explicabilidade obrigatória:** registro de qual regra ou modelo decidiu a atribuição e por quê.

---

## 4. Previsão de demanda e dimensionamento de equipe

**Problema:** quantas pessoas são necessárias em cada horário. Raramente oferecido para pequenas e médias empresas, muito valorizado em grandes centrais.

### 4.1 Previsão de volume

- Série temporal de conversas por hora, com sazonalidade diária e semanal.
- **Linha de base:** média sazonal ingênua (mesma hora, mesmo dia da semana, últimas 4 semanas). Simples e difícil de bater com pouco dado.
- **Modelos:** ETS/Holt-Winters, MSTL (sazonalidades múltiplas), ou regressão de Poisson/binomial negativa com variáveis de hora, dia da semana, feriados e campanhas.
- **Feriados brasileiros** nacionais, estaduais e municipais (biblioteca `holidays`), além de datas comerciais (Black Friday, Dia das Mães).
- **Campanhas:** um disparo de marketing gera pico previsível de respostas; incluir como variável.
- **Bibliotecas:** `statsforecast` (rápida, AutoETS, MSTL), `statsmodels`.
- Sempre com **intervalo de previsão**.

### 4.2 Dimensionamento

- **Erlang C:** dado volume previsto, tempo médio de atendimento e meta de SLA ("80% respondidos em 5 minutos"), calcula o número de atendentes necessários.
- **Erlang A:** variante que considera **abandono**, mais realista para chat, em que o cliente desiste.
- **Ajuste para concorrência:** no chat, um atendente conduz várias conversas ao mesmo tempo; modelar a capacidade efetiva.
- **Saída para o empresário:** "na segunda entre 9h e 11h você precisa de 3 pessoas; hoje tem 2, então cerca de 30% dos clientes vão esperar mais de 15 minutos."

---

## 5. Tempos, SLA e controle de qualidade da operação

### 5.1 Análise de sobrevivência para tempos

- Tempo até a primeira resposta e até a resolução são **dados censurados**: conversas ainda abertas não têm tempo final. A média simples os ignora e distorce o resultado.
- **Kaplan-Meier** estima corretamente a curva "probabilidade de ainda não ter sido respondido após t minutos".
- **Modelo de Cox** mostra quais fatores (canal, horário, fila, atendente) aceleram ou atrasam a resposta.

### 5.2 Percentis, não médias

- Mostrar P50 e P90. Uma média de 4 minutos pode esconder 10% dos clientes esperando 2 horas.

### 5.3 Controle estatístico de processo (CEP)

- **Gráficos de controle** (X-mR, gráfico p para proporções como taxa de abandono) separam variação normal de mudança real. Evita alarme falso e reação exagerada a um dia ruim.
- **EWMA e CUSUM** detectam deriva gradual (o tempo de resposta piorando 5% por semana).
- Linguagem natural para auditorias ISO 9001 e familiar para quem vem de qualidade.

---

## 6. Vendas, funil e desempenho

### 6.1 Pontuação de leads (lead scoring)

- **Regressão logística** ou **gradient boosting** (LightGBM) prevendo a probabilidade de conversão a partir de: canal de origem, horário, intenção detectada, tempo de primeira resposta, número de mensagens, se veio de anúncio.
- Explicável por coeficientes ou valores SHAP ("lead quente porque veio de anúncio, pediu orçamento e respondeu rápido").
- Ordena a fila do vendedor por valor esperado, não por ordem de chegada.

### 6.2 Ranking justo de atendentes

- Um vendedor com 3 leads e 3 vendas não é "100% de conversão". Rankings ingênuos premiam a sorte de amostras pequenas.
- **Encolhimento bayesiano (empirical Bayes / modelo Beta-Binomial):** puxa taxas com pouca amostra para a média da equipe.
- **Intervalo de Wilson** para exibir a incerteza de cada taxa.

### 6.3 Testes A/B

- Comparar scripts, templates, mensagens de saudação.
- **A/B bayesiano (Beta-Binomial):** responde "qual a probabilidade de B ser melhor que A?", mais intuitivo para o empresário que valor-p, e funciona com amostras menores.

### 6.4 Efeito do tempo de resposta na venda

- Quantificar quanto cada minuto de atraso reduz a conversão. Argumento de venda poderoso.
- **Cuidado com confusão:** leads quentes podem ser respondidos mais rápido por outros motivos. Usar regressão com controles ou métodos de inferência causal (pareamento por escore de propensão) antes de afirmar causalidade.

### 6.5 Segmentação RFM

- Recência, frequência e valor monetário de cada cliente do cliente. Técnica clássica, barata e muito usada em varejo.
- Gera grupos acionáveis: "clientes valiosos que não falam com você há 60 dias".

---

## 7. Descoberta de motivos de contato

**Problema:** o empresário não sabe por que os clientes entram em contato. As ferramentas atuais exigem exportar dados para descobrir.

- **BERTopic:** embeddings → redução de dimensão (UMAP) → agrupamento (HDBSCAN) → palavras representativas (c-TF-IDF). Descobre os temas sem rótulos.
- **LLM apenas para nomear** cada grupo ("Dúvidas sobre prazo de vistoria"), uma chamada por grupo, não por mensagem.
- **Tendências:** volume por tema ao longo do tempo.
- **Detecção de tema emergente:** um grupo novo crescendo rápido ("muita gente perguntando sobre a nova norma") gera alerta.
- Saída: "38% dos contatos deste mês são sobre prazo de entrega; subiu 12 pontos desde o mês passado."

---

## 8. Sugestão de respostas e base de conhecimento

- **Busca híbrida:** BM25 (palavras) + embeddings (significado), sobre respostas antigas bem avaliadas e a base de conhecimento. Resolve a maior parte do valor sem gerar texto.
- **Reordenação (reranking)** com um modelo pequeno de cross-encoder, opcional.
- **LLM para redigir** apenas a partir de trechos recuperados (RAG), com citação da fonte para o atendente conferir.
- **Detecção de respostas repetidas:** agrupar mensagens enviadas pelos atendentes; quando o mesmo texto aparece muitas vezes, sugerir salvá-lo como resposta pronta.
- **Base de conhecimento versionada**, com backup. Nunca apagar conteúdo do cliente numa atualização (dor real relatada no Brasil).

---

## 9. Sentimento, frustração e risco

- **Sentimento em português:** modelo pequeno ajustado (base BERTimbau ou equivalente multilíngue) ou léxico para começar.
- **Sinais de frustração sem texto:** mensagens repetidas, uso de maiúsculas, intervalos curtos, "?" repetidos, pedido explícito de humano. Combinados num escore simples.
- **Alerta ao supervisor** quando uma conversa passa do limiar.
- **Risco de perda do cliente:** análise de sobrevivência sobre o intervalo entre contatos; clientes que "atrasaram" muito em relação ao próprio padrão entram numa lista de reativação.

---

## 10. Anomalias, custo e saúde do canal

- **Volume anômalo:** resíduo de decomposição sazonal (STL) ou intervalo da previsão de Poisson. "Volume 3x acima do esperado para quinta às 14h" pode indicar problema no produto do cliente.
- **Custo da Meta:** previsão do gasto do mês com intervalo, alerta antes de estourar o orçamento. Responde diretamente às "faturas surpresa".
- **Categoria de template:** classificador que prevê se um template será classificado como marketing ou utilidade antes do envio, com sugestões de redação. A diferença de preço é de cerca de 9 vezes.
- **Saúde do número:** monitorar taxa de bloqueios e denúncias, e alertar antes de a Meta reduzir a classificação de qualidade.
- **Falhas silenciosas:** taxa de notificações não entregues, webhooks com erro, filas paradas. Alerta automático em vez de o cliente descobrir sozinho.

---

## 11. Privacidade e segurança com IA

- **Detecção de dados pessoais (PII):** regex com validação (CPF, CNPJ, cartão com Luhn, telefone, e-mail) mais um reconhecedor de entidades (NER) pequeno para nomes e endereços. Microsoft Presidio é uma boa base.
- **Mascaramento** antes de enviar qualquer texto a um LLM externo e antes de gravar em logs.
- **Retenção configurável** e exclusão completa por titular (LGPD), incluindo os índices de embeddings.
- **Registro de decisões automatizadas** (roteamento, mescla de contatos, priorização) para revisão humana quando solicitada.

---

## 12. BI: padrão ouro de grandes empresas, barato desde o início

### 12.1 A decisão mais barata e mais importante: registrar eventos

- Um **log de eventos imutável** (append-only) desde o primeiro dia: `conversa_iniciada`, `mensagem_recebida`, `mensagem_enviada`, `atribuida`, `reatribuida`, `transferida_para_humano`, `etiqueta_alterada`, `contatos_mesclados`, `resolvida`, `reaberta`, `avaliacao_recebida`.
- Todas as métricas e modelos derivam desse log. Métricas novas podem ser calculadas **retroativamente**.
- Estrutura de modelo dimensional (fatos e dimensões) por cima dos eventos.
- **Tecnologia:** PostgreSQL com visões materializadas no início; DuckDB para análise; ClickHouse quando o volume pedir.

### 12.2 Camada semântica de métricas

- Cada métrica definida **uma única vez**, em código, com nome, fórmula e descrição. O painel, o relatório, a API e a IA usam a mesma definição. Evita "o número do painel não bate com o do relatório".

### 12.3 Métricas canônicas de centrais de atendimento

| Métrica | O que mede |
|---|---|
| Tempo de primeira resposta (P50/P90) | Agilidade percebida |
| Tempo até resolução (P50/P90) | Eficiência |
| Resolução no primeiro contato (FCR) | Qualidade |
| Taxa de recontato em 7 dias | Resolução de verdade (não só fechar a conversa) |
| Taxa de abandono | Clientes perdidos na fila |
| Cumprimento de SLA | Promessa cumprida |
| Backlog e idade do backlog | Saúde da fila |
| Ocupação dos atendentes | Equipe sobrecarregada ou ociosa |
| CSAT, NPS e CES (esforço do cliente) | Satisfação e esforço |
| Taxa de automação **com resolução** | IA que resolve, não só que responde |
| Taxa de transferência IA → humano | Limites da automação |
| Conversão por etapa do funil | Vendas |
| Receita atribuída por canal e atendente | Retorno |
| Custo por conversa (Meta + IA) | Eficiência financeira |

### 12.4 Análises de alto valor e baixo custo

- **Mapa de calor hora × dia da semana** do volume e do tempo de resposta.
- **Funil de conversão** por canal de origem.
- **Coortes:** clientes que entraram em cada mês e quantos voltaram.
- **Análise de Pareto** dos motivos de contato (20% dos temas geram 80% do volume).
- **Benchmark anônimo entre clientes** do mesmo segmento: "seu tempo de resposta está no quartil superior dos escritórios de engenharia". Um diferencial que só uma plataforma com vários clientes pode oferecer (exige consentimento e agregação mínima para anonimato).

### 12.5 Do painel à ação

- **Relatório semanal automático em linguagem natural:** os números são calculados de forma determinística e o LLM só redige o texto. Chega por e-mail ou WhatsApp ao dono da empresa. Pequeno empresário não abre painel, mas lê uma mensagem.
- **Alertas proativos** no lugar de painéis que ninguém consulta: "o tempo de resposta de ontem saiu do controle estatístico".
- **Metas com acompanhamento:** "faltam 12% para a meta de SLA do mês; no ritmo atual, você fecha em 91%".

---

## 13. Priorização por fase

| Técnica | Problema | Complexidade | Dados necessários | Fase |
|---|---|---|---|---|
| Log de eventos + camada de métricas | Base de tudo | Baixa | Nenhum | **MVP** |
| Normalização de telefone/e-mail (incl. 9º dígito) | Duplicatas | Baixa | Nenhum | **MVP** |
| Roteamento por regras + carteira | Atribuição | Baixa | Nenhum | **MVP** |
| Métricas canônicas com percentis | Gestão | Baixa | Poucos dias de uso | **MVP** |
| Mapa de calor e funil | Gestão | Baixa | Poucas semanas | **MVP** |
| Detecção e mascaramento de PII | LGPD | Baixa–média | Nenhum | **MVP** |
| Fellegi-Sunter com três zonas | Identidade | Média | Contatos existentes (EM não exige rótulos) | v2 |
| Classificação em cascata + SetFit | Triagem | Média | 8–16 exemplos por fila | v2 |
| Previsão de volume + Erlang C/A | Dimensionamento | Média | 4–8 semanas | v2 |
| Kaplan-Meier, CEP, EWMA | Qualidade | Média | Algumas semanas | v2 |
| Ranking bayesiano de atendentes | Desempenho | Baixa–média | Algumas semanas | v2 |
| Busca híbrida de respostas | Produtividade | Média | Histórico de respostas | v2 |
| Relatório semanal em linguagem natural | Engajamento | Baixa | Métricas prontas | v2 |
| Previsão de custo da Meta | Custo | Baixa | Um mês | v2 |
| BERTopic para motivos de contato | Descoberta | Média | Centenas de conversas | v3 |
| Lead scoring | Vendas | Média | Conversões registradas | v3 |
| Sentimento e frustração | Risco | Média | Opcional | v3 |
| A/B bayesiano | Otimização | Baixa | Volume suficiente | v3 |
| Bandits contextuais com justiça | Atribuição ótima | Alta | Muito histórico | v4 |
| Benchmark entre clientes | Diferencial de rede | Média | Vários clientes e consentimento | v4 |

**Observação:** quase todo o MVP é barato e sem modelos treinados. O que torna os itens das fases seguintes possíveis é ter registrado corretamente os eventos desde o primeiro dia.

---

## 14. Onde o LLM realmente vale a pena

Para fechar o contraponto: o LLM é a melhor ferramenta quando a tarefa envolve **linguagem aberta**.

- Responder dúvidas a partir da base de conhecimento (com RAG e citação).
- Resumir conversas longas na passagem para o humano.
- Redigir rascunhos de resposta para revisão.
- Nomear grupos de temas e escrever o relatório semanal a partir de números já calculados.
- Classificar os casos difíceis que o modelo pequeno não resolveu (e gerar rótulos para ele aprender).

Em todo o resto — identidade, previsão, dimensionamento, rankings, controle de qualidade, custos — a estatística clássica é mais barata, mais precisa e mais explicável.
