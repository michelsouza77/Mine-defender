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
- **Vida em 2500 = VALOR DE TESTE** (linha com comentário `// TEST value`) — ⚠️ voltar ao normal (100)!
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

### Minimapa
- Centrado nos pés reais; círculo vermelho = alcance de visão dos inimigos; bolinha de inimigo cresce
  com o tamanho; ouro = bolinha maior brilhante; ferro laranja; **baús = quadrados**; atualização 33ms.

### Tela de recompensas (preview do nível)
- Só ÍCONES do que pode ser achado (sem quantidades/porcentagens): madeira, pedra, ferro, ouro,
  cogumelo (onde nasce) e baús.

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

## ⏳ PENDÊNCIAS (o que falta / aguardando o Michel)

1. **⚠️ Voltar a vida do jogador pro normal** (está 2500 de teste; procurar `TEST value`).
2. **Balanceamento do jogador**: dano é 8 vs inimigos de até 2300 HP (tipo 10 mata com 1 golpe).
   Aguardando tabela de armas/progressão do Michel.
3. **🏛️ Casa do Ancião**: implementar tela de níveis — jogador entrega o que o Ancião pede pra subir de
   nível e desbloquear partes do jogo. **Aguardando a tabela** (pedido X → desbloqueio Y por nível).
4. **IDs de itens SFL** pro depósito real dos loots da caçada: `EXP_LOOT_ITEM_IDS` (wood/stone/iron/gold/coin
   estão null), pickaxe (`EXP_PICK_ITEM_ID`), ação `burn_pickaxe`. Modelo escolhido: depósito só na saída
   segura (`deposit_wood`). Machado: id 130, ação `burn_axe`.
5. Cogumelo não tem ID SFL (só inventário local).
6. Possíveis melhorias faladas: interior das telas de bancada (cores pontuais), decorações extras locais
   (tocha tiki, globo de neve — embutir se pedir), inimigos contornando obstáculos com IA melhor.

## 🐛 Avisos conhecidos

- Diagnóstico do IDE "Não use conjuntos de regras vazios" (~linha 1400) é **pré-existente**, inofensivo.
- 20–32 inimigos/mapa = muitos sheets de bumpkin carregados (performance a observar).
- Sessão SFL: `window._sflLastSession`; jogador desconectado usa token padrão `32_1_5_13_20_22_23`.

## 📝 Como retomar no PC novo

1. Abrir o Claude Code na pasta do projeto (`dinohunt`).
2. Pedir: **"leia a pasta memoria-chat e continue de onde paramos"**.
3. O arquivo do jogo é `game2/index.html` — buscar constantes por nome (números de linha mudam).
