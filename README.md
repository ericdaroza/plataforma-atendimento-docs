# Plataforma de Atendimento — Documentação do Projeto

> Projeto pessoal / empreendedor: plataforma de atendimento e relacionamento omnichannel com IA e estatística, para pequenas e médias empresas, com interface adaptada ao segmento do negócio.
> Iniciado em outubro de 2026.

---

## Sobre este repositório

Repositório **somente de documentação**: visão, pesquisas de mercado, arquitetura e registro de decisões do projeto. Não contém código.

- O código do produto fica em repositórios próprios, ao lado deste em `D:\Projetos`.
- Este repositório é a **fonte única** das decisões e do contexto do projeto. Conversas com assistentes de IA partem destes arquivos, anexados ou com a pasta conectada.

---

## Visão em uma frase

**Nenhum cliente se perde, venha de onde vier** — com uma interface que fala a língua de cada negócio e números que ajudam o empresário a decidir.

---

## Contexto

- **Piloto:** ex-empregador do setor de PPCI, que já testou ferramentas do gênero e as achou caras e difíceis de configurar.
- **Diferenciais pretendidos:** boas práticas de software, segurança, ISO, arquitetura e, principalmente, IA e estatística aplicadas.
- **Ritmo:** sem pressa; desenvolvimento acelerado por multiagentes de LLM, com foco humano em desenho e decisão.

---

## Documentos

| Arquivo | Conteúdo |
|---|---|
| `README.md` | Este índice e o resumo do projeto |
| `decisoes-arquiteturais.md` | Registro de decisões (ADR), com status de cada uma |
| `arquitetura-nucleo-pacotes-dados.md` | Núcleo + pacotes de segmento, modelo de dados, arquitetura hexagonal, multiagentes |
| `ia-estatistica-bi-plataforma.md` | Catálogo de técnicas de IA, estatística e BI por problema, com priorização por fase |
| `dores-plataformas-atendimento.md` | Pesquisa de dores (internacional + Brasil) e síntese dor → diferencial |
| `elogios-plataformas-atendimento.md` | Pesquisa de elogios e funcionalidades amadas, classificação obrigatório × diferencial × encantador |
| `.gitattributes` | Normaliza os `.md` com quebra de linha LF, evitando diffs falsos entre Windows e outros ambientes |

---

## Pilares do produto

1. **Agnóstico de canal** — WhatsApp, Instagram, e-mail, chat do site e futuros canais num núcleo único.
2. **Identidade do cliente** — resolução estatística de quem é quem entre canais; base de todo o resto.
3. **IA em cascata** — regra → estatística → modelo pequeno → LLM; barato, rápido e explicável.
4. **Estatística como produto** — dimensionamento de equipe, previsão de demanda, controle de qualidade, rankings justos, motivos de contato.
5. **Pacotes de segmento** — interface, objetos, funil e recorrências adaptados a cada tipo de negócio.
6. **Confiança operacional** — suporte humano, preço previsível, portabilidade do número, sem bloqueios abruptos.

---

## Principais achados da pesquisa

- A dor mais forte de 2026 é a **IA que bloqueia o acesso a um humano**; a confiança na IA dobra quando há uma opção humana visível.
- O elogio nº 1 da categoria é a **simplicidade**, não a quantidade de funcionalidades.
- No Brasil, as reclamações são majoritariamente de **relação comercial e operação** (suporte, ativação, cobrança, cancelamento), não de funcionalidades.
- O custo da Meta é igual para todos os concorrentes; a margem do produto depende do modelo de integração (Tech Provider × BSP) e do custo de IA.

---

## Questões em aberto

- Construir do zero ou partir do Chatwoot?
- Tech Provider ou BSP?
- Chat web como modo opcional: validar com A/B no piloto.
- Confirmar PPCI como primeiro segmento.
- Definir linguagem e stack principal.

---

## Próximos passos sugeridos

1. Roteiro da conversa de descoberta com o piloto (dores reais, volume, equipe, quanto paga hoje, quanto pagaria).
2. Esboço do pacote PPCI: objetos, recorrências, funil, filas, templates.
3. Escopo do MVP a partir das fases do documento de IA/BI.
4. Definição da stack e das réguas automáticas (testes de arquitetura, validação de esquemas).

---

## Como manter esta documentação

- **Commits feitos pelo Eric**, no Git do Windows. Alterações sugeridas por assistentes de IA são escritas nos arquivos e revisadas com `git diff` antes do commit.
- **Mensagens de commit** no formato `docs: <o que mudou>`.
- **Decisões novas ou alteradas viram ADRs** em `decisoes-arquiteturais.md`; nunca apagar uma decisão antiga, marcar como substituída.
- **Documento novo** entra na tabela de documentos deste README no mesmo commit.
- Atualizar os **pilares**, **questões em aberto** e **próximos passos** deste README sempre que uma ADR mudar de status.
