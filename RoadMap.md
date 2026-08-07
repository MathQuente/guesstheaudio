# GuessTheSong — Roadmap do Projeto

> Jogo de adivinhar músicas (anime, séries, games, filmes) com preview de 30s via iTunes API  
> Stack: PHP 8.2+ / Laravel 11 / MySQL / Vue 3 + Inertia.js

---

## Visão Geral

O projeto é dividido em 8 fases:

1. Setup & Estrutura base
2. Banco de dados & Models
3. Integração iTunes API
4. Lógica do jogo (backend)
5. Interface do jogo (frontend)
6. Painel admin
7. Funcionalidades extras (ranking, auth, etc.)
8. Deploy

> **Escopo atual: sessão local, sem contas.** Por enquanto o jogo roda inteiro sem sistema de login — cada jogador é identificado só por um ID anônimo gerado no navegador e guardado no `localStorage` (e opcionalmente um cookie). Isso simplifica bastante o MVP: sem tela de login, sem se preocupar com LGPD/consentimento de conta, sem a tabela `users` no caminho crítico de nenhuma das fases abaixo. A **Fase 7 (Autenticação)** continua no roadmap, mas é opcional e fica para depois — só entra em cena se/quando o projeto quiser contas de verdade e ranking global entre jogadores.

---

## FASE 1 — Setup & Estrutura Base

- Crie o projeto Laravel normalmente com o Composer
- Configure o `.env` com as credenciais do banco e nome da aplicação — a iTunes API não precisa de chave, é pública
- Instale o `guzzlehttp/guzzle` para fazer requisições HTTP ao iTunes
- Para o painel admin, o **Filament** é uma boa pedida — gera CRUD completo com pouco esforço. Mas um admin manual com Blade também resolve
- Organize as pastas desde o início: pense em separar Controllers, Services e Models de forma clara. Services são ótimos para isolar a lógica de negócio dos controllers

---

## FASE 2 — Banco de Dados & Models

### Tabelas sugeridas

**`songs`** — o acervo de músicas  
- Guarde os dados básicos da música: título, artista, nome da obra de origem (anime, série, game, filme)
- Tenha uma coluna de categoria e outra de dificuldade
- Salve o `itunes_track_id` — ele é fundamental para revalidar a URL do preview depois
- Salve a `preview_url` (o `.m4a` de 30s) e a `artwork_url` (capa)
- Tenha uma flag `is_active` — músicas precisam ser revisadas antes de entrar no jogo
- Tenha uma flag `has_daily` para saber se a música já foi usada no desafio diário

**`daily_challenges`** — músicas agendadas por dia  
- Relacionamento com `songs`
- Uma música por dia (coluna de data com unique)
- Pode ter categoria do desafio (ex: "só anime hoje")

**`game_sessions`** — registro de cada partida  
- Relacionamento com `songs` e opcionalmente com `daily_challenges`
- Guarde o `user_id` como nullable — mantenha a coluna pensando no futuro, mas por enquanto ela nunca é preenchida (não há login)
- Salve as tentativas como JSON — simples e funcional
- Guarde se o jogador ganhou, quantas tentativas usou e quanto tempo levou
- O identificador de quem jogou é o `anonymous_id` (UUID gerado no frontend e salvo no `localStorage`) — é ele que faz o papel de "usuário" por enquanto

> Um exemplo simples do que fica salvo no `localStorage` do navegador, pra deixar concreto:
> ```json
> {
>   "anonymous_id": "b3f1c2e0-...-uuid",
>   "daily": { "date": "2026-08-07", "songId": 42, "attempts": 3, "won": true },
>   "stats": { "played": 12, "won": 9, "streak": 4 }
> }
> ```
> O backend recebe esse `anonymous_id` a cada requisição (header ou body) e usa ele pra achar/criar a linha em `game_sessions` — sem sessão de login, sem cookie de autenticação do Laravel.

### Dicas para os Models

- No model `Song`, crie um método `toGameArray()` que retorna **apenas os campos seguros** para o frontend — nunca exponha o título antes do fim da rodada
- No model `DailyChallenge`, um método estático `today()` que retorna o desafio de hoje é muito conveniente
- Use `$casts` para booleanos e JSON — facilita muito o trabalho

---

## FASE 3 — Integração iTunes API

- Crie um `ItunesService` dedicado para centralizar todas as chamadas à API
- A API base é `https://itunes.apple.com/search` — sem autenticação
- Filtre apenas resultados que tenham `previewUrl` — nem todas as músicas têm preview disponível
- Normalize o retorno da API para o seu padrão interno antes de salvar no banco
- Crie um **Artisan Command** para fazer o seed de músicas — assim você pode rodar quando quiser e adicionar novos termos de busca
- Pense nos termos de busca: "naruto opening", "attack on titan ost", "final fantasy vii ost" — seja específico
- Adicione um `sleep(1)` entre as requisições em batch para não abusar da API
- Crie outro command para **revalidar as URLs** periodicamente — as URLs do iTunes podem mudar. O `itunes_track_id` é o que permite buscar a URL atualizada. Esse command pode rodar como cron semanal

---

## FASE 4 — Lógica do Jogo (Backend)

### Regras sugeridas

- **6 tentativas** por rodada
- A cada erro, revela mais do áudio: os primeiros segundos são poucos, vai aumentando gradualmente até os 30s completos
- O jogador digita o nome — não precisa ser case-sensitive nem exato
- Use `similar_text()` do PHP para aceitar respostas próximas — uma similaridade de ~80% é um bom threshold
- Aceite tanto o título da música quanto o nome da obra de origem como respostas corretas
- **Modo diário**: mesma música para todos, troca à meia-noite
- **Modo livre**: música aleatória por categoria, o jogador pode jogar quantas vezes quiser

> **Como saber se o jogador "já jogou hoje"?** Sem auth, esse controle é só no cliente: ao terminar o desafio diário, salve a data no `localStorage` (o campo `daily.date` do exemplo da Fase 2). Ao abrir a página, compare com a data de hoje — se bater, mostra o resultado salvo em vez do jogo. **Trade-off aceito por enquanto:** o jogador pode limpar o `localStorage` e jogar de novo, ou até forjar o histórico local com um pouco de esforço. Pra um MVP casual isso é aceitável; se um dia importar corrigir, dá pra mitigar com um cookie assinado pelo servidor (`Cookie` + `hash` de validação) ou, futuramente, com autenticação de verdade (Fase 7).

### Dicas para o GameController

- Nunca envie `title` ou `artist` no payload inicial da página — use o `toGameArray()` do model
- No endpoint de tentativa, só revele a resposta quando o jogo acabar (acerto ou 6ª tentativa)
- O endpoint de autocomplete busca títulos e nomes de obras para sugerir enquanto o usuário digita — faça com `LIKE` e limite a 10 resultados

---

## FASE 5 — Interface do Jogo (Frontend)

### Estrutura sugerida

- Separe bem as páginas: uma para o desafio diário, uma para o modo livre, uma para quando não há desafio do dia
- O `AudioPlayer` merece ser um componente isolado e reutilizável

### O player de áudio — ponto central do jogo

- Use a tag `<audio>` do HTML com a `previewUrl` como source
- O segredo é **não usar a duração natural do arquivo** — você controla até onde o áudio pode tocar via código, pausando e resetando quando atingir o limite de segundos revelados
- Use um `setInterval` para monitorar o `currentTime` do audio element
- Mostre uma barra de progresso baseada em `currentTime / revealSeconds`
- Quando o `revealSeconds` aumentar (nova tentativa), resete o player

### Dicas gerais de frontend

- Use `debounce` no input de autocomplete — não dispare uma requisição a cada tecla
- As sugestões do autocomplete devem fechar ao clicar em uma delas ou clicar fora
- Mostre o histórico de tentativas abaixo do player: tentativas erradas em vermelho, a correta em verde
- Quando o jogo acabar, mostre a capa do álbum — é um belo momento visual
- O resultado no estilo Wordle (emojis coloridos para compartilhar) é um diferencial que engaja muito

---

## FASE 6 — Painel Admin

### O que o admin precisa fazer (no mínimo)

- Listar músicas com filtros por categoria e status (`is_active`)
- Ativar e desativar músicas (idealmente em massa)
- Editar o `source_name` das músicas — o seed importa do iTunes mas não sabe a qual anime/série pertence
- Agendar a música do dia (criar entradas na `daily_challenges`)
- Ter um jeito de buscar novas músicas no iTunes diretamente pelo painel

### Filament ou manual?

- **Filament** é mais rápido: gera os Resources com CRUD completo, filtros e ações em massa automaticamente
- **Admin manual** com Blade é mais simples de entender e customizar se você quiser controle total

---

## FASE 7 — Funcionalidades Extras (Pós-MVP)

### Autenticação — **não fazer agora, é opcional/futuro**

- Fica de fora do escopo atual (sessão local, sem contas). Só faz sentido revisitar isso se/quando o projeto quiser: histórico de partidas persistente entre dispositivos, ranking global de verdade, ou qualquer coisa que exija saber "quem" é o jogador além do `anonymous_id` do navegador
- Usuários anônimos continuam usando só o `anonymous_id` no `localStorage` — simples e sem atrito, é o suficiente para o MVP
- Se um dia isso virar prioridade, o **Laravel Breeze** com Vue + Inertia é o caminho mais rápido para implementar auth nessa stack — mas isso é uma decisão para depois, não parte do plano atual

### Compartilhamento de resultado (Wordle-style)

- Gere um texto com emojis representando as tentativas (🟥 = errou, ✅ = acertou, ⬜ = pulou)
- Use `navigator.clipboard.writeText()` para copiar com um clique
- Inclua a data e o link do jogo no texto
- Não depende de conta — funciona igual com o `anonymous_id`, então pode ser feito dentro do escopo atual

### Ranking — reformulado: sem auth, sem ranking global (por enquanto)

- Sem contas de usuário, não dá pra montar um ranking global confiável entre jogadores (qualquer um pode ter vários `anonymous_id` diferentes, então comparar "jogadores" globalmente não é honesto)
- Em vez disso, use o `stats` que já fica salvo no `localStorage` (ver exemplo na Fase 2) para mostrar **stats pessoais locais**: jogos jogados, % de vitórias, sequência atual (streak) — tudo calculado e exibido só no navegador do próprio jogador, sem endpoint novo no backend
- Um ranking global de verdade (semanal, por vitórias etc.) volta a fazer sentido junto com a Autenticação, no futuro — a tabela `game_sessions` já guarda `won`, `attempts_count`, `created_at`, então quando/se a Fase 7 (Auth) acontecer, o ranking é só uma query em cima do que já existe

### Cron jobs

- Revalidação semanal das URLs do iTunes
- Seed automático semanal para manter o acervo crescendo

---

## FASE 8 — Deploy

### Opções recomendadas

| Plataforma | Custo | Indicado para |
|---|---|---|
| Railway | Gratuito (limitado) / ~$5/mês | Começar, protótipo |
| Render | Gratuito (limitado) / ~$7/mês | Igual Railway |
| DigitalOcean App Platform | ~$12/mês | Produção estável |
| VPS (Hetzner/DigitalOcean) | ~$5-6/mês | Mais controle, produção |

### Checklist antes de subir

- Rode os caches do Laravel em produção: config, route, view
- Desative o `APP_DEBUG`
- Configure as variáveis de ambiente no painel da plataforma escolhida
- Certifique-se de que as migrations rodam no deploy

---

## Ordem de Execução Sugerida

**Semana 1**
- Setup do projeto
- Migrations e Models
- ItunesService + seed inicial de músicas

**Semana 2**
- GameController e rotas
- GameService (lógica de tentativas)
- Tela do jogo com player de áudio

**Semana 3**
- Admin básico (ativar músicas, agendar diário)
- Autocomplete
- Testes manuais do fluxo completo

**Semana 4+**
- Compartilhamento de resultado
- Stats pessoais locais (via `localStorage`, ver Fase 7)
- Deploy
- *(opcional/futuro, fora do escopo atual)* Auth, histórico persistente e ranking global

---

## Pontos de Atenção

1. **Nunca envie `title` ou `artist` ao frontend antes do fim da rodada** — use sempre o `toGameArray()` que omite esses campos
2. **Guarde sempre o `itunes_track_id`** — é o que permite revalidar a `preview_url` quando ela quebrar
3. **Requisições ao iTunes devem sair do backend Laravel**, nunca do frontend — evita problemas de CORS
4. **Respeite um intervalo entre requisições em batch** para não ser bloqueado pela API
5. **A iTunes API é pública mas não documentada formalmente** — monitore se algo parar de funcionar
6. **A sessão do jogador vive inteiramente no cliente por enquanto** (`localStorage`/cookie) — o backend recebe o `anonymous_id` a cada requisição mas não autentica ninguém; não existe login, senha ou sessão do Laravel no meio disso. Essa é uma escolha deliberada para manter o MVP simples, não um item pendente

---

_Roadmap do projeto GuessTheSong — Versão 1.1 (escopo atual: sessão local, sem contas)_
