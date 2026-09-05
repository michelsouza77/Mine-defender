# 🧠 MEMÓRIA DO PROJETO DINOHUNT — Resumo completo do chat

> **Para o assistente (Claude):** leia este arquivo inteiro antes de continuar o trabalho.
> Ele resume tudo que foi feito e decidido até 10/07/2026.

---

## 👤 Sobre o Michel (usuário)

- Brasileiro, fala português — **SEMPRE responder em PT-BR**.
- **NÃO é programador** — o assistente faz toda a implementação técnica; explicar as coisas de forma simples, sem jargão.
- Já joga o jogo **em produção** e dá feedback testando em dispositivos reais (PC e celular).
- Dá pedidos em lotes ("corrige X, adiciona Y, muda Z") — atender todos e resumir o que foi feito em lista.

## 🎮 O projeto

- **DinoHunt**: jogo de cartas colecionáveis de dinossauros + modos de ação, construído como **minigame/portal do Sunflower Land (SFL)**.
- **TODO o jogo é UM arquivo**: `game2/index.html` (~14.000 linhas, HTML/CSS/JS puro, sem build).
- Sem Node/Deno instalado na máquina — validação é manual + Michel testa no navegador.
- Repositório git em `Documents/GitHub/dinohunt`. Há uma cópia do código-fonte do SFL em
  `game2/sunflower-land-2.21.42-portal-ai-resolution-changes/` (ótima pra achar assets/caminhos/sons).
- Pasta `adicional/` na raiz: sprites que o Michel fornece (ex.: baús abertos) — embutir como data URI no HTML.

## 🔑 Conhecimento técnico essencial

### Servidor de animação de bumpkins (usado pra jogador E inimigos)
- URL: `https://animations.sunflower-land.com/animate/0_v1_{tokenUri}/{anim}`
- Anims usadas: `idle_walking` (17 frames: idle 0–8, andar 9–16), `attack` (10), `death` (13), `axe`, `mining`.
- Frames são **96×64**. MEDIDO nos sprites reais: os **pés do bumpkin ficam na linha ~38,5** do frame
  (não na base!). Constantes no código: `EXP_FEET_ROW`, `EXP_PLAYER_FEET` (≈+6 do centro), `EXP_ENEMY_FOOT_RAW`.
- tokenUri = 17 números após `0_v1_`: [background, body, hair, shirt, pants, shoes, **tool(slot 6)**, hat,
  necklace, secondaryTool, coat, onesie, suit, wings, dress, beard, aura].
- Corpo Goblin = 4; armadura goblin 320/321/322/323; Goblin Axe 324.

### ⚠️ Assets do GitHub agora vêm pelo jsDelivr (04/09/2026)
O `raw.githubusercontent.com` **não é um CDN** — ele tem limite de requisições por IP e, quando
estoura, devolve erro. O sintoma é feio e confuso: **o chão do modo explorar fica sem textura** e
**vários ícones somem**, enquanto tudo que vem do `sunflower-land.com/game-assets` continua normal
(porque é outro servidor). O código está certo, é só o host recusando.

As 44 URLs foram trocadas para o espelho do jsDelivr, que serve o MESMO arquivo do mesmo repo:
`https://raw.githubusercontent.com/sunflower-land/sunflower-land/main/`
→ `https://cdn.jsdelivr.net/gh/sunflower-land/sunflower-land@main/`

É CDN de verdade (sem limite) e ainda entrega de São Paulo/Rio, então carrega **mais rápido** no
Brasil. Tentei antes usar o CDN oficial do SFL, mas esses arquivos **não existem lá** (404) —
o jsDelivr foi a única saída que mantém os mesmos arquivos.

> Se algum dia o `cdn.jsdelivr.net` for bloqueado no editor do SFL, o conserto é uma
> busca-e-substitui só, voltando pro `raw.githubusercontent.com`.

### CDN de assets
- `https://sunflower-land.com/game-assets/...` funciona no navegador do Michel (verificar sempre com
  `Invoke-WebRequest` antes de usar — alguns caminhos dão 404).
- O que NÃO está no CDN mas existe na cópia local do SFL → **embutir como data URI**.
- Baús (GitHub raw `public/world/`): `basic_chest.png` (nv1), `wooden_chest.png` (nv2), `luxury_chest.png` (nv3).
- Sons: `game-assets/sfx/...` (ver `EXP_SFX` no código). Voz de goblin: `game-assets/sound-effects/goblin-recruiter.mp3`.

### Como inspecionar imagens (PowerShell nesta máquina)
- `System.Windows.Media.Imaging` (WIC) decodifica webp, MAS **perde o canal alfa** (fundo vira preto) →
  medir sprites por **cor** (soma RGB > 12), não por alfa.
- `BitmapImage` com mesma URI **cacheia** — usar `BitmapDecoder` com `IgnoreImageCache` e arquivos únicos.
- A ferramenta Read do Claude mostra PNGs visualmente (converter webp→png antes).

---

## ✅ O QUE JÁ FOI FEITO (modo Explorar / Caçada — ilha estilo LDOE)

### Inimigos
- **Bumpkins vestidos de goblin** via servidor de animação — 10 tokens em `EXP_ENEMY_TOKENS` (editável:
  colar número do perfil). Animações: idle/andar, ataque (dano no frame de impacto ~60%, esquivável),
  morte (13 frames) — tudo automático.
- **10 tipos** em `ENEMY_TYPES` (HP 60→2300, dano 6→180, velocidade, tamanho 0.8x→2.8x).
- **Spawn por mapa** em `MAP_SPAWN`: dropMin/dropMax = QUANTIDADE de inimigos; pesos I1–I10 = quais tipos.
- **Visão**: só perseguem se você entra no campo de visão (`GOB_AGGRO` = 100px, medido pés-a-pés);
  desistem a 1,4×; **voltam pro ponto onde nasceram** (homeX/homeY) e ficam de guarda.
- **Alinhamento pixel-perfect**: mira/ataque/profundidade tudo calculado pelos PÉS reais (linha 38,5) —
  corrigiu inimigo "batendo de cima". Ataque exige estar do lado E nivelado (`atkRange`, `tolY`).
- **Colisão**: não atravessam árvores/pedras/baús/decorações nem o jogador (deslizam por eixo);
  não se empilham (separação); o JOGADOR empurra inimigos pro lado (não trava).
- **Travado durante o golpe** (não desliza atrás de você no meio do swing).
- **Barra de vida** acima da cabeça: aparece só após o 1º dano, enche esquerda→direita (contra-flip),
  verde→vermelha <35%, some na morte.
- **Sons**: grunhido ao te avistar (goblin-recruiter, com intervalo 1,6s), som do golpe (aviso de esquiva),
  grito na morte — **tom grave nos grandes** (`enemyVoiceRate`: playbackRate + preservesPitch=false).
- **Filtro por tema** (`enemyFx`) pra combinar com a iluminação do mapa (flash de dano usa `!important`).

### Jogador
- Ataque/corte/mineração: **não anda durante a animação**; efeito só no impacto (~60%); mexer no
  direcional **cancela** (sem efeito, sem gastar cooldown). `playToolAnim(kind, onImpact)` + `cancelToolAnim`.
- **Vida = 100** (`EXPLORE.hp / EXPLORE.hpMax`). O valor de teste 2500 já foi revertido.
- `EXP_PLAYER_WEAPON` (perto de `PLAYER_DMG`): 0 = arma equipada do SFL; colar ID de wearable pra trocar
  a arma da animação de ataque.
- **Teclado**: WASD/setas andar (preventDefault), **Espaço/E** = ação contextual, **B** = mochila.
  Foco agarrado ao abrir (retries 120ms/600ms), teclas presas limpas entre rodadas.

### Mapas / Temas (níveis 1–10)
- `EXP_THEMES`: 1 Planícies, 2 Floresta, 3 Caverna, 4 Deserto (cactos + toco de cacto embutido),
  5 Pântano, 6 Montanhas Congeladas (neve), 7 Vulcão, 8 Castelo, 9 Dimensão das Sombras,
  10 Arena do Chefe Final (chão vermelho inferno).
- Cada tema: chão (cor+filtro), árvore (16 variantes reais SFL: 4 biomas × 4 estações, 448×48·7f),
  pedra, filtros, minimapa, animação de queda. Título aparece na tela de preview ("Caçada · Nível X" +
  tema) e como letreiro ao entrar.
- **Decorações por tema** (`EXP_DECOR`): cercas, tulipas, cactos, caveiras, pedregulhos, Observador etc. —
  **com colisão** (exceto coletáveis). **Cogumelo coletável** 🍄 (mapas 1/2/3/5): pisar = pega.

### Recursos
- `EXP_RES_TABLE` por nível: árvore/rocha/ferro/ouro min–max (tabela do Michel).
- Pedras: 3 tipos (`stone`/`stone2`/`stone3` — l1 672×48, l2/l3 288×27) + ferro e ouro com l2/l3 também.
- **Tier por mapa**: níveis 1–4 nós básicos, 5–8 versões l2, 9–10 l3 (`expNodeTier`).
- **Rebaixamento ao minerar**: l3→l2→básico (troca a arte, com delay de 300ms pra faísca não bugar);
  o básico encolhe (0.8/0.6) e **some na última batida** (sem colisão depois).
- Drops: pedra=2, ferro=1, ouro=1 (`STONE/IRON/GOLD_PER_ROCK`).
- **Baús abertos**: sprites da pasta `adicional/` embutidos (`EXP_CHEST_OPEN_IMG`) — troca ao abrir.

### Rebalanceamento do DinoCoin nos baús (03/09/2026)
O DinoCoin estava dropando em quantidade muito alta. A pedido do Michel: **teto de 10 no melhor baú**,
os outros reescalados na mesma proporção de antes, e a **chance dobrada** em todos os tiers.
Fonte da verdade: `CHEST_CONTENTS` (a tela de preview lê daqui, então ela se atualiza sozinha).

| Baú | Antes | Agora |
|---|---|---|
| 1 Comum | 2% · 5–20 | **4% · 1–2** |
| 2 Raro | 3% · 10–40 | **6% · 2–5** |
| 3 Épico/Luxo | 6% · 50–150 | **12% · 5–10** |

Confirmado por simulação de 2000 aberturas do baú 3: sai em ~12,7% delas, média 7,3, máximo 10.
**DinoCoin só cai de baú** na Caçada (não está em `EXP_MAP_DROP_KEYS` nem em drop de inimigo),
então essa tabela é o controle completo da entrada de DinoCoin no modo.

### Cash Coin nos baús (03/09/2026)
Adicionado o Cash Coin (token de temporada, item **129**) como drop de baú, na regra pedida pelo Michel:
**o dobro da chance do DinoCoin** e **50% a mais de quantidade**.

| Baú | DinoCoin | Cash Coin |
|---|---|---|
| 1 Básico | 4% · 1–2 | **8% · 2–3** |
| 2 Raro | 6% · 2–5 | **12% · 3–8** |
| 3 de Luxo | 12% · 5–10 | **24% · 8–15** |

(o máximo do baú 2 ficou 8 em vez de 7,5 por arredondamento). Simulação de 3000 baús de Luxo:
Cash Coin saiu em 22,7%, média 11,5, máximo 15.

Mexeu em 4 lugares: `LOOT_ITEMS.cash` (catálogo), `CHEST_CONTENTS` (os 3 tiers),
`EXP_REWARD_ORDER` (ordem na tela de recompensas) e `EXP_LOOT_ITEM_IDS.cash = 129` (depósito).

> ⚠️ **AÇÃO NECESSÁRIA NO PAINEL DO SFL:** a ação `deposit_exploracao` recebe a quantidade de TODOS
> os itens de `EXP_LOOT_ITEM_IDS`. Como o **129** entrou nessa lista, a ação precisa aceitar/mintar
> o item 129, senão **todos os depósitos passam a falhar** (não só o do Cash Coin). Se acontecer, a
> janela de erro do depósito mostra os itens enviados, e o conserto imediato é trocar `cash: 129`
> por `cash: null` — o Cash Coin continua aparecendo no baú, só não é creditado (igual o ouro hoje).

### Nomes dos baús — resolvido (03/09/2026)
Passaram a usar os nomes **oficiais do SFL**, tirados da cópia local do código-fonte
(`src/lib/i18n/dictionaries/en.json` e `pt-BR.json`, chave `chestRewardsList.treasureChest.tabTitle1-3`):
**Baú Básico / Baú Raro / Baú de Luxo** ↔ **Basic Chest / Rare Chest / Luxury Chest**.
A duplicação acabou: `CHEST_NAMES` (tela de preview) agora **deriva** de `EXP_CHEST_NAME` /
`EXP_CHEST_NAME_EN` (tela do mapa), então as duas telas não podem mais divergir.

### Contador do topo: restantes / total (03/09/2026)
Antes mostrava só o total e **nunca atualizava** durante a partida (era um debug do spawn).
Agora mostra `🎁 6/7  👹 26/28` — o que **ainda resta** sobre o total do mapa.
Funciona porque baú aberto vira `opened:true` e inimigo morto vira `dead:true`, e **nenhum dos dois
sai da lista** — então total = `length` e restante = os não-marcados.
Função `updateExploreCounts()`, chamada no spawn, em `killEnemy()` e ao abrir o baú.

### Velocidade dos inimigos (03/09/2026)
Jogador = **50 px/s** (`EXPLORE.speed`, linha do `speed: 50, zoom: 3`).
Inimigo = `type.speed × ENEMY_SPEED_K` (K = 6.5).

**Regra: todos abaixo de 50 px/s — com UMA exceção proposital.**
O **tipo 3 é a PRAGA**: `speed 8.6` = **55,9 px/s (112% do jogador)**, ou seja **ele te alcança**.
Dela não se foge, tem que matar — por isso tem **dano baixo (7)**, é **bem pequena (0.7)** e
morre rápido (HP 110). **Estreia no nível 2**: o peso em `MAP_SPAWN[1].enemies` foi zerado (era 2).
Presença: 16,7% dos monstros no nv2, ~18% nos nv3–5, caindo até 3% no nv10.

Velocidades em px/s: t1 33,8 · t2 27,3 · **t3 55,9 (praga)** · t4 20,2 · t5 31,2 · t6 22,1 ·
t7 41,6 · t8 17,6 · t9 33,8 · t10 24,7. Os grandões (4, 6, 8, 10) seguem lentos de propósito.
O mais rápido depois da praga é o t7 com 41,6.

> ⚠️ Ao mexer em `ENEMY_TYPES`: só o tipo 3 pode passar de 50 px/s. Os outros ficam abaixo,
> senão o jogador perde a opção de fugir de qualquer coisa.

### Padrão visual SFL — telas convertidas (03/09/2026)
O jogo tinha dois visuais convivendo: o **antigo** (`linear-gradient(145deg,#2a1808,#1a0e04)` +
`3px solid var(--gold)` — caixa escura com borda dourada) e o **SFL** (`.lvprev-box`: fundo
`#e4a672` + `border-image` do `light_border.png`).

Convertidas para o padrão SFL: **nível bloqueado** (`showLockedLevelPreview` — as recompensas dele
agora usam o mesmo `renderDropTileGrid` das telas destravadas, nos dois modos; a Arena antes
mostrava só texto solto "130 🪙"), **seletor de idioma** (primeira tela do jogo) e o **modal de
confirmação** (`showConfirmModal`).

**Tela de confirmação antiga REMOVIDA** (a pedido do Michel): Arena e Defesa mostravam a tela nova
de recompensas e, ao clicar em jogar, abriam uma segunda tela antiga (seu deck × deck inimigo +
Voltar/Batalhar). Agora a batalha começa direto pelo botão da tela nova. Foram apagadas
`showDefensePreview`, `closeDefensePreview`, `confirmDefenseBattle` e `showBattlePreview` (~200
linhas). A validação de deck vazio que vivia dentro delas foi preservada em `selectLevel`, e o
fechamento do `#level-modal` foi movido pros dois pontos de entrada. `closeBattlePreview()` ficou.

Ainda no estilo antigo, **de propósito**: `showDepositError` (janela técnica de erro),
`showSflDebug` e `showLeaderboardInternal` — estas duas **não são chamadas por ninguém** (código
morto; se o ranking voltar a ser usado, precisa ser redesenhado).

### Botão de saída + trilha no chão (03/09/2026)
Botão **🚪** no HUD da Caçada (`#exp-exit-btn`, canto inferior direito, acima da mochila).
Ao clicar, desenha **setinhas no chão** dos pés do jogador até a saída, com um círculo marcando o
ponto exato onde a saída dispara. Some sozinho em 9s; clicar de novo apaga antes.

**A trilha ACOMPANHA o jogador**: `updateExitPath()` é chamada a cada quadro pelo `exploreLoop`
(logo depois do `drawMinimap()`) enquanto `EXPLORE._exitPathOn` estiver ligada. Ela redesenha a
partir da posição atual, as setas vão sumindo conforme você se aproxima, e o alvo **troca de borda
sozinho** se outra ficar mais perto. Os elementos são **reaproveitados** (só muda left/top/transform):
recriar o DOM a cada quadro faria a animação escalonada reiniciar e piscar.

Visual (versão final, depois de 2 rodadas de ajuste com o Michel):
- Setinhas de **8×7px**, verdes (`#8fe06a`), **sem sombra nenhuma** (sombra dava impressão de
  estarem flutuando em vez de pintadas no chão).
- **Coladas**: `EXP_EXIT_STEP = 14px`, começando no i=0 (bem no pé do personagem).
- **Andam em direção à saída** em vez de piscar: `@keyframes expExitFlow` faz
  `translateY(0 → -14px)`, linear, **sem `animation-delay`** (todas em sincronia).

> ⚠️ O `-14px` do keyframe **tem que ser igual a `EXP_EXIT_STEP`**. Como cada seta avança
> exatamente o espaço até a próxima, ao reiniciar o ciclo ela cai no lugar da seguinte e o fluxo
> fica contínuo, sem emenda. Se mudar um e não o outro, a trilha passa a "pular".

Como são até 90 setas, `updateExitPath()` **não faz nada se o jogador não andou** e a saída é a
mesma (compara `_exitPathKey/_exitPathX/_exitPathY`) — parado, custo zero.

O botão fica no **canto superior direito** (`top:12px; right:12px`) escrito só **"Sair"** / "Exit".

Como funciona: a saída é **qualquer borda do mapa** (você sai ao chegar a `EXP_EXIT_MARGIN` = 90px
dela, mapa 1200×1200), então o caminho mais curto é sempre a **perpendicular até a borda mais
próxima** — `nearestExitPoint()` compara as 4 distâncias e devolve a menor. Não existe labirinto,
então não precisa de pathfinding.

Detalhes que importam:
- A camada `#exp-exit-path` fica logo depois do `#exp-ground` e **sem z-index**, então passa por
  baixo de árvores, pedras, baús, inimigos e do jogador — parece pintado no chão.
- Mede a partir dos **pés** (`EXPLORE.y + EXP_PLAYER_FEET`), igual ao resto do código.
- A seta é um triângulo CSS apontando pra cima, então a rotação é `atan2(dy,dx) + 90°`.
- Cada seta tem `animation-delay` escalonado (0.09s) — dá o efeito de correrem em direção à saída.
- Limpa sozinha ao sair (`exitExploreFlow`) e ao começar uma caçada nova.

Funções: `nearestExitPoint()`, `showExitPath()`, `clearExitPath()`, `toggleExitPath()`.

### Power ups + mochila de perna (04/09/2026)

**Slot de uso rápido na Caçada.** Botão redondo no HUD (`#exp-pouch-btn`, direita, acima da mochila).
Estado vazio = tracejado com "+". Clicar nele **usa** o item que estiver dentro (gasta 1), no mesmo
modelo do machado/picareta. Clicar vazio abre a mochila avisando pra escolher.

**Como se equipa:** abre a mochila e **toca no item**. Só os power ups ficam clicáveis (contorno
dourado, `.exp-bp-powerup`) e o que está no slot fica marcado em verde (`.equipped`).

Registro em **`EXP_POWERUPS`** (perto do `addToInv`). Hoje só a poção:
`portion: { heal: 25 }` — `heal` é **% da vida máxima** curada por uso.
> ⚠️ O item se chama "Poção 5% HP" porque **na batalha de cartas** ele dá +5% de HP máximo. Na
> Caçada é cura na hora, e 5% seriam 5 de vida (nada, contra inimigos que batem 6–180). Por isso
> ficou 25%. É só um número, muda à vontade.

Pra adicionar um power up novo: criar o item no painel do SFL → registrar em `LOOT_ITEMS` (pra
poder cair/entrar na mochila) → acrescentar a chave em `EXP_POWERUPS` com o efeito → e o id na
categoria Power Ups do mercado. A espada, quando virar consumível, entra por aqui.

A mochila **continua com a cara de sempre**: sem itens ela aparece igual, só com os slots vazios
(nada de mensagem no lugar dela), e o power up não recebe destaque nenhum — só o que ESTÁ no slot
ganha um contorno verde. Foi pedido explícito do Michel depois da 1ª versão, que punha contorno
dourado em todo power up e trocava a mochila vazia por um aviso.

> 🔜 **PLANEJADO:** clicar na mochila vai abrir o **perfil do personagem**, e é ali que se vai
> equipar as coisas NO PERSONAGEM (não nas cartas). A mochila de perna é o primeiro slot desse
> sistema. Quando isso for feito, o painel atual da mochila vira parte dessa tela.

Detalhes de comportamento:
- Usar com **vida cheia não gasta** o item (`applyPowerUp` devolve false → não consome).
- Quando o **estoque acaba, o slot esvazia sozinho**.
- O slot **começa vazio a cada caçada** (resetado junto com o resto em `openExplore`).
- `renderPouch()` é chamada de dentro do `renderBackpackGrid()` — um lugar só, então o número do
  slot nunca dessincroniza do que há na mochila (havia 11 pontos que chamam o render da mochila).

**Aba Power Ups no mercado.** O mercado (`showMarketplace`) **já montava as abas a partir do array
`categories`**, então bastou acrescentar a categoria — a aba nasceu sozinha. Ela lista Poção,
Cristal, Totem Beta e Totem Dino, e clicar leva pro marketplace do SFL, igual às outras abas
(o mercado **não vende dentro do jogo**, só encaminha). Categoria aceita um campo `hint` novo pra
ter legenda própria em vez do subtítulo padrão.

### Depósito não atualizava o inventário (04/09/2026)
`finishExplore()` chamava `depositRunGains()` e pronto — **nada re-sincronizava**. O jogador saía do
mapa e não via os itens no inventário até alguma outra ação sincronizar, parecendo que o depósito
tinha falhado. Agora, ao dar certo, o depósito faz `syncFromSFL(true)` e chama `updateHUD`,
`buildItemsGrid`, `buildInventory` e `refreshAllPanels`.

### Proporção dos baús — corrigido (04/09/2026)
Os sprites de baú são **mais altos que largos** (basic 16×21, wooden/luxury 16×20, abertos 16×20),
mas o baú **fechado** era desenhado numa caixa **quadrada** (`w × w`). Com `background-size:contain`
o sprite encolhia pra 80% da largura, e ao abrir — onde a altura JÁ era corrigida — ele **crescia
de repente**. Agora a proporção certa é aplicada desde o começo, via `EXP_CHEST_RATIO` (fechado,
por tier) e `EXP_CHEST_OPEN_RATIO` (aberto). Ao abrir o tier 1 a altura muda só de 21 pra 20.

### ⚠️ Baú do meio: sprite verde destoa (aguardando decisão do Michel)
`rare_chest.png` e `luxury_chest.png` do SFL são **byte a byte idênticos** (mesmo MD5) — por isso o
tier 2 usa `wooden_chest.png`. Só que ele é **verde**: some no fundo de grama e é o único fora da
família de moldura dourada dos outros dois. O `red_chest.png` seria o encaixe perfeito (mesmo
desenho do luxury, em vermelho).

**O que trava a troca:** o sprite de baú ABERTO do tier 2 que o Michel forneceu
(`adicional/wooden_chest aberto.png`) é verde, combinando com o fechado atual. Trocar só o fechado
deixaria fechado-vermelho + aberto-verde. Pra usar o `red_chest` é preciso a arte do vermelho aberto.

### Levar itens de casa pro mapa — o problema da duplicação (04/09/2026)

**Por que duplicaria:** a ação `deposit_exploracao` **não transfere, ela MINTA** (cria item do nada).
Se o jogador pudesse carregar itens da própria conta pro mapa, o item continuaria na conta *e* seria
fabricado de novo na saída — entrar e sair viraria uma máquina de imprimir item.

**Como as ferramentas já escapavam disso:** o machado e a picareta **nunca são retirados da conta**.
`getAxeBalance()`/`getPickBalance()` só **leem** o saldo, e `burn_axe`/`burn_pickaxe` queimam 1 no
momento em que a ferramenta se gasta. Como ferramenta **não está em `EXP_LOOT_ITEM_IDS`**, ela nunca
é depositada. Nada é criado → nada duplica.

### A MOCHILA é só uma SELEÇÃO (modelo final — 04/09/2026)
> Cheguei a implementar uma versão em que o saque ficava guardado na mochila até o jogador mandar
> pro inventário (formato `{q, off}`). O Michel **descartou**: ficou complicado à toa e o saque
> passava a morar no localStorage, fora da conta. Se aparecer algo de `off`/`bagOff` no código,
> é resto dessa versão e pode sair.

**Regra final, e é só isso:**
- **Pôr item na mochila não muda NADA no SFL** — ele continua no inventário. É só marcar o que levar.
- **ENTRAR no mapa → QUEIMA** tudo que está na mochila (`retirada_exploracao`), e a mochila esvazia.
- **SAIR vivo → MINTA** tudo que sobrou na caçada (`deposit_exploracao`), no `finishExplore`.
- **MORRER → não minta nada.** Perdeu o que levou e o que coletou.

Formato: `{ chave: quantidade }` no localStorage `dinohunt_loadout` (o loader ainda aceita o
formato antigo `{q, off}` e converte).

Como o depósito CRIA item, tudo que ele cria tem que ter sido destruído antes — a queima na
entrada é o que fecha essa conta. Não existe estoque fora da conta em lugar nenhum.

Ao sair vivo, `bagPreselectRunLoot()` deixa o mesmo conjunto **já marcado** na mochila, pra dar
pra entrar de novo levando as mesmas coisas sem remontar. É **só marcação**: se não entrar no
mapa, nada acontece com esses itens (eles estão no inventário, normais).

**Espaços padronizados:** `EXP_BAG_SLOTS = 8` manda em todas as mochilas — a do mapa
(`EXP_INV_MAX`), a da ilha (`EXP_LOADOUT_MAX`) e o painel do baú. Antes eram três números
diferentes (6 na ilha, 8 no mapa, 10 no baú). ⚠️ A capacidade da mochila da caçada caiu de 10 pra
8 pilhas — foi de propósito, pra o número de espaços que aparece ser o número de verdade.

### ✅ Machado e picareta são itens de mochila (05/09/2026)
As ferramentas seguem **a mesma regra de todo o resto**: você escolhe quantas levar, são
**queimadas na entrada** junto com a mochila, as que quebram somem, e as que sobram **voltam no
depósito da saída**. Ids 130 e 131 entraram em `EXP_LOOT_ITEM_IDS`.

O Michel adicionou 130/131 nas duas ações e **removeu `burn_axe` e `burn_pickaxe` do painel**.
Por isso o modelo antigo foi **apagado do código**: `burnOneAxe`, `burnOnePick`, as constantes
`EXP_AXE_ACTION`/`EXP_PICK_ACTION` e a flag `EXP_TOOLS_IN_BAG` não existem mais.
`EXPLORE.axeCount`/`pickCount` agora saem do que está na mochila, não do saldo da conta.

> ⚠️ **Não reintroduza queima avulsa na quebra.** A ferramenta já foi queimada na entrada;
> queimar de novo tiraria uma a mais da conta. Ao quebrar, só se faz `takeFromInv` pra ela não
> voltar no depósito. (`getAxeBalance`/`getPickBalance` continuam definidas mas não são mais
> chamadas — a tela de levar itens lê o saldo direto por `EXP_LOOT_ITEM_IDS`.)

**Durabilidade real: 1.** `EXP_AXE_DURABILITY` e `EXP_PICK_DURABILITY` são **1** — cada árvore
gasta um machado e cada pedra gasta uma picareta. Vários comentários e a descrição do item diziam
"quebra após 5 árvores"; era texto velho e foi corrigido. Se a intenção for 5, muda a constante.

### ⚠️ Fabricação travada: ações que faltam no painel
Estas estão marcadas `TODO(SFL)` no código e provavelmente **nunca foram criadas** — sem elas não
dá pra construir a Bancada nem forjar ferramenta:

| Ação | Pra quê | Queima | Produz |
|---|---|---|---|
| `craft_workbench` | Construir a Bancada | 5×112, 20×102, 10×105 | Bancada (133) |
| `pickaxe_craft_workbench` | Forjar Picareta | 1×132, 3×102 | 1× Picareta (131) |
| `stick_craft_sawbench` | Fazer Graveto na Serraria | 1×101 | 3× Graveto (132) |

`axe_craft_workbench` é a única do grupo sem TODO, então talvez já exista. Repare na cadeia: o
machado precisa de **Graveto**, que vem da Serraria — cuja ação também falta (mas Graveto também
cai no chão do mapa).
- `EXP_WITHDRAW_ACTION = 'retirada_exploracao'` queima o que você leva, ao entrar.
- Na saída, o depósito de sempre devolve o que sobrou na mochila.
- Vantagem: **não precisa separar "trazido" de "coletado"**. O que volta é simplesmente o que ficou
  na mochila. Morrer custa o que levou, sobreviver devolve o que não usou — de graça, sem controle extra.
- A alternativa (não queimar e depositar só o saldo líquido) obrigaria a queimar na morte, ao usar
  **e ao fechar a aba** — e esse último é impossível garantir, então a duplicação voltaria por ali.

✅ A ação **`retirada_exploracao` já foi criada** pelo Michel no painel (04/09/2026) e
`EXP_WITHDRAW_SFL_READY` está **true**. Ela é o espelho exato da de depósito: mesma lista de itens,
recebendo a quantidade de cada um (0 pros que não vão), só que **queimando** em vez de mintar.

**Regra de segurança:** se a queima falhar, o jogador **entra sem os itens**. Deixar entrar com eles
faria o depósito da saída criar item do nada. A queima roda **antes** da cobrança de energia, pra
uma falha não custar energia.

**Tela do personagem** (`homeCharClick`, o bumpkin parado na ilha — antes só dizia "em breve").
> ⚠️ Ela usa **exatamente o mesmo painel do baú do modo explorar** — não é "parecido", é a mesma
> marcação e as mesmas classes: `.exp-loot-modal` › `.exp-loot-header` › `.exp-loot-cols` ›
> `.exp-loot-col` › `.ldoe-section-title` + `.ldoe-inv-grid` › `.exp-loot-hint`.
> Vários estilos são **escopados em `#exp-loot-panel`**, então o `#charloadout-modal` foi
> acrescentado a esses 4 seletores. Se mexer no painel do baú, confira as duas telas.

São **duas páginas** no mesmo modal, controladas por `_charPage` / `charLoadoutPage()`:

1. **Personagem** — coluna **esquerda: a mochila** (bolsa de viagem) com os dois botões lado a lado
   embaixo (*Colocar itens* / *Mandar tudo pro inventário*); coluna **direita: o personagem**, com
   o bumpkin animado no meio (`animateHomeSprite`, que **define largura/altura sozinho** — não force
   tamanho no CSS ou ele desalinha), **6 slots de equipamento em volta** (`EXP_CHAR_SLOTS`) e as
   **3 mochilas de perna** embaixo. Os 6 slots e as mochilas de perna 2 e 3 estão **bloqueados de
   propósito** — ainda não existe item de equipamento de personagem; a estrutura já está pronta.
2. **Itens** — mochila à esquerda e inventário da conta à direita, na mesma estrutura do baú, com
   **arrastar e soltar** entre as colunas (`wireLoadoutDrag`) e **clique** movendo 1 (arrastar não
   funciona no celular, então o clique é o caminho garantido).

No HUD do mapa também existem **3 botões de mochila de perna** (o 1º funciona, o 2º e o 3º
bloqueados), empilhados à direita em `bottom: 184 / 250 / 316px`.

**A mochila DENTRO do mapa mostra o personagem também**, na mesma coluna da tela da ilha —
`charRigHtml()` é compartilhada pelas duas. Escala do bumpkin: `EXP_CHAR_SCALE = 2.1`.

> ⚠️ A coluna do personagem na mochila do mapa é montada **UMA VEZ** (`col._built`).
> `renderBackpackGrid()` é chamada de **11 lugares**; refazer o HTML a cada chamada reiniciava a
> animação do bumpkin e deixava um `setInterval` órfão por chamada. A cada render só as mochilas
> de perna são redesenhadas.

**Arrastar até o slot da mochila de perna** funciona nas duas telas (`.pouch-drop` + `wirePouchDrop`);
clicar num slot cheio esvazia. Só power up é aceito.

A **mochila de perna** na tela do personagem escolhe qual power up já entra equipado na caçada
(`dinohunt_pouch_pick`, aplicado no `openExplore`); clicar cicla entre os power ups da bolsa e volta
pro vazio. Limite `EXP_LOADOUT_MAX = 6` tipos. A bolsa fica no localStorage (`dinohunt_loadout`) e
sobrevive entre partidas. Só aparecem itens que o depósito conhece (`EXP_LOOT_ITEM_IDS` com id
!= null) — senão não teriam como voltar na saída.

*Mandar tudo pro inventário* só **esvazia a bolsa** — nada precisa ser devolvido porque a retirada
só acontece ao entrar no mapa.

### Temporada: loja e missões (05/09/2026)

**Loja — "disponível" ignorava o que o jogador já tem.** Era `remaining = tr.max - used`, e `used`
vinha de `getSeasonRedeemed()` (localStorage), que só conta o que foi comprado **por aquela loja**.
Quem já possuía 1 item de limite 1 continuava vendo "1 disponível". Agora:
`remaining = max - Math.max(used, quantidade que o jogador tem)`, lendo o saldo real por id.
Usei `Math.max` (e não só o saldo) pra não quebrar os consumíveis: quem comprou 3 ovos e abriu
todos continua com o limite gasto. Corrigido nos DOIS lugares — o card e a tela de detalhe.

**Missões viraram LISTA.** Antes era uma só, com o objeto `DAILY_MISSION` e campos fixos.
Agora existe `MISSIONS = [...]`; pra criar outra, é acrescentar ali e criar a ação no painel.
Estado: `{ date, m: { <id>: { p, c } } }`, com **migração** do formato antigo (`{date, progress,
claimed}` → vira `iron10`). `DAILY_MISSION`, `addDailyMissionProgress()` e `claimDailyMission()`
continuam existindo como apelidos, pro código antigo não quebrar.

Missões de hoje:
| id | o que é | meta | ação do SFL |
|---|---|---|---|
| `iron10` | Fabrique Ferro | 10 | `claim_daily_mission1` ✅ existe |
| `kill30` | Cace 30 inimigos | 30 | `claim_daily_mission2` ⚠️ **precisa ser criada** |

O progresso de `kill30` é somado em `killEnemy()`. Missão sem matéria-prima (`reqId: null`)
desenha só o resultado, sem a seta.

**⚠️ Por que o progresso não passava do celular pro PC:** estava **só em localStorage**
(`dinohunt_daily_mission`), que é por aparelho. Agora salva também no Firebase em
`missions/<uid>` (mesmo mecanismo já usado por correio/deck/equipamentos), e o
`syncFromFirebase()` traz de volta. Duas proteções na volta:
- só aproveita se for do **mesmo dia** (senão o estado de ontem ressuscitaria);
- faz **merge pelo MAIOR progresso** entre aparelho e servidor, pra jogar nos dois não
  fazer um sobrescrever o outro pra baixo.

### Minimapa
- Centrado nos pés reais; círculo vermelho = alcance de visão dos inimigos; bolinha de inimigo cresce
  com o tamanho; ouro = bolinha maior brilhante; ferro laranja; **baús = quadrados**; atualização 33ms.

### Tela de recompensas (preview do nível) — atualizada em 30/08/2026
- Mostra ícone + **faixa de quantidade + porcentagem** de cada item que pode aparecer no mapa
  (verde ≥60%, amarelo ≥25%, vermelho abaixo disso).
- **Baús são interativos**: passar o mouse (PC) ou tocar (celular) abre um balão com o CONTEÚDO do baú —
  item, ×min-max e a % de cada um (`CHEST_CONTENTS`, `rwToggleChest`).
- Grade de níveis em 5 colunas (`.level-grid`); modal maior e rolável; layout ajustado pra celular.

### Depósito do loot no SFL — ✅ FUNCIONANDO
- **Saída segura deposita tudo de uma vez**: `depositRunGains()` chama a ação `EXP_DEPOSIT_ACTION =
  'deposit_exploracao'` mandando a quantidade de CADA item configurado (0 pros não coletados — se faltar
  algum, o servidor rejeita). **Morrer não chama** → perde o loot.
- `EXP_LOOT_ITEM_IDS`: wood 101, stone 102, iron 111, coin (DinoCoin) 1, mushroom 124, stick 132,
  scrab 103, egg 113, bone 108, portion 125, coins 122. **gold = null** (ainda sem item no painel).
- **Machado**: item 130, `EXP_AXE_DURABILITY` = 5 árvores, queima 1 via ação `burn_axe` só quando quebra.
- **Picareta**: item 131, `EXP_PICK_DURABILITY` = 1, queima via ação `burn_pickaxe`.
- Se o depósito falhar, abre janela persistente com a ação, os itens enviados e o erro (com botão copiar).

## ✅ O QUE JÁ FOI FEITO (Lobby / Base)

- **O painel central do lobby É a ilha-base** (substituiu a lista vertical de bancadas):
  - Arte real do SFL embutida (recorte 540×378 do `easter_island_tileset.png`, exibido 2× = 1080×756).
  - **Pan** (arrastar) + **zoom** (scroll ancorado no cursor / pinça) com **limite de visão** (clamp ±560z/±420z).
  - Mar ao redor = **tile 64×64 recortado da própria arte** (emenda perfeita, embutido) — água 100% uniforme.
  - 15 **nuvens** SFL flutuando ao redor (animação suave).
  - Sem `will-change` (nitidez no zoom); imagens sem drag nativo (`user-drag:none` + dragstart preventDefault);
    arrasto real não dispara clique; **sem efeito hover** (construções fixas).
- **Construções na ilha** (IDs preservados — lógica de status/cadeado intacta): 🏛️ Casa do Ancião
  (manor, 51%/42% na estrada — clique = "em breve"), 🔥 Fogueira (80px, 64%/27%), ⚙️ **Gerador** (94px,
  84%/44%, renomeado de "Gerador de Carvão"), 🪚 **Serraria** (traduzido de Sawbench, 69%/79%),
  🪺 Chocadeira (cadeado Defesa Lv2), 🧪 Mesa de Poções (cadeado Ancião Lv6). Mercado removido da ilha
  (já tem botão no HUD). Bancada = só a sprite + plaquinhas (sem caixa).
- **Bumpkin do jogador parado na ilha** (idle 1,6×, roupa real do SFL, retry aos 6s pela sessão).
- Botão 🏝️ removido; overlay antigo `#island-home` ficou no código sem uso.
- **Modal das bancadas em estilo SFL**: moldura pergaminho (`--sfl-panel-bg` + `--sfl-light-border`),
  variáveis de cor remapeadas dentro de `.wb-shell` (as 2 telas — construir e produção — mudaram juntas).
- Fundo atrás da interface: azul chapado `#0099db` (mesma cor da arte).

---

## ✅ TRADUÇÃO PT/EN — revisão completa (03/09/2026)

O jogo usa **3 mecanismos** de idioma; saber disso evita quebrar tradução no futuro:
1. Dicionário `I18N = { pt:{...}, en:{...} }` + função `t('chave')`.
2. Atributo `data-i18n="chave"` no HTML estático (aplicado por `applyI18n()`).
   Variante `data-i18n-html="1"` troca innerHTML; `data-i18n-attr` troca um atributo.
3. Ternários no meio do código: `I18N_LANG === 'en' ? 'X' : 'Y'` (ou `isEN ? ...`).

### O que estava errado e foi corrigido
- **Descrição de item nunca traduzia** (o pior): o código lia `SFL_ITEMS_INFO[id].desc` direto,
  sem passar pelo dicionário. Agora usa `t('item.desc.'+id)` com o texto fixo só como último recurso.
- **28 dos 34 itens não tinham descrição no dicionário** e **5 não tinham nome**
  (Coins, Machado, Picareta, Graveto, Workbench) → todos escritos em PT e EN. Cobertura agora 34/34.
- **Textos presos em português no HTML** (nunca trocavam de idioma, em nenhuma situação):
  subtítulo da Coleção, os **8 cabeçalhos de raridade** (COMUNS/INCOMUNS/…/TITÃ),
  "CLIQUE PARA VIRAR", "🏠 Casa" e "🧙 Ancião" → todos ganharam `data-i18n`.
- **"Workbench" aparecia em inglês no modo português** na ilha-base. A chave `map.workbench.label`
  ("Bancada"/"Workbench") já existia no dicionário e simplesmente não estava sendo usada.
- **`section.guide.short`** existia só em PT → adicionada em EN.

### Como validar tradução nesta máquina (sem Node)
Não tem Node/Deno instalado, mas dá pra testar de verdade com o **Edge headless**:
copiar o `index.html` pro scratchpad injetando um `<script>` antes de `</body>` que chama
`setLanguage('en')` e escreve o resultado num `<pre>`; rodar
`msedge --headless=new --dump-dom --virtual-time-budget=25000 file:///...` e ler o `<pre>` do dump.
Serve pra auditar todo elemento `data-i18n` contra o dicionário. Dá pra validar só o dicionário
com `Microsoft.JScript` do .NET (precisa remover as vírgulas finais antes — ES3 não aceita).

### 2ª passada (o Michel viu textos em PT na Caçada jogando)
A auditoria por CHAVE não bastava: existiam chaves presentes nos dois idiomas **com o valor em
português nos dois**. Foi assim que a **linha 2920 do bloco `en:`** acabou sendo uma cópia literal da
linha equivalente do bloco `pt:` — só a última chave tinha sido traduzida. Isso deixava as abas
**Caçada/Defesa** e a descrição do modo em português no jogo inglês, com o título logo acima em inglês.
Traduzidas: `mode.defense.short`, `mode.hunt.short`, `battle.mode.defense.desc`, `battle.mode.arena.desc`,
`battle.mode.hunt.desc`, `battle.title.hunt`, `battle.btn.hunt`.

> **Lição:** ao revisar tradução, comparar **valores**, não só chaves. O teste é: chave existe nos dois
> blocos, valores idênticos, e o valor tem cara de português → provável cópia sem traduzir.
> (Valores idênticos legítimos: `profile.lang.pt/pt_full/changed.pt` e `guide.cashcoin.title`.)

Também corrigido nesta passada:
- **Colisão de nomes das bancadas**: `CRAFT_STATIONS.workbench` é na verdade a **Serraria** (item 109) e
  mostrava "SAWBENCH" nos dois idiomas → em PT agora é "SERRARIA". E `CRAFT_STATIONS.ferraria` é a
  **Bancada de verdade** (item 133), não tinha `nameKey` e caía no `name:'Workbench'` fixo → ganhou
  `nameKey: 'craft.station.ferraria'` (BANCADA / WORKBENCH). `item.name.109` em PT era "Sawbench" → "Serraria".
- **`Você`** sobre o personagem na ilha era fixo. Agora o elemento recebe `data-i18n="map.you"` quando é
  o rótulo padrão (e perde o atributo quando existe nome real do SFL, que não deve ser traduzido) —
  assim ele acompanha a troca de idioma sem precisar recarregar.
- Toasts fixos (`homeCharClick`, `ferrariaClick`), a mensagem de deck vazio e o texto de cópia do erro
  de depósito.

Estado final validado no navegador: **683/683 chaves nos dois idiomas** e nada em português no modo EN.

### Sobrou (não é bug visível, mas vale limpar)
- **Overlay `#island-home` é código morto**: `openIslandHome()` nunca é chamada. Dentro dele tem
  "Sawbench" (não traduzido) e "Gerador de Carvão" (nome antigo, hoje é só "Gerador").
- **Duas funções trocam idioma**: `setLanguage()` (completa) e `selectLanguageAndStart()` (subconjunto,
  sem `refreshAllPanels`/`renderFusionPicker`/`renderBattle`). Hoje não dá problema porque a segunda
  só roda na primeira abertura, antes dos painéis existirem — mas as duas podem divergir no futuro.
- **Chaves duplicadas** no dicionário (a última vence, e são consistentes entre PT/EN):
  `time.in`, `guide.battle.body`, `preview.cost`.

## ⏳ PENDÊNCIAS (o que falta / aguardando o Michel)

1. **Balanceamento do jogador**: dano é 8 vs inimigos de até 2300 HP (tipo 10 mata com 1 golpe).
   Aguardando tabela de armas/progressão do Michel.
2. **🏛️ Casa do Ancião**: implementar tela de níveis — jogador entrega o que o Ancião pede pra subir de
   nível e desbloquear partes do jogo. **Aguardando a tabela** (pedido X → desbloqueio Y por nível).
3. **Ouro sem item no SFL**: `EXP_LOOT_ITEM_IDS.gold` continua `null` — o ouro é coletado no mapa mas
   **não é depositado**. Falta o Michel criar o item de Ouro no painel do SFL e colar o id aqui.
4. **Durabilidade da picareta**: `EXP_PICK_DURABILITY = 1` (1 picareta = 1 pedra), mas o comentário ao
   lado diz "every 5th rock". Confirmar com o Michel qual é a intenção.
5. Possíveis melhorias faladas: interior das telas de bancada (cores pontuais), decorações extras locais
   (tocha tiki, globo de neve — embutir se pedir), inimigos contornando obstáculos com IA melhor.

## 🐛 Avisos conhecidos

- Diagnóstico do IDE "Não use conjuntos de regras vazios" (~linha 1400) é **pré-existente**, inofensivo.
- 20–32 inimigos/mapa = muitos sheets de bumpkin carregados (performance a observar).
- Sessão SFL: `window._sflLastSession`; jogador desconectado usa token padrão `32_1_5_13_20_22_23`.

## 📝 Como retomar no PC novo

1. Abrir o Claude Code na pasta do projeto (`dinohunt`).
2. Pedir: **"leia a pasta memoria-chat e continue de onde paramos"**.
3. O arquivo do jogo é `game2/index.html` — buscar constantes por nome (números de linha mudam).
