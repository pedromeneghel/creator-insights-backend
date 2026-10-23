# PRD — Creator Insights Dashboard

Oct 23, 2026 · @Pedro Luiz Meneghel Filho

## 1. Visão geral

O Creator Insights Dashboard é uma aplicação de uso pessoal que, toda terça-feira, coleta dados do YouTube e transforma esses dados em duas coisas: um retrato do desempenho do canal Codificando com Pedro e uma lista priorizada de ideias de conteúdo. A v1 cobre só o YouTube. A arquitetura já nasce preparada para Instagram, Substack e outras fontes.

### Problema

Hoje a decisão de "qual vídeo gravar a seguir" depende de intuição e de ferramentas pagas. O plano gratuito do vidIQ mostra 3 a 4 resultados por seção de palavras-chave, o que é pouco para mapear um nicho. As métricas do canal ficam no YouTube Studio, separadas de qualquer sinal sobre o que está em alta no nicho.

### Objetivos

1. Em menos de 5 minutos por semana, saber quais 3 a 5 temas merecem virar vídeo.
2. Acompanhar a saúde do canal (crescimento, retenção, CTR, tráfego e progresso rumo à monetização) em uma única tela, com comparação contra a semana e o mês anteriores.
3. Transformar comentários do público e buscas que já trazem tráfego em pautas concretas.
4. Custo operacional zero: tudo dentro da cota gratuita das APIs do YouTube, rodando em containers na máquina local.

### Métricas de sucesso do produto

- Pelo menos 1 vídeo por mês gravado a partir de uma ideia sugerida pelo dashboard.
- Coleta semanal concluída sem erro em pelo menos 95% das terças-feiras em que a máquina estiver ligada.
- Consumo de cota da coleta semanal abaixo de 50% do limite diário de cada bucket.

## 2. Escopo

A v1 é single-user, cobre um canal (Codificando com Pedro), é somente leitura em relação ao YouTube e não usa IA.

### Dentro do escopo (v1)

- Login com Google restrito a um e-mail autorizado, configurado por variável de ambiente.
- Conexão OAuth com o canal do YouTube (escopos somente leitura).
- Coleta agendada toda terça-feira às 08:00, mais coleta manual disparada pelo dashboard a qualquer momento.
- Métricas do canal: crescimento, retenção e CTR, tráfego e termos de busca, e progresso para o Programa de Parcerias do YouTube.
- Inteligência de conteúdo sobre vídeos longos em português e inglês: temas em alta no nicho, outliers de concorrentes, pautas a partir de comentários e lacunas do canal.
- Cadastro de termos do nicho e de canais concorrentes pelo dashboard, gravados no banco.
- Resumo da coleta enviado por Telegram.
- Execução 100% em containers via Docker Compose, na máquina local.

### Fora do escopo (v1)

- Editar qualquer coisa no YouTube: título, descrição, tags, miniaturas, playlists, respostas a comentários.
- Métricas de receita (receita estimada, RPM, CPM). O canal ainda não é monetizado. Elas entram quando o canal for aceito no programa, com o escopo `yt-analytics-monetary.readonly`.
- Shorts no radar, nos outliers e nas ideias.
- Classificação ou síntese com LLM. Na v1, a classificação de comentários é feita por regras e palavras-chave.
- Instagram, Substack e outras redes. Ficam para fases futuras, com a arquitetura já preparada.
- Multiusuário, múltiplos canais e qualquer aspecto de SaaS.
- Deploy em nuvem ou em VPS.
- Volume de busca de palavras-chave. Nenhuma API oficial do YouTube fornece esse dado.

## 3. Persona e casos de uso

A persona única é Pedro: criador de conteúdo técnico e engenheiro de software sênior, que decide a pauta semanal com pouco tempo disponível.

| # | Caso de uso | Resultado esperado |
| --- | --- | --- |
| UC-01 | Na terça à tarde, abrir o dashboard depois da coleta semanal | Ver o resumo da semana e o top 5 de ideias em uma tela |
| UC-02 | Escolher o próximo vídeo | Abrir uma ideia e ver as evidências: vídeos de referência, velocidade de views, comentários e termos de busca relacionados |
| UC-03 | Entender por que um vídeo foi bem ou mal | Ver CTR, retenção, origem de tráfego e comparação com a mediana do canal |
| UC-04 | Acompanhar a evolução do canal | Ver inscritos, views, watch time e receita por semana e por mês |
| UC-05 | Responder a uma notícia ou tema novo no meio da semana | Disparar uma coleta manual e ver os resultados atualizados |
| UC-06 | Ajustar o radar | Adicionar ou remover termos do nicho e canais concorrentes |
| UC-07 | Saber, fora do computador, que a coleta terminou | Receber no Telegram o resumo com o top 3 de ideias e as principais variações de métricas |

## 4. Requisitos funcionais

Os requisitos se dividem em cinco blocos: coleta, métricas do canal, inteligência de conteúdo, telas do dashboard e notificações.

### 4.1 Coleta de dados

- **RF-01 Agendamento semanal:** a coleta roda toda terça-feira às 08:00 (America/Sao\_Paulo). Nesse horário a cota diária do YouTube, que é renovada à meia-noite do horário do Pacífico, já foi renovada.
- **RF-02 Execução atrasada:** se a máquina estiver desligada no horário agendado, a coleta roda assim que o backend subir, desde que a última coleta completa tenha mais de 7 dias.
- **RF-03 Coleta manual:** um botão no dashboard dispara a coleta a qualquer momento. Antes de iniciar, o sistema mostra a cota estimada que será consumida.
- **RF-04 Exclusivid****U****m**ó uma coleta roda por vez. Uma solicitação feita durante uma coleta em andamento é recusada, com indicação do status atual.
- **RF-05 Etapas independentes:** a coleta tem etapas separadas (canal, analytics, nicho, concorrentes, comentários, cálculo de scores). Uma falha em uma etapa não descarta o que as outras coletaram. O status final pode ser concluída, parcial ou falhou.
- **RF-06 Histórico:** cada coleta registra início, fim, status por etapa, unidades de cota consumidas por bucket e erros.
- **RF-07 Snapshots:** as estatísticas de vídeos (views, likes, comentários) são gravadas como snapshots com data e hora. Assim é possível calcular variação entre coletas, sem sobrescrever dados.

### 4.2 Métricas do meu canal

- **RF-10 Crescimento:** inscritos ganhos e perdidos, views, watch time (horas) e número de vídeos publicados, por semana e por mês. Cada número é mostrado com a variação em relação ao período anterior.
- **RF-11 Retenção:** duração média de visualização, percentual médio assistido e curva de retenção por vídeo.
- **RF-12 CTR:** impressões e CTR da miniatura por vídeo e no agregado do canal.
- **RF-13 Tráfego:** distribuição por origem (busca do YouTube, sugeridos, navegação, externo, Shorts etc.) no período.
- **RF-14 Termos de busca:** termos de busca do YouTube que trouxeram visualizações ao canal e a cada vídeo, com volume de views por termo.
- **RF-15 Progresso para o Programa de Parcerias:** como o canal ainda não é monetizado, a v1 mostra a distância até cada nível do programa. Os indicadores são inscritos, uploads públicos nos últimos 90 dias e horas de exibição públicas de vídeos longos nos últimos 12 meses. Cada um aparece com percentual atingido, ritmo semanal e data projetada para chegar ao limite.
  - **Limites com vigência:** os níveis ficam em uma tabela com data de início de vigência. Nível de fan funding: 500 inscritos, 3 uploads públicos em 90 dias e 3.000 horas em 12 meses. Nível de receita de anúncios: 1.000 inscritos e 4.000 horas em 12 meses, passando a 8.000 horas para novas candidaturas a partir de 1º de fevereiro de 2027. A projeção usa o limite vigente na data projetada.
  - **Estimativa:** o YouTube conta só horas públicas válidas. Vídeos privados, não listados e excluídos, além de tráfego de anúncios, ficam de fora. O número calculado pela Analytics API é uma estimativa, e a tela indica que o valor oficial está no YouTube Studio, na aba Ganhar dinheiro.
  - O caminho por views de Shorts não é acompanhado na v1.
- **RF-16 Ranking de vídeos:** tabela de vídeos ordenável por views, watch time, CTR, retenção e "desempenho relativo" (múltiplo da mediana do canal).
- **RF-17 Detalhe do vídeo:** página por vídeo com todas as métricas acima, a evolução de views entre snapshots e os comentários classificados.
- **RF-18 Filtro por formato:** as métricas do canal podem ser filtradas entre vídeos longos (incluindo lives e estreias) e Shorts. O progresso para o Programa de Parcerias usa só vídeos longos.

### 4.3 Inteligência de conteúdo

- **RF-20 Radar do nicho:** para cada termo ativo, buscar vídeos longos publicados nos últimos 7 dias, no idioma do termo (português do Brasil ou inglês), e calcular velocidade de views e score de tendência (seção 5).
- **RF-21 Temas em alta:** agrupar os vídeos do radar por termo e listar os temas ordenados por score, separados por idioma, com os vídeos de referência que sustentam cada tema.
- **RF-22 Outliers de concorrentes:** para cada canal concorrente ativo, identificar vídeos longos com desempenho acima de N vezes a mediana do próprio canal (N configurável, padrão 3). Outliers de vídeos encontrados no radar do nicho também entram.
- **RF-23 Pautas dos comentários:** coletar comentários dos vídeos do canal publicados nos últimos 90 dias. Classificar cada comentário por regras em dúvida, pedido de conteúdo, problema técnico, elogio ou outro, e extrair palavras-chave para agrupar assuntos recorrentes.
- **RF-24 Lacunas do canal:** apontar demanda que o canal ainda não atende. Os sinais são: termos de busca que trazem tráfego, mas nenhum título de vídeo do canal contém; temas em alta sem vídeo correspondente no canal; e pedidos recorrentes nos comentários sem vídeo dedicado. Temas em alta em inglês sem conteúdo equivalente no radar em português são marcados como oportunidade de adaptação.
- **RF-25 Ranking de ideias:** consolidar os sinais acima em uma lista única de ideias, com score, idioma de origem, origem dos sinais e evidências clicáveis. A lista respeita a composição por idioma (padrão 60% português e 40% inglês, ver seção 5.2).
- **RF-26 Status da ideia:** cada ideia pode ser marcada como nova, em avaliação, aprovada, gravada ou descartada. Uma ideia descartada não volta ao topo nas coletas seguintes, a não ser que o score suba mais de 50%.
- **RF-27 Cadastro do radar:** a tela de Configurações oferece criação, edição, ativação/desativação e exclusão de termos do nicho e de canais concorrentes, gravados nas tabelas `niche_terms` e `competitors`.
  - **Termo:** texto, idioma (pt-BR ou en) e status ativo.
  - **Concorrente:** cadastro por URL do canal, @handle ou ID. Antes de salvar, o canal é resolvido e validado na Data API, e nome e miniatura são exibidos para conferência. Também tem idioma e status ativo.
  - **Importação em lote:** colar uma lista com um item por linha. A tela mostra pré-visualização, duplicados e itens inválidos antes de confirmar.
  - **Limites:** contador visível de 30 termos e 20 concorrentes ativos, configuráveis.
- **RF-28 Somente vídeos longos:** na v1, Shorts ficam fora do radar, dos outliers e das ideias. Lives e estreias já encerradas contam como vídeos longos.

### 4.4 Dashboard

| Tela | Conteúdo principal |
| --- | --- |
| Visão geral | KPIs da semana com variação, top 5 ideias, status da última coleta, botão de coleta manual |
| Meu canal | Gráficos de crescimento, retenção, CTR, tráfego e progresso para o Programa de Parcerias; filtros de período (7, 28 e 90 dias, 12 meses) e de formato (longos ou Shorts) |
| Vídeos | Ranking de vídeos (RF-16) e detalhe do vídeo (RF-17) |
| Ideias | Ranking de ideias com filtros por origem e status, mais o detalhe da ideia com evidências |
| Radar | Temas em alta e outliers do nicho e dos concorrentes |
| Comentários | Comentários classificados, agrupados por assunto, com link para o comentário no YouTube |
| Coletas | Histórico, status por etapa, consumo de cota e erros |
| Configurações | Termos do nicho, canais concorrentes, limiares dos scores, conexão com o YouTube e com o Telegram |

### 4.5 Notificações

- **RF-30 Resumo no Telegram:** ao fim de cada coleta, agendada ou manual, o sistema envia uma mensagem por bot do Telegram com status, top 3 ideias, variação de inscritos e de views na semana, e um link para o dashboard local.
- **RF-31 Falha:** se a coleta falhar ou ficar parcial, a mensagem informa quais etapas falharam e o motivo resumido (por exemplo, cota excedida ou token expirado).

## 5. Métricas, fórmulas e scoring

Os scores medem velocidade e desempenho relativo, não volume de busca, porque esse é o sinal que as APIs oficiais permitem calcular. Todos os pesos e limiares ficam em Configurações, com os valores abaixo como padrão.

### 5.1 Fórmulas base

Velocidade de views (views por hora desde a publicação):

```latex
VPH = \frac{views}{horas\ desde\ a\ publicação}
```

Desempenho relativo de um vídeo, que define o outlier. A base é a mediana do VPH dos últimos 30 vídeos longos do mesmo canal:

```latex
múltiplo = \frac{VPH_{vídeo}}{mediana(VPH_{últimos\ 30\ vídeos\ do\ canal})}
```

Aceleração, calculada quando o vídeo aparece em duas coletas seguidas:

```latex
aceleração = \frac{views_{t2} - views_{t1}}{horas(t2 - t1)}
```

### 5.2 Regras

- **Outlier:** múltiplo ≥ 3 e vídeo com pelo menos 48 horas de publicação, para evitar ruído das primeiras horas.
- **Formato:** a Data API não informa se um vídeo é Short. Por isso, vídeos com até 180 segundos são tratados como Shorts e descartados do radar, dos outliers, das medianas e das ideias. Lives e estreias já encerradas, identificadas por `liveStreamingDetails`, entram como vídeos longos. Lives em andamento ou agendadas são ignoradas até terminarem.
- **Score de tendência do tema (0 a 100):** 50% pela mediana dos múltiplos dos vídeos do termo, 30% pelo número de canais distintos com vídeo recente sobre o termo e 20% pelo VPH somado. Cada componente é normalizado por percentil dentro do mesmo idioma, para que o volume maior do conteúdo em inglês não esconda os temas em português.
- **Score de ideia (0 a 100):** 35% tendência, 25% outlier, 20% comentários (frequência do assunto) e 20% lacuna (views vindas de busca sem vídeo dedicado). Uma ideia sustentada por 2 ou mais sinais diferentes recebe bônus de 10 pontos. Uma ideia vinda de tema em inglês sem equivalente no radar em português recebe bônus de adaptação de 5 pontos. O total é limitado a 100.
- **Composição por idioma:** os rankings de temas e de ideias seguem a proporção configurada, com padrão de 60% português e 40% inglês. A lista é montada intercalando os melhores de cada idioma nessa proporção. Se um idioma não tiver itens suficientes, o outro completa as posições restantes.
- **Busca por idioma:** termos em português usam `relevanceLanguage=pt` e `regionCode=BR`; termos em inglês usam `relevanceLanguage=en` e `regionCode=US`. A distribuição de termos por idioma é livre, dentro do limite total de 30.
- **Classificação de comentários por regras:** dúvida (contém "?", "como", "por que", "qual"); pedido ("faz um vídeo", "poderia fazer", "próximo vídeo", "ensina"); problema técnico ("erro", "não funciona", "exception", "bug"); elogio (lista de termos positivos); outro. As listas de termos podem ser editadas.
- **Correspondência de lacuna:** um termo é considerado coberto quando todas as suas palavras significativas (sem stopwords) aparecem no título ou nas tags de algum vídeo do canal.

### 5.3 Fontes de dados

| Dado | API | Recurso | Escopo OAuth |
| --- | --- | --- | --- |
| Vídeos e estatísticas do canal | Data API v3 | channels.list, playlistItems.list, videos.list | youtube.readonly |
| Crescimento, watch time, retenção média | Analytics API | reports.query (dimensões day, video) | yt-analytics.readonly |
| Curva de retenção | Analytics API | reports.query (elapsedVideoTimeRatio, audienceWatchRatio) | yt-analytics.readonly |
| Impressões e CTR | Reporting API (validar no spike) | relatórios de alcance do canal | yt-analytics.readonly |
| Origem de tráfego | Analytics API | insightTrafficSourceType | yt-analytics.readonly |
| Termos de busca | Analytics API | insightTrafficSourceDetail filtrado por YT\_SEARCH | yt-analytics.readonly |
| Progresso para o Programa de Parcerias | Analytics API | views e estimatedMinutesWatched dos últimos 12 meses, só vídeos longos públicos | yt-analytics.readonly |
| Radar do nicho | Data API v3 | search.list + videos.list + channels.list | API key ou youtube.readonly |
| Concorrentes | Data API v3 | playlistItems.list (uploads) + videos.list | API key ou youtube.readonly |
| Comentários | Data API v3 | commentThreads.list | youtube.readonly |

### 5.4 Orçamento de cota por coleta

| Bucket | Limite diário | Estimativa por coleta | Base da estimativa |
| --- | --- | --- | --- |
| search.list | 100 chamadas | 30 chamadas | 30 termos × 1 página de 50 resultados |
| Unidades gerais | 10.000 unidades | 500 a 1.500 unidades | videos.list em lotes de 50, uploads de 20 concorrentes, medianas dos 100 vídeos de maior VPH, comentários de até 40 vídeos |

Com essa margem, cabem a coleta semanal e algumas coletas manuais no mesmo dia. O limite de 30 termos e 20 concorrentes é configurável e protegido por validação.

## 6. Requisitos não funcionais

| ID | Categoria | Requisito |
| --- | --- | --- |
| RNF-01 | Execução | Tudo roda em containers. `docker compose up` sobe backend, frontend, banco e worker, sem dependências instaladas no host além do Docker. |
| RNF-02 | Ambientes | Um compose de desenvolvimento (hot reload, volumes) e um de execução local "produção" (imagens multi-stage, sem devDependencies). |
| RNF-03 | Segurança | Login com Google restrito a um único e-mail (`ALLOWED_EMAIL`). Sessão em cookie httpOnly, Secure e SameSite=Lax. |
| RNF-04 | Segurança | Tokens OAuth do YouTube e o token do bot do Telegram ficam criptografados no banco (AES-256-GCM, chave em variável de ambiente). Nada sensível em logs. |
| RNF-05 | Segurança | Portas expostas só em 127.0.0.1. Segredos via arquivo `.env` fora do controle de versão, com `.env.example` documentado. |
| RNF-06 | Resiliência | Chamadas às APIs com retry e backoff exponencial para 429 e 5xx. Erro 403 por cota encerra a etapa sem novas tentativas e marca a coleta como parcial. |
| RNF-07 | Cota | Toda chamada externa registra o custo estimado por bucket. Antes de cada etapa, o sistema verifica se o orçamento restante comporta a etapa. |
| RNF-08 | Desempenho | Telas do dashboard carregam em menos de 1 segundo com dados de 2 anos. Métricas agregadas são pré-calculadas ao fim da coleta. |
| RNF-09 | Duração | Uma coleta semanal completa termina em menos de 10 minutos. |
| RNF-10 | Observabilidade | Logs estruturados em JSON (pino) com correlationId por coleta e por requisição. Endpoint de health check para cada container. |
| RNF-11 | Dados | Migrations versionadas. Backup diário do banco para um volume local, com retenção de 30 dias. |
| RNF-12 | Conformidade | Uso das APIs do YouTube conforme os Termos de Serviço: um único projeto do Google Cloud e dados de terceiros (concorrentes) usados só para análise pessoal. |
| RNF-13 | Qualidade | Lint e formatação automáticos, TypeScript em modo strict, cobertura mínima de 80% no domínio e nos casos de uso do backend. |
| RNF-14 | Acessibilidade | Frontend navegável por teclado, contraste AA e suporte a tema claro e escuro. |

## 7. Arquitetura

São dois repositórios independentes (`creator-insights-backend` e `creator-insights-frontend`), orquestrados por Docker Compose na máquina local. O PostgreSQL é o único serviço de estado: guarda os dados e também a fila e o agendamento de jobs, o que evita um Redis.

&#91;embedded content: arquitetura · 4 containers, 4 serviços externos\]

O worker executa as coletas e fala com as APIs do YouTube e com o Telegram; a api só fala com o Google no login e na conexão do canal.

### 7.1 Decisões de stack

| Camada | Escolha | Motivo |
| --- | --- | --- |
| Runtime backend | Node.js 24 + TypeScript 6 (strict, ESM) | Requisito |
| HTTP | Fastify 5 + Zod (validação e geração de OpenAPI) | Baixo overhead e schemas tipados reaproveitados como contrato |
| Injeção de dependência | Awilix, montado em um composition root | Mantém o domínio livre de frameworks |
| Banco | PostgreSQL 17 | Séries temporais simples, JSONB para payloads brutos, um único serviço de estado |
| Acesso a dados | Drizzle ORM + drizzle-kit (migrations) | Tipagem forte e SQL explícito, isolado na camada de infraestrutura |
| Jobs e agendamento | pg-boss (cron `0 8 * * 2`, tz America/Sao\_Paulo) | Fila e cron sobre o próprio Postgres, com garantia de execução única |
| Google | googleapis (Data, Analytics, Reporting) + OAuth 2.0 | Clientes oficiais |
| Telegram | Bot API via fetch nativo | Um único endpoint, sem dependência extra |
| Logs | pino | JSON estruturado |
| Frontend | React 19 + Vite + TypeScript | Requisito |
| UI | Tailwind CSS 4 + shadcn/ui + gráficos do shadcn (Recharts) | Requisito, com gráficos no mesmo design system |
| Dados no frontend | TanStack Query + React Router | Cache, refetch e estados de carregamento padronizados |
| Servidor do frontend | nginx (proxy de `/api` para o backend) | Mesma origem, sem CORS e com cookie de sessão simples |

### 7.2 Backend: clean architecture

As dependências apontam sempre para dentro: `interface` e `infrastructure` dependem de `application`, que depende de `domain`. O domínio não importa nada de Fastify, Drizzle ou googleapis.

- **domain:** entidades (Channel, Video, VideoSnapshot, NicheTerm, Competitor, Comment, Idea, CollectionRun), value objects (Platform, Period, Score, QuotaCost) e serviços puros (TrendScorer, OutlierDetector, CommentClassifier, GapDetector, IdeaRanker).
- **application:** casos de uso (RunCollection, GetChannelOverview, GetVideoDetail, ListIdeas, UpdateIdeaStatus, ManageNicheTerms, ManageCompetitors) e portas (VideoPlatformPort, ChannelAnalyticsPort, NotifierPort, repositórios, ClockPort, QuotaBudgetPort).
- **infrastructure:** adaptadores (YouTubeDataAdapter, YouTubeAnalyticsAdapter, YouTubeReportingAdapter, repositórios Drizzle, TelegramNotifier, PgBossScheduler, criptografia de tokens).
- **interface:** rotas Fastify, DTOs Zod, autenticação e handlers de jobs.
- **main:** composition root com dois entrypoints, `api` e `worker`, na mesma imagem.

**Extensibilidade multiplataforma:** cada rede social entra como um conector que implementa `VideoPlatformPort` ou `ChannelAnalyticsPort` e devolve dados normalizados (ContentItem, MetricPoint), sempre marcados com o campo `platform`. Adicionar o Instagram ou o Substack significa escrever um novo adaptador e registrá-lo no container de DI, sem alterar o domínio nem os casos de uso.

```text
creator-insights-backend/
  src/
    domain/          entities/ value-objects/ services/
    application/     use-cases/ ports/ dto/
    infrastructure/  youtube/ telegram/ db/ (schema, migrations, repositories) scheduler/ crypto/
    interface/       http/ (routes, schemas, auth) jobs/
    main/            container.ts api.ts worker.ts
  test/              unit/ integration/ e2e/ fixtures/
  docker/            Dockerfile docker-compose.yml docker-compose.dev.yml docker-compose.e2e.yml
```

### 7.3 Modelo de dados (principais tabelas)

| Tabela | Conteúdo |
| --- | --- |
| users, sessions | Usuário único autorizado e sessões |
| platform\_accounts | Conta conectada por plataforma, com tokens OAuth criptografados |
| channels | Canal próprio e canais de terceiros (concorrentes e canais vistos no radar) |
| videos | Metadados: título, tags, duração, data de publicação, tipo (longo ou Short) |
| video\_snapshots | Views, likes e comentários por vídeo e por coleta |
| channel\_daily\_metrics | Métricas diárias do canal vindas do Analytics (views, watch time, inscritos, receita) |
| video\_period\_metrics | Métricas por vídeo e período: retenção, CTR, impressões, tráfego |
| search\_terms | Termos de busca que trouxeram tráfego, por período e vídeo |
| niche\_terms, competitors | Configuração do radar: texto do termo ou ID do canal, idioma (pt-BR ou en) e status ativo |
| radar\_hits | Vídeo encontrado por termo em cada coleta, com VPH e múltiplo |
| comments | Texto, autor, vídeo, categoria e palavras-chave extraídas |
| ideas, idea\_evidence | Ideias consolidadas, score, status e evidências ligadas a vídeos, comentários ou termos |
| collection\_runs, collection\_steps | Histórico das coletas, status por etapa e cota consumida |
| settings | Pesos, limiares, proporção por idioma, listas de termos do classificador e limites do Programa de Parcerias com data de vigência |

### 7.4 API REST (v1)

Todas as rotas ficam sob `/api/v1`. A especificação OpenAPI é gerada a partir dos schemas Zod e publicada em `/api/docs`.

| Método | Rota | Uso |
| --- | --- | --- |
| GET | /auth/google, /auth/google/callback | Login com Google |
| POST | /auth/logout | Encerrar sessão |
| GET | /me | Usuário logado e status das integrações |
| GET | /integrations/youtube/connect | Iniciar OAuth do YouTube (escopos de leitura) |
| POST | /integrations/telegram/test | Enviar mensagem de teste |
| GET | /channel/overview?period= | KPIs com variação e progresso para o Programa de Parcerias |
| GET | /channel/timeseries?metric=&period= | Séries para gráficos |
| GET | /channel/traffic?period= | Origem de tráfego e termos de busca |
| GET | /videos?sort=&type=&page= | Ranking de vídeos |
| GET | /videos/:id | Detalhe do vídeo |
| GET | /ideas?status=&source=&page= | Ranking de ideias |
| GET | /ideas/:id | Ideia com evidências |
| PATCH | /ideas/:id | Alterar status |
| GET | /radar/topics, /radar/outliers | Temas em alta e outliers |
| GET | /comments?category=&videoId=&page= | Comentários classificados |
| GET, POST, DELETE | /settings/niche-terms, /settings/competitors | Configuração do radar, incluindo importação em lote (\`POST .../bulk\`) e resolução de canal por URL ou @handle |
| GET, PUT | /settings/scoring | Pesos, limiares e listas do classificador |
| POST | /collections | Disparar coleta manual (responde 202, ou 409 se houver coleta em andamento) |
| GET | /collections, /collections/:id | Histórico e status por etapa |
| GET | /health | Health check |

### 7.5 Frontend

- Organização por feature: `features/overview`, `channel`, `videos`, `ideas`, `radar`, `comments`, `collections`, `settings`, mais `shared/` com componentes shadcn, cliente HTTP e hooks.
- Tipos do cliente HTTP gerados a partir do OpenAPI do backend (openapi-typescript), para manter o contrato sincronizado entre os dois repositórios.
- Estados de carregamento, vazio e erro padronizados em todas as telas. O status de uma coleta em andamento é atualizado por polling a cada 5 segundos.
- Imagem Docker multi-stage: build com Vite e entrega dos arquivos estáticos pelo nginx.

### 7.6 Execução local

| Serviço | Imagem | Porta (host) |
| --- | --- | --- |
| web | creator-insights-frontend (nginx) | 127.0.0.1:8080 |
| api | creator-insights-backend (entrypoint api) | interna, via proxy do nginx |
| worker | creator-insights-backend (entrypoint worker) | nenhuma |
| db | postgres:17 | interna |

Como os repositórios são separados, o `docker-compose.yml` que sobe a stack completa fica no repositório do backend e referencia a imagem do frontend publicada no GitHub Container Registry. Em desenvolvimento, ele aponta para o caminho local do repositório web.

O OAuth do Google funciona com `http://localhost` como URI de redirecionamento, então não é preciso domínio nem HTTPS na v1.

## 8. Estratégia de testes

Nenhum teste chama as APIs reais do Google ou do Telegram. As integrações externas são substituídas por fakes (unidade), por um servidor HTTP falso (integração) ou por um container WireMock com fixtures gravadas (e2e).

| Nível | Repositório | Ferramentas | O que cobre | Onde roda |
| --- | --- | --- | --- | --- |
| Unidade | backend | Vitest | Serviços de domínio (scores, outliers, classificador, lacunas) e casos de uso com portas fake | Container de teste, a cada commit |
| Integração | backend | Vitest + Testcontainers (PostgreSQL) + MSW | Repositórios Drizzle e migrations, adaptadores YouTube e Telegram contra respostas gravadas, jobs pg-boss | Container com acesso ao Docker, a cada PR |
| E2E | backend | Vitest + cliente HTTP contra `docker-compose.e2e.yml` (api, worker, db, WireMock) | Fluxos completos: login simulado, coleta manual, consulta de ideias, troca de status, 409 em coleta concorrente | Compose dedicado, a cada PR |
| Unidade | frontend | Vitest + React Testing Library | Componentes, hooks, formatação de métricas e utilitários | Container de teste, a cada commit |
| Integração | frontend | Vitest + React Testing Library + MSW | Telas inteiras com a API mockada: carregamento, vazio, erro, filtros e mutações | Container de teste, a cada PR |

### Regras

- Cobertura mínima: 80% em `domain` e `application` no backend; 70% no frontend.
- Fixtures de resposta do YouTube ficam versionadas em `test/fixtures` e são anonimizadas.
- O relógio é injetado (`ClockPort`), para que regras dependentes de data, como "últimos 7 dias" ou "terça-feira 08:00", sejam testáveis de forma determinística.
- Todos os testes rodam dentro de containers, com um script único por repositório (`docker compose -f docker-compose.test.yml run --rm test`).
- Um pipeline de CI no GitHub Actions roda lint, typecheck e os três níveis de teste no backend, e lint, typecheck, unidade e integração no frontend. Em caso de sucesso, publica as imagens no GHCR.

## 9. Roadmap, riscos e questões em aberto

O desenvolvimento começa por um spike de validação das APIs, porque duas métricas pedidas (CTR e termos de busca) dependem de relatórios que precisam ser confirmados na prática antes de fixar o modelo de dados.

### 9.1 Roadmap

As fases são sequenciais e cada uma termina em um critério de saída. As datas ficam para quando o ritmo de desenvolvimento estiver definido.

&#91;embedded content: roadmap · 4 fases da v1 e o que vem depois\]

### 9.2 Riscos

| Risco | Impacto | Mitigação |
| --- | --- | --- |
| App OAuth do Google em modo "Testing" emite refresh tokens que expiram em 7 dias, o mesmo intervalo da coleta semanal | Coleta de terça falha por token expirado | Publicar o app em produção (uso próprio, aceitando a tela de app não verificado). Testar a renovação do token no spike e avisar pelo Telegram quando ela falhar |
| Impressões e CTR podem estar disponíveis só na Reporting API, com atraso de cerca de 2 dias | RF-12 com dados defasados ou mais complexos de obter | Validar no spike. Se confirmado, criar o job de relatório na conexão do canal e exibir a data de referência dos dados |
| Máquina desligada na terça às 08:00 | Semana sem coleta | Execução atrasada ao subir o backend (RF-02) e coleta manual |
| search.list retorna poucos resultados e ordena por relevância | Radar enviesado | Usar `order=viewCount` e `publishedAfter`, com idioma e região conforme o idioma do termo, além de termos específicos |
| Conteúdo em inglês tem volume muito maior que o em português | Ranking dominado por temas em inglês | Normalização por idioma e composição 60/40 no ranking (seção 5.2) |
| Horas de exibição calculadas pela API diferem das horas públicas válidas do YouTube | Projeção de monetização otimista ou pessimista | Exibir como estimativa e permitir registrar manualmente o valor oficial do YouTube Studio para calibrar a projeção |
| Regra de 180 segundos classifica como Short um vídeo longo curto | Vídeo curto fora do radar | Aceitável na v1, porque vídeos longos de até 3 minutos são raros no nicho |
| Classificação de comentários por regras é imprecisa | Pautas ruidosas | Listas de termos editáveis e porta `CommentClassifier` pronta para um adaptador com LLM no futuro |
| Score favorece vídeos muito recentes | Falsos outliers | Mínimo de 48 horas de publicação |
| Mudança de cotas ou endpoints pelo Google | Coleta quebra ou estoura cota | Orçamento de cota configurável, testes com fixtures e acompanhamento do revision history da API |

### 9.3 Decisões

As perguntas abertas na primeira versão do PRD foram respondidas e já estão refletidas nas seções acima.

| Questão | Decisão |
| --- | --- |
| Monetização do canal | Canal fora do Programa de Parcerias e sem receita. O dashboard acompanha o progresso até os limites do programa (RF-15); métricas de receita ficam para depois da aprovação |
| Termos do nicho e concorrentes | Tabelas `niche_terms` e `competitors`, com cadastro pela tela de Configurações, validação e importação em lote (RF-27) |
| Idioma do radar | Português do Brasil e inglês, com composição de 60% e 40% nos rankings (seção 5.2) |
| Shorts | Fora do radar, dos outliers e das ideias na v1 (RF-28) |
| Horário da coleta | Terça-feira às 08:00 (America/Sao\_Paulo) |
| Lives e estreias | Tratadas como vídeos longos depois de encerradas |
| Nomes | Produto: Creator Insights Dashboard (provisório). Repositórios: `creator-insights-backend` e `creator-insights-frontend` |
