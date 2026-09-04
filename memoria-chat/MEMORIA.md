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

### Velocidade dos inimigos — regra de ouro (03/09/2026)
**Nenhum inimigo pode ser mais rápido que o jogador, em nenhum nível.**
Jogador = **50 px/s** (`EXPLORE.speed`, linha do `speed: 50, zoom: 3`).
Inimigo = `type.speed × ENEMY_SPEED_K` (K = 6.5). **Teto adotado: speed 6.9 = 44,85 px/s (~90%)**,
pra sempre dar pra escapar correndo. Antes o mais rápido era 39 px/s.

O **tipo 3 virou a PRAGA**: o mais rápido do jogo (6.9 → 44,9 px/s), mas **dano baixo (7)** e
**bem pequeno (0.7)** — pra ser chato de perseguir, não letal. HP 110 pra morrer rápido.
**Estreia no nível 2**: o peso dele em `MAP_SPAWN[1].enemies` foi zerado (era 2).
Presença: 16,7% dos monstros no nv2, ~18% nos nv3–5, caindo até 3% no nv10.

Velocidades atuais em px/s: t1 33,8 · t2 27,3 · **t3 44,9** · t4 20,2 · t5 31,2 · t6 22,1 ·
t7 41,6 · t8 17,6 · t9 33,8 · t10 24,7. Os grandões (4, 6, 8, 10) seguem lentos de propósito.

> ⚠️ Ao mexer em `ENEMY_TYPES`, conferir sempre `speed × 6.5 < 50`.

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
