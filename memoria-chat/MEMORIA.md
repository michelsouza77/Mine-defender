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

### ✅ Bancada liberada de fábrica — jogador novo estava travado (05/09/2026)
**O sintoma:** quem começava não tinha a Bancada, e o tutorial parecia não existir.
**A causa era uma só, em cadeia:** `isStationBuilt('ferraria')` só era verdadeiro se o jogador
possuísse o item **133**, e o único jeito de obtê-lo era a ação `craft_workbench` — que **não
existe no painel**. Ou seja, ninguém conseguia a Bancada. E como o tutorial tem um passo que
**espera a construção** (`mustBuild`, perto da linha 8692), ele ficava parado ali pra sempre.

**Correção:** a Bancada agora **vem liberada de fábrica** (`isStationBuilt` devolve `true` pra
`ferraria`). Ela forja o machado e a picareta, que são o mínimo pra jogar a Caçada — não faz
sentido exigir construção. Isso destrava o tutorial junto: com ela já disponível, o passo não bloqueia.

Verificado num navegador limpo: seletor de idioma aparece → tutorial abre → **21 passos até o fim
sem travar** → a Bancada abre direto na tela de uso (inventário + slots), não na de construir.

> ℹ️ **O seletor de idioma e o tutorial SÓ aparecem uma vez**, controlados por
> `dinohunt_tutorial_part1_done` no localStorage. Quem já jogou não vê de novo — não é bug.
> Pra testar como jogador novo: janela anônima ou limpar os dados do site.
> Em Configurações existe "Ver tutorial", que repete só o tutorial (não o seletor de idioma).

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

> ⚠️ **AS DUAS AÇÕES SE COMPORTAM DIFERENTE — não copie uma da outra:**
> - **`deposit_exploracao` (mintar):** exige receber **TODOS** os itens configurados, e **aceita 0**.
> - **`retirada_exploracao` (queimar):** exige **TODOS** os itens configurados **e cada um POSITIVO**.
>   - mandar 0 → `fixed burn for <id> must be a positive integer`
>   - omitir → `Missing or invalid burn amount for <id>`
>   - Ou seja: **não existe "não queime este item"**. Queimar 0 não é possível.

### O truque do ENCHIMENTO (solução do Michel, 05/09/2026)
Como a retirada não aceita zero, nos itens que o jogador **não está levando** o jogo manda **1** —
e esse 1 é **criado logo antes**, pelo depósito. **Mint 1 + queima 1 = saldo inalterado**, tudo
dentro da entrada e invisível pro jogador. Esses itens **não entram no mapa**, então morrer não
custa nada por causa deles.

Duas regras que **não podem ser afrouxadas** (as duas quebraram no 1º teste e foram corrigidas):
1. **O mint do enchimento é SEMPRE 1, mesmo se o jogador já tiver o item.** Se você "otimizar"
   criando só o que falta, a queima come **1 unidade de verdade** de tudo que ele já tem, a cada
   entrada — perda silenciosa.
2. **Nunca levar mais do que o jogador tem** (`leva = min(quer, saldo)`). Se ele pedir 5 de algo
   que não possui, levar 1 "criado" faria o depósito da saída **duplicar** esse item.

Validado em 4 cenários (leva o que tem / jogador zerado / pede mais do que tem / mochila vazia):
payload sempre sem zeros e com todos os itens, e o saldo fecha exatamente em `-(o que levou)`.

> ⚠️ Risco residual conhecido: se o **mint do enchimento** der certo e a **queima** falhar logo em
> seguida, o jogador fica com +1 dos itens de enchimento. É pequeno e limitado, mas se um dia
> aparecer relato de item aparecendo do nada, é por aqui.

Foi a **janela de erro detalhada** (`showActionError`, com a mensagem crua do servidor) que
destravou o diagnóstico — o aviso genérico anterior escondia tudo. Vale manter.

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

**O documento das missões leva o número da FAZENDA, não o usuário do Firebase.** Dentro do
iframe do SFL o jogo não enxerga o jwt, então cada aparelho ganhava um uid anônimo próprio —
por isso o progresso do PC e do celular nunca se encontravam. `missionsDocId()` deriva o id da
sessão (`sfl_<farmId>`, procurando o campo em vários lugares do `_sflLastSession`) e cai no
`window.firebaseUid` só se não achar. **De propósito, só as missões usam isso**: mudar o
`window.firebaseUid` global renomearia os documentos de deck/correio/equipamentos e o jogador
perderia o que já está salvo lá.

### ✅ Regras do Firestore — conferidas em 05/09/2026
Faltavam `mail`, `equipped`, `rankings` e `missions` (havia um `leaderboard` que o código não
usa), e isso dava `permission-denied` na hora de salvar. O Michel aplicou as regras corrigidas e
**foram conferidas direto no projeto pelo MCP do Firebase**: as 7 coleções que o código usa
(`players`, `inventories`, `decks`, `mail`, `equipped`, `rankings`, `missions`) estão liberadas.
`missions` é a única com `allow read, write: if request.auth != null` sem amarrar ao uid —
tem que ser assim, já que o documento é da fazenda e não do usuário.

Projeto do Firebase: **`dinohunt-428a7`** (a conta também tem um `mine-defender` antigo, que o
jogo **não** usa — o `firebaseConfig` do `index.html` aponta pro `dinohunt-428a7`).

### ✅ Sincronia PC × celular — TESTADA de verdade (05/09/2026)
Teste automatizado contra o Firestore **real** (não simulado), fingindo estar dentro do SFL com
`window._sflLastSession = { farmId: 999999 }`. Resultado:

| o que foi testado | resultado |
|---|---|
| id do documento vem da fazenda | `sfl_999999` ✅ |
| gravação no Firestore | sem `permission-denied` ✅ |
| aparelho A com 27 mortes, B com 3 → fica com 27 | ✅ merge pelo maior |
| missão coletada no A aparece coletada no B | ✅ |
| estado de ontem no servidor não ressuscita | ✅ ignorado |

O documento de teste (`missions/sfl_999999`) foi limpo no fim.

### 🆕 Tutorial da Caçada + "onde encontrar" nas recompensas (06/09/2026)
**Tutorial** (`showExploreTutorial`, chamado no fim do `openExplore`): 5 linhas — objetivo, COMO
SAIR (era o que ninguém descobria sozinho), inimigos e a perda da mochila ao morrer, ferramentas
e baús. Caixa "não mostrar de novo" grava em `localStorage['dinohunt_exptut_off']`.
Enquanto está aberto, `EXPLORE.paused = true`; ao fechar, **zera `exploreKeys`** — sem isso, uma
tecla segurada durante a leitura fazia o boneco sair andando sozinho.

**Onde encontrar:** `EXP_ITEM_SOURCE` liga cada item do mapa a um texto de origem, e a tela de
recompensas mostra num balão ao tocar no item. Reusa a marcação dos baús (`.rw-chest` +
`.rw-chest-tip`), então o abre/fecha e o clique-fora já vinham prontos.
⚠️ **Os textos descrevem o código de verdade** (sucata só de pedra de ferro, osso 40% ao matar,
ovo raro no chão). Mudando a geração, mude `EXP_ITEM_SOURCE` junto.

### ✅ Mochila de perna não duplica mais o item (06/09/2026)
O item equipado aparecia **na mochila E no slot**, parecendo duas pilhas. Agora sai da mochila:
`renderBackpackGrid()` pula `it.key === EXPLORE.pouch`, e `charEquipPageHtml` filtra o `pick`.

### ✅ Nome do Ancião / do jogador fora de lugar (06/09/2026) — MEDIDO, não chutado
Duas causas empilhadas:
1. `animateHomeSprite()` **sobrescreve width/height** quando o sprite carrega (Ancião: 56px →
   134px, ou seja 64 × 2.1). Qualquer `top` em px fixo vira lixo depois disso.
2. **O bumpkin ocupa MUITO menos que o quadro.** Medido no sheet do SFL (canvas + alfa): num
   quadro de **96×64** ele vai de **y=23 a y=38** — 16px de altura — com **25px de vazio
   embaixo dos pés** e 23px em cima. Na escala 2.1 isso são ~52px de espaço morto.

Por isso as duas tentativas erraram em direções opostas: `top:74px` caía **em cima** dele, e
`top:100%` (fundo da caixa) caía **bem abaixo**, no vazio.

Solução: `animateHomeSprite()` posiciona o `.hb-lb` e o escudo **na hora**, a partir da cabeça
real — `BUMPKIN_HEAD_Y = 23` (e `BUMPKIN_FEET_Y = 39`, se um dia precisar dos pés), vezes a escala.

**O nome fica ACIMA DA CABEÇA** (pedido do Michel), e o **escudo acima do nome** — empilhados,
usando `lb.offsetHeight` pra não se cobrirem. Conferido no Ancião: cabeça em 48px, nome
ocupando 26–44px, escudo terminando em 24px. Jogador: cabeça em 53px, nome terminando em 49px.

⚠️ **Nunca posicione nada em volta desses sprites por px fixo nem pelo fundo da caixa.**
Use `BUMPKIN_FEET_Y`/`BUMPKIN_HEAD_Y` × escala.

### 🔴 "Insufficient 111" ao clicar VÁRIAS VEZES em EXPLORAR (06/09/2026)
O botão EXPLORAR **não tinha trava nenhuma**. Como a queima da entrada demora, o jogador acha
que não começou e clica de novo — e aí duas entradas rodam **em paralelo**: as duas leem a
MESMA mochila e as duas mandam queimar, porque o `saveLoadout({})` só acontece no fim da
primeira. Daí o "Insufficient 111": a segunda tentava queimar o que a primeira já tinha levado.
⚠️ **No pior caso a segunda também dava certo e o jogador perdia os itens duas vezes.**

Corrigido com `_huntEntering`: `huntPlay()` virou uma casca que tranca, chama
`huntPlayInterno()` e destranca no `finally`; o botão também é desabilitado na hora.
Testado: 3 cliques seguidos = **1 entrada só**.

### ⚡ Energia: como funciona hoje (06/09/2026)
**Por que o painel está em mín. 0 / máx. 25** (decisão do Michel): com 25 fixo, a coleta era
**recusada** quando faltava menos de 25 pro topo. O mínimo 0 deixa o servidor creditar só o que
cabe. **Não é sorteio** — com espaço sobrando vêm sempre 25. Por isso `ENERGY_TICK_MIN = 25`,
e o rótulo promete 25 cravado. Se um dia a coleta vier aleatória, pôr 0 ali faz o rótulo voltar
a dizer "até 25" sozinho.

**Coleta automática:** o botão espera o jogador por `ENERGY_AUTOCOLLECT_MS` (1 min) e, se ele
não coletar, o jogo coleta sozinho. `_energyReadySince` / `_energyAutoUuid` contam o tempo e
zeram quando aparece um gerador novo; se a coleta falhar, espera outro minuto em vez de
martelar o servidor. **Barra cheia não coleta** — `updateCollectEnergyButton()` sai antes
quando a energia está no máximo, então os 25 ficam guardados no gerador até abrir espaço.

### 🔴 Energia: coleta vazia matava o ciclo (06/09/2026)
O Michel mudou a ação da recarga no painel: era **mín. 25 / máx. 25** (sempre 25), virou
**mín. 0** — ou seja, **uma coleta pode render menos que 25, inclusive 0**.

O `collectEnergy()` chamava `ensureEnergyJobIfNeeded()` **dentro do `if (ganhou > 0)`**. Com
mín. 0, uma coleta de 0 consumia o gerador e **não iniciava o próximo ciclo de 6h**: o timer
sumia e o jogador ficava sem recarga nenhuma. Era isso o "energia não está sendo coletada
corretamente". Agora o ciclo é reiniciado **sempre**, e a coleta vazia mostra um aviso
informativo em vez de um erro de "saldo não mudou" (que parecia falha).

O rótulo também mentia: o botão prometia "+25⚡" fixo. Existe agora `ENERGY_TICK_MIN` (hoje 0)
e `energyTickLabel()`, que escreve **"+até 25⚡"** enquanto o mínimo for menor que 25.
**Se o painel voltar pra 25 fixo, é só pôr `ENERGY_TICK_MIN = 25`** que o rótulo volta sozinho.

Confirmado: `ENERGY_RECHARGE_MS` = **21600 s = 6 h**, igual ao painel.

### ⚠️ OS 5 OVOS FALTAM NA `deposit_exploracao` (06/09/2026) — AGUARDANDO O MICHEL
Mesmo problema da Tábua, agora nomeado pela conferência nova:
**114 (Ovo Comum), 115 (Ovo Incomum), 116 (Ovo Raro), 117 (Ovo Épico), 118 (Ovo Lendário)**
estão na `retirada_exploracao` e **faltam na `deposit_exploracao`**.
Enquanto não forem acrescentados, quem tiver 0 de qualquer um deles **não consegue entrar no
mapa levando itens** — o enchimento não consegue criar o ovo pra queimar.
Se a intenção for que ovo NÃO entre nas ações, o certo é tirá-lo das DUAS e remover 114–118 de
`EXP_ACTION_EXTRA_IDS` aqui no código.

### 🔴 "Insufficient 105" ao entrar no nível 10 (06/09/2026) — ✅ RESOLVIDO NO PAINEL
**Era a Tábua faltando na `deposit_exploracao`** — o Michel confirmou e corrigiu.
O item **105 é a Tábua**. O erro vem da QUEIMA da entrada: a ação recusou queimar 1 Tábua
porque o jogador tem 0 dela. Como o enchimento deveria ter criado esse 1 antes, a explicação
mais provável é que **105 está na `retirada_exploracao` mas NÃO está na `deposit_exploracao`** —
é exatamente a assimetria que o arquivo já avisava ser proibida (as duas ações TÊM que ter a
mesma lista). ⚠️ **Michel: confira o 105 nas duas ações.**

Duas defesas foram postas no código, mas **nenhuma substitui o acerto no painel**:
1. **Confere se o enchimento nasceu.** O depósito pode responder OK e não criar nada. Agora,
   depois do mint, o saldo é relido e, se o item continuar em 0, a entrada para com uma
   mensagem dizendo qual item falta em qual ação — em vez do 400 cru lá na frente.
2. **Segunda chance na queima.** Se o servidor disser "Insufficient <id>" de um item que era só
   enchimento, o jogo cria esse 1 e repete a queima uma vez (cobre saldo velho, item gasto em
   outra aba/aparelho). O item sai da lista de devolução, senão devolver de novo criaria uma
   unidade do nada.

A janela de erro também **não reconhecia** esse formato: ela só casava `"...for <id>"`, e
`"Insufficient <id>"` passava batido — por isso o erro aparecia cru, sem nomear o item.
Agora reconhece os dois.

### ✅ Fonte da energia e do timer — LIMPA, de propósito (06/09/2026)
⚠️ **Não "corrigir" isto de volta pro estilo do jogo.** Energia e tempo de recarga são os
números que o jogador mais lê, e fonte de pixel em tamanho pequeno fica ruim de ler. O Michel
pediu explicitamente "o mais limpo possível".

Passaram por três estados no mesmo dia: `Pixelify Sans` (fonte de **texto**, destoava) →
`SFLSecondary` (a de **número** do SFL, combinava com o HUD mas continuava pixel) → **fonte do
sistema**, que é o estado final. Existe a variável `--font-clean`
(`system-ui, -apple-system, 'Segoe UI', Roboto, …`): não baixa nada e sai sempre nítida.

Detalhes que importam: o contorno de 4 lados virou **sombra simples** (contorno grosso suja
fonte lisa), com `font-variant-numeric:tabular-nums` pro número não dançar ao mudar de dígito.
Tamanhos: 18px no PC, 14px no celular, 13px na tela menor — menores que os da fonte de pixel
porque fonte limpa **lê maior** no mesmo tamanho. O botão do timer ficou em 16px / 13px.

### ✅ Mochila: arrastar move tudo, teto 64, e os itens que sumiam (05/09/2026)

**1. Faltavam itens na tela do personagem (os totens).** "NA SUA CONTA" só listava o que estava
em `EXP_LOOT_ITEM_IDS` **e** em `LOOT_ITEMS` — ou seja, só o saque da caçada. Totens, ovos e
fabricados estavam apenas em `EXP_ACTION_EXTRA_IDS`, então o jogador via o item no inventário
mas não conseguia levar nem usar.
**Agora todo id que está nas ações é levável**, porque as duas ações têm a mesma lista: o que
pode ser queimado na entrada pode ser mintado na saída. As chaves são geradas como `it<id>`
(ex.: `it127` = Totem Beta), com ícone e nome vindos do SFL/tradução. Foi de 17 pra 28 chaves.
⚠️ **Levar um totem hoje não dá bônus nenhum** — os slots de equipamento continuam "em breve".
Ele só deixou de ser invisível.

**2. Arrastar movia 1 item por vez.** Agora `loadoutAdd(key, tudo)` e `loadoutRemove(key, tudo)`:
o **arrastar** move a pilha inteira, o **toque** continua movendo 1 (importante no celular).
Teto de `EXP_STACK_MAX = 64` por espaço, aplicado também no `bagPreselectRunLoot`.
O teto é só da mochila de casa — a mochila DENTRO do mapa não tem teto de propósito, senão o
jogador perderia saque ao minerar muito de um recurso só.

**3. Não dava pra tirar o item da mochila de perna arrastando.** O slot era só alvo de soltura,
nunca origem — só dava pra desequipar clicando até ciclar pro vazio. Agora o slot é
`draggable` (`pouch:<chave>`) e a mochila aceita a soltura, nas **duas** telas (personagem e
dentro do mapa). No mapa o `addEventListener` é ligado uma vez só (`grid._pouchDrop`), senão
empilharia um handler a cada render.

### 🆕 Aba "Novidades" no correio (05/09/2026)
Segunda aba dentro da janela do correio (a estrutura `.mail-tabs` já existia, com uma aba só e
`cursor:default`; agora são duas, clicáveis, via `switchMailTab`). Mostra versão, data e o que
mudou, em PT e EN, com selo verde "mais recente" no topo.

**A versão do jogo agora SAI DAQUI.** `DINOHUNT_VERSION` e `DINOHUNT_BUILD` são derivados de
`CHANGELOG[0]` — antes eram dois `const` soltos que ninguém lembrava de atualizar (estavam em
`v1.9.1 · 2026-06-25`, meses atrasados, apesar do comentário mandando bumpar a cada mudança).
**Pra lançar uma versão, acrescente um item no TOPO do `CHANGELOG` e pronto.**

Regra do texto: só o que o **jogador percebe**, impessoal, sem detalhe técnico.
A data é escrita em ISO (`2026-09-05`) e `changelogDate()` vira dd/mm/aaaa em PT e mm/dd/aaaa em EN.

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

## 🔌 MCPs instalados nesta máquina (05/09/2026)

Três servidores configurados no `.mcp.json` da raiz do repositório: **playwright**, **firebase** e
**github**. Servem pra eu conseguir testar e conferir sozinho, em vez de pedir print pro Michel.

| servidor | pra que serve |
|---|---|
| `playwright` | abrir o jogo de verdade num navegador, clicar, ler o console e tirar print |
| `firebase` | ler/escrever regras do Firestore e dados do projeto `dinohunt-428a7` |
| `github` | ler o repositório, commitar e abrir PR sem passar pelo GitHub Desktop |

**Como testar o jogo nesta máquina** (o método headless continua valendo e não depende do MCP):
1. `& "C:\Program Files\nodejs\npx.cmd" -y http-server . -p 8123 -c-1` dentro de `game2/`
   (servir por **localhost** importa: o Firebase só autoriza domínios conhecidos, e `file://` não é um).
2. Copiar o `index.html` pra um `__test_x.html` com um `<script>` no fim que escreve o resultado
   num `<pre>`, rodar o Edge headless com `--dump-dom` e extrair o marcador. **Apagar a cópia depois.**
3. O Edge headless precisa de `--user-data-dir` apontando pra uma pasta temporária — sem isso o
   `--dump-dom` sai **vazio**.

**Armadilhas do Windows que já custaram tempo:**
- O `npx` fica em `C:\Program Files\nodejs\npx.cmd`, **não** em `%APPDATA%\npm` (lá só tem o firebase).
- O MCP do Playwright pede o Chrome, que **não** está instalado aqui. Configurado com
  `--browser msedge`, que já existe na máquina.
- `npx` e `firebase` puros **não rodam no PowerShell** — a política é `RemoteSigned` e os atalhos
  `.ps1` do npm não são assinados. Use sempre **`npx.cmd`** e **`firebase.cmd`**. Não precisa mexer
  na política do sistema.
- O firebase estava dando **timeout de 30s** porque o `.mcp.json` usava `npx -y
  firebase-tools@latest`, que baixa o pacote inteiro toda vez que o servidor sobe. Resolvido com
  `npm install -g firebase-tools` e apontando o comando direto pro `firebase`.
- O MCP do github lê o token da variável de ambiente **`GITHUB_TOKEN`** (gravada com `setx`).
  É um token clássico com escopo **`repo`** só. **Se um dia der "bad credentials", é a validade
  que venceu** — gerar outro em github.com/settings/tokens e rodar o `setx` de novo.

## 📝 Como retomar no PC novo

1. Abrir o Claude Code na pasta do projeto (`dinohunt`).
2. Pedir: **"leia a pasta memoria-chat e continue de onde paramos"**.
3. O arquivo do jogo é `game2/index.html` — buscar constantes por nome (números de linha mudam).
