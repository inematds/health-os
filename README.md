# Health OS

> 🌐 [English](./README.en.md) · **Português**

Um blueprint completo e pronto para uso de um **coach de saúde pessoal com IA**: um agente no Telegram que lembra de tudo no seu próprio banco de dados Supabase, lê seu wearable (WHOOP) pela API oficial, faz uma revisão matinal guiada pela recuperação e fundamenta cada resposta nos seus exames de sangue, genética, composição corporal e metas reais.

Clone, aponte para o seu próprio projeto Supabase e token de bot, percorra o checklist abaixo e você terá um coach que revisa o dia anterior, lê a recuperação da última noite e te diz se hoje é dia de avançar ou recuar.

> ⚠️ **Não é aconselhamento médico.** Isto é um blueprint de software, não orientação de saúde. Tudo aqui é simplesmente o que funcionou para uma pessoa que consultou seus médicos a cada passo. Não é uma prescrição, diagnóstico ou recomendação para você. Converse com profissionais de saúde qualificados antes de mudar qualquer coisa nos seus exames, suplementos, treino ou dieta. O coach de IA pode errar ou alucinar, então trate qualquer coisa que ele sugerir como uma pergunta para levar ao seu médico, nunca como uma instrução a seguir.

![Mapa do sistema Health OS](./docs/health-os-schematic.png)

> Tudo neste repositório é genérico ou um exemplo. Não contém **nenhuma informação pessoal de saúde**. Traga seus próprios dados, bot e projeto Supabase. Mantenha o seu `CLAUDE.md` preenchido, seus valores de seed e seu `~/.env` fora do controle de versão.

---

## O que você recebe

- **Um coach de saúde no Telegram** fundamentado nos seus dados, não em conselhos genéricos.
- **Seu próprio banco de dados Supabase** (schema completo em `agent/supabase/migrations/`): alimentação, treinos, pesagens, composição corporal, cafeína, suplementos, sinais vitais, exames, check-ins diários, metas, contexto, além de memória semântica das mensagens.
- **Um blueprint de conexão com o WHOOP** de ponta a ponta: app OAuth, o redirect/callback, o sync diário, o mapeamento JSON-para-banco e os dois detalhes de produção que pegam todo mundo (user-agent do Cloudflare, rotação do refresh token).
- **Uma revisão matinal guiada pela recuperação** que liga a comida/cafeína/treino de ontem a como você recuperou, e condiciona o plano de hoje a esse número.
- **Fluxos com foto** (comida -> macros, exame escaneado -> marcadores, foto do corpo -> treino).
- **O dashboard ao vivo** (`agent/dashboard/`): exatamente o visual dark, lendo direto do Supabase.
- **Todos os diagramas** gerados para explicar o sistema, embutidos ao longo do texto.

## Um dia na vida

Um dia típico, todo no Telegram. Exemplo ilustrativo, não são números reais, e não é aconselhamento.

- **07:00** dispara o check-in matinal: "Recuperação 71% (verde), dormiu 7h20m, HRV 64, FC de repouso 52. A proteína de ontem bateu a meta e a cafeína foi cedo e moderada, essa rodada limpa é o motivo de você estar verde. Hoje é dia de avançar, coloque os carboidratos em torno do treino. Peso desta manhã? Alguma PA?"
- **08:30** você fotografa o café da manhã: "~38g de proteína, baixa gordura saturada, registrado." (grava uma linha em `food_log`)
- **13:00** você pergunta "café agora?": "Você está verde e ainda é cedo, tudo bem, mantenha abaixo do seu teto e corte no meio da tarde."
- **19:00** "bife e uma taça de vinho": registrado; o coach observa que o álcool pode prejudicar a recuperação desta noite.
- **Na manhã seguinte** o ciclo se fecha: "Recuperação caiu para 48% âmbar, o vinho e a refeição tardia são a causa provável. Pegue leve hoje, aposte em proteína, fibra e hidratação."

Cada número acima é ilustrativo. As respostas do seu coach são tão boas quanto o perfil que você dá a ele, e devem sempre ser verificadas com o seu médico.

---

---

## Arquitetura

O agente lê um **snapshot da sessão** compacto no início de cada turno (tendência de peso, ingestão de hoje, PA, recuperação da última noite, o padrão de sono dos 7 dias, metas), então sempre responde a partir do contexto atual. Os dados entram a partir do wearable, das fotos de comida e dos registros manuais para o Supabase; o coach raciocina sobre eles, fundamentado nos seus exames e genética; e entrega a revisão matinal, o dashboard, conselhos com fonte citada e cronogramas de suplementos.

![Modelo de dados do Health OS](./docs/health-os-data-model.png)

A revisão matinal guiada pela recuperação que o coach executa todo dia:

![A revisão matinal guiada pela recuperação](./docs/health-os-morning-review.png)

Veja o mapa completo acima. A estrutura do repositório:

```
health-os-private/
├── README.md                  ← você está aqui (visão geral + o checklist de testes)
├── docs/
│   ├── BUILD_GUIDE.md         ← build passo a passo, de ponta a ponta
│   ├── health-os-schematic.png
│   ├── whoop-1-setup.png
│   └── whoop-2-data.png
└── agent/                     ← o agente autocontido e higienizado
    ├── CLAUDE.md              ← o cérebro do coach (template, preencha seu perfil)
    ├── agent.yaml.example     ← config do bot + slash commands
    ├── AGENTS.md
    ├── scripts/               ← estado, memória, db, WHOOP, suplementos, conselhos, ...
    ├── supabase/migrations/   ← o schema completo + seed de exemplo
    └── dashboard/             ← o dashboard web ao vivo (página + camada de dados + rotas)
```

---

## A conexão com o WHOOP (o blueprint principal)

### Configuração única
![Fluxo de configuração do WHOOP](./docs/whoop-1-setup.png)

### Caminho dos dados: JSON da API até o coach
![Caminho dos dados do WHOOP](./docs/whoop-2-data.png)

Walkthrough completo em [`docs/BUILD_GUIDE.md`](./docs/BUILD_GUIDE.md). A versão curta: crie um app WHOOP, registre `https://<seu-host>/whoop/callback`, habilite `read:recovery read:sleep read:cycles offline`, autorize uma vez, e um cron diário (`agent/scripts/whoop-sync.py`) puxa recuperação + sono para a tabela `vitals`. Ele rotaciona o refresh token a cada execução e envia um user-agent de navegador (o Cloudflare bane o padrão do Python).

---

## Variáveis de ambiente

Todo segredo vive em `~/.env` (seu diretório home, nunca um `.env` de projeto, nunca commitado):

| Variável | Finalidade |
|---|---|
| `HEALTH_BOT_TOKEN` | Token do bot do Telegram (do @BotFather) |
| `SUPABASE_URL` | `https://<project-ref>.supabase.co` |
| `SUPABASE_SERVICE_ROLE_KEY` | Acesso ao DB no lado do servidor (ignora o RLS) |
| `SUPABASE_ANON_KEY` | Chave pública (com RLS, não lê nada) |
| `SUPABASE_DB_PASSWORD` | Para as migrations do CLI `supabase` |
| `OPENAI_API_KEY` | Embeddings para a memória semântica |
| `GOOGLE_API_KEY` | Visão para fotos de comida/exames + clipes de treino (Gemini) |
| `DASHBOARD_TOKEN` | Protege o dashboard web |
| `WHOOP_CLIENT_ID`, `WHOOP_CLIENT_SECRET` | Credenciais do seu app WHOOP |
| `WHOOP_REFRESH_TOKEN` | Escrito pelo callback OAuth, rotacionado a cada sync |

---

## Início rápido

1. Provisione um projeto Supabase privado; faça o push de `agent/supabase/migrations/`.
2. Crie um bucket de storage privado `health-assets`.
3. Preencha `~/.env` (chaves do Supabase, `OPENAI_API_KEY`, `GOOGLE_API_KEY`, seu `HEALTH_BOT_TOKEN` e `WHOOP_CLIENT_ID/SECRET`).
4. Crie um bot do Telegram via @BotFather; defina `telegram_bot_token_env` em `agent.yaml`.
5. Copie `CLAUDE.md` e preencha com o seu próprio perfil.
6. Conecte o WHOOP (OAuth único). Agende o sync + o check-in matinal.

Detalhes completos: [`docs/BUILD_GUIDE.md`](./docs/BUILD_GUIDE.md).

---

## Checklist de validação

Trabalhe de cima para baixo. Marque cada caixa assim que verificar pessoalmente no seu próprio setup.

### Supabase + schema
- [ ] Projeto Supabase privado provisionado em uma região apropriada
- [ ] Extensão `pgvector` habilitada (a migration `0001_init.sql` faz isso)
- [ ] Todas as migrations aplicadas; todas as 14 tabelas existem
- [ ] RLS habilitado em todas as tabelas, sem policies (a chave anon não lê nada)
- [ ] Bucket de storage privado `health-assets` criado
- [ ] Seed de exemplo substituído pelas suas próprias metas + contexto (`0002_seed_example.sql`)
- [ ] `scripts/db.py select goals` retorna as suas linhas

### Agente + Telegram
- [ ] Bot criado via @BotFather, token em `~/.env`
- [ ] O agente sobe e responde a uma mensagem comum no Telegram
- [ ] Slash commands aparecem no menu "/" (`/checkin`, `/today`, `/sofar`, `/newday`, `/supplements`, `/advice`)
- [ ] `CLAUDE.md` preenchido com o seu perfil real (meta, exames de sangue, genética, restrições)
- [ ] O snapshot do `state.py` imprime seu peso, ingestão e metas

### Registro (logging)
- [ ] Uma foto de comida é lida e gravada em `food_log` com macros + flags
- [ ] Uma mensagem de peso grava uma linha em `weigh_ins`
- [ ] Uma mensagem de treino / cafeína / suplemento grava a sua linha
- [ ] Uma leitura de PA grava uma linha em `vitals`
- [ ] Um escaneamento de exame extrai marcadores para `lab_results`
- [ ] A recuperação semântica (`mem.py recall`) retorna mensagens passadas relevantes

### WHOOP
- [ ] App de desenvolvedor WHOOP criado; escopos habilitados
- [ ] URI de redirect registrada exatamente e salva
- [ ] OAuth único concluído; `WHOOP_REFRESH_TOKEN` escrito em `~/.env`
- [ ] `whoop-sync.py` roda e grava `recovery_pct`, `hrv_ms`, `resting_hr`, `sleep_hours`
- [ ] Rodar o sync de novo é idempotente (sem linhas duplicadas para um dia)
- [ ] O cron está agendado e dispara de manhã
- [ ] Rotação do token verificada (uma segunda execução ainda funciona, sem `invalid_grant`)
- [ ] User-agent de navegador confirmado (sem erro 1010 do Cloudflare)

### Comportamento de coaching
- [ ] O check-in matinal dispara e abre com a recuperação da última noite
- [ ] O check-in liga as escolhas de ontem ao número da recuperação
- [ ] O plano de hoje é visivelmente condicionado à recuperação (verde = avançar, vermelho = pegar leve)
- [ ] Um "devo comer isso dado como dormi" sob demanda puxa a recuperação e lidera com ela
- [ ] `coach_summary` registra a recuperação + a sua causa (o histórico se acumula)
- [ ] A tendência de sono/recuperação dos 7 dias aparece no snapshot

### Opcional
- [ ] Dashboard de tendências ao vivo acessível e mostrando o card de Sono & Recuperação
- [ ] Geração do clipe de demonstração de treino funciona (`exercise_clip.py`)
- [ ] O RAG de conselhos de influenciadores retorna dicas com fonte citada, reconciliadas com os seus dados

---

## O painel em torno do qual este sistema é construído

O coach é mais útil quando fundamentado em uma linha de base abrangente. Este é o conjunto completo de **marcadores de sangue** e **SNPs de DNA** que o design acompanha, o mesmo painel sobre o qual o schema, as risk flags e as metas são modelados. Uma caixa por exame, para você ir marcando conforme os solicita. Apenas os nomes, sem valores ou genótipos; os seus próprios resultados vivem nas suas linhas privadas de `lab_results` e no seu `CLAUDE.md` preenchido.

### Marcadores de sangue

**Cardiometabólicos**
- [ ] LDL-C
- [ ] ApoB
- [ ] ApoA-1
- [ ] Lp(a)
- [ ] hs-CRP
- [ ] HOMA-IR (glicose de jejum + insulina)

**Hormônios**
- [ ] Testosterona, total
- [ ] Testosterona, livre
- [ ] SHBG
- [ ] Estradiol
- [ ] DHEA-S
- [ ] Pregnenolona
- [ ] Cortisol (manhã)

**Tireoide**
- [ ] T3 reverso
- [ ] Painel tireoidiano (TSH, T3 livre, T4 livre)

**Fígado**
- [ ] ALT
- [ ] AST

**Metilação**
- [ ] Homocisteína

**Vitaminas + minerais**
- [ ] Vitamina D (25-OH)
- [ ] Magnésio
- [ ] Zinco
- [ ] Cobre
- [ ] Selênio

**Base (foundation)**
- [ ] Painel lipídico completo
- [ ] Hemograma completo (CBC)
- [ ] Painel metabólico abrangente

### SNPs de DNA

**Lipídios + cardiovascular**
- [ ] APOE
- [ ] LPA
- [ ] PCSK9
- [ ] CETP
- [ ] ACE
- [ ] AGT
- [ ] NOS3 (eNOS)

**Metilação / vitaminas do complexo B**
- [ ] MTHFR
- [ ] MTHFD1
- [ ] MTR
- [ ] MTRR
- [ ] CBS

**Detox (Fase II)**
- [ ] GSTM1
- [ ] GSTT1
- [ ] GSTP1

**Cafeína + neurotransmissores**
- [ ] COMT
- [ ] CYP1A2

**Vitamina D**
- [ ] VDR
- [ ] CYP2R1
- [ ] GC

**Metabólico / peso corporal**
- [ ] FTO
- [ ] PPARG
- [ ] TCF7L2

**Inflamação / antioxidante**
- [ ] TNF
- [ ] SOD2
- [ ] GPX1

**Outros**
- [ ] HFE (manejo do ferro)
- [ ] TAS2R38 (paladar / sensibilidade ao amargo)

Cada marcador ou SNP mapeia para uma risk flag, um suplemento ou uma meta no schema. É assim que o coach dá conselhos cientes do mecanismo em vez de dicas genéricas.

---

## Por que é construído assim

- **Específico ganha do genérico.** Um coach fundamentado nos seus exames, genética e metas reais dá conselhos cientes do mecanismo; um bot genérico dá platitudes. Todo o design força a especificidade.
- **A memória é o produto.** Tudo é gravado em linhas estruturadas, então o coach conhece todo o seu histórico e consegue enxergar padrões ao longo de semanas, não só reagir à última mensagem.
- **A recuperação é a espinha do dia.** Abrir cada dia com como você de fato recuperou, e então ligar isso ao que você fez, te ensina as suas próprias alavancas.
- **Os dados do dono sempre vencem.** Qualquer dica externa é reconciliada com os seus próprios números.
- **Trancado por padrão.** Projeto privado, service-role no lado do servidor, segredos em `~/.env`, fora do git.
- **Não é um médico.** É uma ferramenta de acompanhamento e raciocínio que te encaminha a profissionais de verdade para qualquer coisa clínica.

---

## FAQ

**Isto é aconselhamento médico?** Não, veja "Importante: não é aconselhamento médico" acima. É um blueprint de software; verifique tudo com os seus próprios médicos.

**Eu preciso de um WHOOP?** Não. O WHOOP é o exemplo trabalhado, mas o mesmo padrão de OAuth + sync diário serve para Oura, Garmin, Fitbit ou registro manual. O coach e o dashboard funcionam com o que quer que chegue na tabela `vitals`.

**Quanto custa rodar?** O tier gratuito do Supabase dá conta de uma pessoa. Você paga pelas chamadas ao LLM, embeddings (centavos) e a visão do Gemini para fotos. A API do WHOOP é gratuita com uma assinatura.

**Onde meus dados ficam?** No seu próprio projeto Supabase privado, trancado (service-role no lado do servidor, RLS ligado sem policies). Nada sai exceto as chamadas ao LLM que você escolher fazer.

**Pode diagnosticar ou mudar minha medicação?** Não, e ele é explicitamente instruído a não fazer isso. Ele sinaliza preocupações clínicas em direção a um médico e nunca toca em medicações.

**Quão precisos são os macros das fotos de comida?** São estimativas de visão, boas para acompanhar tendências, não um substituto para pesar a comida. O coach avisa quando está chutando, e você ainda deve passar qualquer coisa que importe por um profissional.

---

## Importante: não é aconselhamento médico

Este repositório é um **blueprint de software** para construir um assistente pessoal de acompanhamento de saúde. **Não** é aconselhamento médico, nutricional ou de fitness, e nada nele é uma recomendação para você.

- **É a experiência de uma pessoa.** Cada meta, marcador, suplemento e hábito referenciado aqui é um exemplo do que funcionou para o autor, que trabalhou com médicos qualificados a cada passo. A sua fisiologia, exames e riscos são diferentes.
- **Consulte profissionais, sempre.** Antes de agir sobre qualquer valor, painel, suplemento ou plano, converse com o seu médico e os especialistas pertinentes, e tenha os seus próprios exames interpretados pelos seus próprios profissionais, a cada passo do caminho.
- **A IA pode alucinar.** O coach é um modelo de linguagem grande. Ele pode estar confiantemente errado, perder contexto ou inventar detalhes. Trate cada recomendação que ele produzir como um gatilho para verificar com um médico, não como direção a seguir. Passe as sugestões da IA por profissionais de verdade, exatamente como o autor fez.
- **As decisões sobre a sua saúde são suas.** Os autores e contribuidores não aceitam nenhuma responsabilidade por como você usa isto.

---

## Pré-requisitos

- **Python 3.9+** — os scripts usam só a stdlib, exceto `advice.py` (`pip install requests`).
- **ffmpeg** — só para `exercise_clip.py` (clipes de demonstração de treino).
- **Supabase CLI** — para aplicar as migrations.
- **Node** — para rodar o agente + o servidor do dashboard.
- **Chaves de API** (veja [`agent/.env.example`](agent/.env.example)): bot do Telegram, Supabase, OpenAI (embeddings), Google/Gemini (visão). Opcionais: WHOOP, Apify (`find-restaurants.sh`), uma API de vídeo (`exercise_clip.py`).

Copie `agent/.env.example` para `~/.env` e preencha. Nunca commite o seu `~/.env` real.

---

## Operação

**Bucket de storage** — um comando cria o bucket de fotos privado:
```bash
python3 agent/scripts/db.py mkbucket health-assets
```

**Agendamento**
- O sync do WHOOP é determinístico; agende `agent/scripts/whoop-sync.py` algumas vezes pela manhã. Use [`agent/setup/whoop-sync.plist.example`](agent/setup/whoop-sync.plist.example) (launchd do macOS) ou [`agent/setup/crontab.example`](agent/setup/crontab.example) (Linux). O agendamento 7/10/13 pega a recuperação sempre que o WHOOP pontua a noite; repetições são idempotentes.
- O check-in matinal aciona o agente (um turno do LLM), então dispare o prompt `/checkin` toda manhã pelo scheduler da sua plataforma (o ClaudeClaw tem um embutido).

**O link `/healthdb`** — o comando envia um botão de deep-link para o dashboard, `https://<seu-host>/healthdb?token=<DASHBOARD_TOKEN>`. O token é guardado em um cookie HttpOnly no primeiro carregamento, e então removido da URL (veja [`agent/dashboard/routes.example.ts`](agent/dashboard/routes.example.ts)).

---

## Privacidade

Estes são dados sensíveis. O projeto Supabase é privado e trancado (service-role no lado do servidor, sem policies anon). Mantenha o seu `CLAUDE.md` real, valores de seed, fotos e `~/.env` fora do git. Este repositório é o blueprint higienizado, não os registros de ninguém.
