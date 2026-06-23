# DinoHunt — Documento de Passagem (para continuar no Claude Code / VS Code)

> **Para o Claude que ler isto:** Você está assumindo um projeto em andamento. Leia tudo antes de editar. O usuário é o **Michel**, brasileiro, fala **português**, prefere honestidade direta sem rodeios. Ele NÃO é programador — depende de você pra implementação técnica, mas tem ótimo conhecimento do domínio (Sunflower Land, jogos). Responda sempre em PT-BR.

---

## 1. O QUE É O PROJETO

**DinoHunt** é um jogo de cartas colecionáveis de dinossauros, feito como **minigame/portal do Sunflower Land (SFL)** — uma plataforma de jogos blockchain. O jogo já está **em produção** (Michel joga nele de verdade e dá feedback de dispositivos reais).

- **Arquivo único:** todo o jogo é UM arquivo HTML (`index.html`), com CSS e JS embutidos. ~11.000+ linhas.
- **Plataforma:** roda dentro do editor de minigames do SFL (https://sunflower-land.com).
- **Stack do SFL (o jogo "pai"):** TypeScript + React 19 + Vite + Phaser 3. Mas o DinoHunt em si é **HTML/CSS/JS puro** (vanilla).
- **Testes:** Michel testa no Android (Kiwi Browser / Chrome) e no PC.

---

## 2. ESTADO ATUAL DO JOGO (o que já existe e funciona)

Todas as telas já estão estilizadas no visual pixel-art do SFL (bordas, painéis, abas conectadas):

- **Coleção** (99 dinos, 8 raridades), **Ovos** (gacha), **Itens** (inventário de recursos), **Fusão**, **Guia**
- **Loja de Temporada** + **Recompensas** (missão diária: fabricar ferro 10x → ganha token)
- **Marketplace P2P**, **Caixa de Mensagem** (mailbox), **Config/Perfil**, **Idioma** (PT/EN)
- **Modo Batalha** com 3 sub-modos em abas: **Defesa**, **Arena**, **Caçada**
  - Defesa e Arena: deck de 5 cartas, níveis (só número, clica → preview com recompensas em tiles + %), tela de vitória com recompensas
- **MODO CAÇADA / EXPLORAR** (em desenvolvimento ativo — ver seção 4)

---

## 3. COMO O JOGO INTEGRA COM O SFL (importante!)

O SFL fornece uma SDK pro minigame. Pontos-chave:

- **Saldos/balances:** `window._sflLastSession?.data?.playerEconomy?.balances` — objeto `{ "itemId": quantidade }`. Ex: `bal['129']` = Cash Coin (token de temporada), `bal['1']` = DinoCoin, `bal['112']` = Ferro, `bal['103']` = Sucata.
- **Ações SFL:** `await sflExecuteAction('NomeDaAcao')` → retorna `{ ok, error }`. Cada ação precisa ser **criada por Michel no painel do SFL** primeiro. Exemplos já criados: `Mint`, `burn`, `Win_level_X`, `Buy_egg`, `Collect_energy`, `Claim_daily_mission1`, `score.submitted`.
- **Sincronização:** `syncFromSFL(silent)` atualiza `window._sflLastSession`. O SFL é a **fonte da verdade** pros saldos. Firebase (projeto `dinohunt-428a7`) é usado só pra deck/níveis/ranking.
- **i18n:** TODA string tem versão PT e EN. Ao adicionar texto, edite as DUAS.

### IDs de itens conhecidos
- 1 = DinoCoin | 103 = Sucata (Scrap) | 112 = Ferro (Iron) | 122 = Coins | 124 = Cogumelo | 125 = Poção 5%HP | 129 = Cash Coin (token temporada)
- 127 = Totem Beta | 128 = Totem Dino

---

## 4. MODO EXPLORAR / CAÇADA (foco atual — estilo Last Day on Earth)

Acessado por: **Batalha → aba Caçada → "Explorar Ilha" → nível 1**.

### Conceito
Uma zona quadrada de farm de recursos. O jogador anda com o personagem (bumpkin animado do SFL), coleta madeira (árvores) e pedra (rochas), e sai pela borda (nuvens) levando o que coletou — como o "sair do bunker" do Last Day on Earth.

### O que JÁ funciona
- **Mapa quadrado** (1200×1200px), fixo.
- **Câmera segue o personagem** (o mundo se move via `transform: translate + scale`, personagem fica centralizado). Zoom 3 (mobile) / 4 (desktop) — igual SFL.
- **Velocidade do personagem = 50 px/s** (constante real do SFL: `WALKING_SPEED = 50`), aplicada com delta-time.
- **Spawn de 10–20 árvores/pedras** sem sobreposição (espaçamento mínimo 90px), longe do canto onde o player nasce.
- **Player nasce no canto superior esquerdo** (longe de onde inimigos vão spawnar no futuro).
- **Colisão** estilo SFL: caixa retangular pequena na BASE do recurso (sprites são ancorados embaixo, `translate(-50%,-100%)`). Player colide com os "pés" (offset pra baixo). Desliza ao encostar (testa X e Y separados).
- **Coleta:** toca/clica perto de um recurso → bate algumas vezes → quebra → +2 madeira/pedra.
- **Mochila** (botão 🎒 no canto) → painel com itens coletados.
- **Nuvens nas bordas DO MAPA** (3 camadas, dentro do mundo, se movem com o mapa). Pisar nelas → sai.
- **Tela de recompensa** ao sair: mostra tudo coletado em tiles SFL + botão COLETAR.

### Controles
- PC: setas / WASD
- Mobile: analógico virtual (canto inferior esquerdo)

### O QUE FALTA (próximos passos, em ordem de prioridade que o Michel definiu)

1. **Personagem animado real:** hoje mostra um EMOJI 🧑‍🌾 como fallback. O personagem animado do SFL SÓ aparece quando o Michel estiver **conectado ao SFL** (a animação vem das roupas equipadas do bumpkin dele). A URL é montada assim:
   - `https://animations.sunflower-land.com/animated_webp/0_v1_{tokenUri}/idle` (parado)
   - `.../animated_webp/0_v1_{tokenUri}/walking` (andando)
   - O `{tokenUri}` vem dos IDs das roupas equipadas, na ordem dos Slots (Background=0, Body=1, Hair=2, Shirt=3, Pants=4, Dress=5, Shoes=6, Tool=7...). Bumpkin padrão de teste: `0_1_5_13_20_0_22_23`.
   - **AÇÃO PENDENTE:** quando o Michel conectar, pedir pra ele rodar `console.log(window._sflLastSession?.data?.bumpkin)` no console e confirmar onde estão as roupas (`equipped`) pra montar o tokenUri certo. A função é `buildBumpkinTokenUri(eq)` e `getBumpkinAnimUrls()`.
   - **BÔNUS descoberto:** o SFL tem animações `axe` (machado) e `mining` (picareta) — usar no sistema de coleta futuro.

2. **Sistema de coleta com botão de mão ✋** (estilo Last Day on Earth): em vez de tocar no recurso, vai ter um botão de mão. Ao apertar, o personagem anda até o recurso MAIS PRÓXIMO, e toca a animação de picareta (pedra) ou machado (árvore). As animações de "spark" existem nos arquivos do SFL: `resources/stone/stone_rock_spark.png`, etc. Hoje a coleta é "toca pra bater" (placeholder).

3. **3 baús + 3 chaves** (PENDENTE — precisa das URLs): Michel quer espalhar baús no mapa aleatoriamente com porcentagem (igual as árvores). O SFL tem 3 chaves oficiais: **Treasure Key, Rare Key, Luxury Key**, e os baús correspondentes. **AÇÃO:** no Claude Code com o repo do SFL clonado localmente, achar os arquivos de imagem dos baús/chaves (procurar por `chest`, `key`, `treasure` nas pastas `src/assets/`). No chat web não deu pra confirmar as URLs (CDN bloqueia + rate limit da API do GitHub).

4. **Inimigos** que spawnam (definir depois): provavelmente usam as cartas de dino. Atacam o player? A definir. Michel disse "primeiro a parte visual".

5. **Dar os recursos no SFL ao sair:** quando o jogador sai com a mochila, chamar uma ação SFL (ex: `Claim_explore_loot`) pra creditar madeira/pedra de verdade. Michel vai criar essa ação no SFL depois. Hoje só mostra a tela de recompensa (sem creditar).

---

## 5. ASSETS DO SFL — onde achar (MUITO IMPORTANTE)

Os assets do SFL ficam em 2 lugares. **Atenção ao bloqueio de ambiente:**

### A) Repo principal do SFL no GitHub (clone local resolve tudo)
`github.com/sunflower-land/sunflower-land` — clonar localmente dá acesso a TODOS os assets como arquivos. Pastas úteis:
- `src/assets/resources/` → pedras (`iron_rock.png`, `gold_rock.png`, etc)
- `public/world/` → decorações de mundo (`palm_tree.webp`, `sand.webp`)
- `src/assets/sfts/` → árvores (`baobab_tree.webp`)
- Tilesheet completo: `src/assets/map/` (tilesheet.png — 1024×1024, tiles de 16px)

### B) CDN do SFL (funciona no navegador do Michel, mas dá 403 no ambiente do Claude web)
Base: `https://sunflower-land.com/game-assets/`
- **Árvore (node):** `resources/tree.png`
- **Pedra (node):** `resources/stone_rock.png` ⚠️ (NÃO `stone.png` — esse é o item coletado!)
- **Bordas de painel:** `ui/panel/light_border.png`, `ui/panel/dark_border.png`, `ui/panel/tab_border_middle.png`
- **Nuvens de borda:** `land/clouds/cloud_1.webp`, `land/clouds/main_clouds_top/bottom/left/right.webp`
- **Ícone X:** `icons/close.png`
- **Item images do DinoHunt:** `https://production-minigame-editor-assets.s3.us-east-1.amazonaws.com/minigames/dinohunt/items/{id}.png` (helper: `sflItemImageUrl(id)`)

### C) Assets do repo do Michel (carregam em qualquer lugar)
Base: `https://raw.githubusercontent.com/michelsouza77/Mine-defender/main/Src-portal-sunflower/src/assets/`
- `ui/3x3_bg.png` (tile de grama), `icons/codex.webp`, `icons/letter_disc.png`, `icons/settings_disc.png`, etc.

> **REGRA DE OURO:** o CDN `sunflower-land.com/game-assets` retorna **403 no ambiente do Claude** (web), mas **funciona no navegador do Michel**. Já o `raw.githubusercontent.com` funciona em ambos. No Claude Code com o repo clonado **localmente**, você acessa tudo direto sem esse problema — é a grande vantagem de ir pro PC.

---

## 6. FÓRMULA DE BORDA SFL (o visual do jogo todo)

Todo painel/slot/aba usa `border-image` (não border normal). Constante: `PIXEL_SCALE = 2.625`.

```css
border-style: solid;
border-width: 5.25px;
border-image: url(LIGHT_OR_DARK_BORDER) 20%;
border-image-repeat: stretch;
border-radius: 13px;
image-rendering: pixelated;
box-sizing: border-box; /* SEMPRE — senão a borda estoura a largura (bug recorrente!) */
```

Cores de fundo: `#e4a672` (painel claro), `#b96f50` (slots), `#c98a64` (painel de fundo escuro da "escadinha"), `#e9b98a` (card claro).

**Aba conectada ("escadinha"):** a aba ativa usa `tab_border_middle.png` + um `::after` que cobre a costura entre a aba e o corpo. Painel de fundo escuro (#c98a64) envolve o corpo claro (#e4a672).

**Cores de texto:** use escuras (`#2d1810`, `#3a2410`, `#4a2e14`, `#6b4a28`). As variáveis `--sfl-cream`, `--parch-dark`, `--gold-dim` são CLARAS e somem no bege.

**Fonte dos números:** `'SFLSecondary'` (a mesma das moedas). Pra textos, `'Pixelify Sans'`.

---

## 7. BUGS RECORRENTES (aprenda com eles)

1. **Especificidade CSS:** regras tipo `.panel-content .item-qty` (0-2-0) sobrescrevem `.item-qty` (0-1-0). Se uma mudança de cor/tamanho "não aplica", procure uma regra mais específica ou `!important` numa media-query. (Foi a causa do bug "a cor não muda".)
2. **Borda estoura largura:** sempre `box-sizing:border-box` em elementos com `width:100%` + border-image. (Causou vários overflows.)
3. **Fonte SFLSecondary renderiza fina/pequena** no mesmo px que Pixelify — aumente o tamanho ou use Pixelify pra números pequenos.
4. **onerror com aspas simples dentro de string i18n quebra o JS** — nunca coloque `onerror="...'...'..."` dentro de valor de string i18n.

---

## 8. WORKFLOW DE VALIDAÇÃO (faça a cada edição)

No chat web eu usava (adapte pro seu ambiente):
1. Extrair o último `<script>` não-module e rodar `node -c` (checar sintaxe JS).
2. Conferir que nenhuma linha passa de 1000 chars (o editor web do GitHub quebra com linhas longas — relevante se o Michel editar lá).
3. Renderizar com Playwright (viewport 480×800) pra conferir visualmente. Como o CDN dá 403 no ambiente do Claude, as imagens do SFL não aparecem no render — só o Michel as vê. Imagens de `raw.githubusercontent` aparecem.

No Claude Code local: você pode abrir o `index.html` direto no navegador pra testar, o que é melhor.

---

## 9. SOBRE O ARQUIVO

- Edite SEMPRE o `index.html` real (não recrie do zero — são 11k+ linhas com muito trabalho).
- Método de edição preferido: substituição de string precisa (str_replace) ou Python `html.replace(old, new)`.
- O jogo tem `addTestCards()` que dá cartas de teste quando NÃO está conectado ao SFL (pra testar sem conta).

---

## 10. PRIMEIRO PASSO QUANDO ABRIR NO CLAUDE CODE

1. Confirme que tem o `index.html` do jogo na mão (peça pro Michel se não tiver).
2. Clone o repo do SFL localmente (`git clone https://github.com/sunflower-land/sunflower-land`) pra ter os assets.
3. Pergunte ao Michel em qual dos pendentes da seção 4 ele quer começar (provavelmente baús ou coleta com botão de mão).
4. Para o personagem animado: peça o `console.log(window._sflLastSession?.data?.bumpkin)` quando ele estiver conectado.

---

*Documento gerado em 19/06/2026 para passagem de contexto. O Michel tem o histórico completo da conversa no Claude (claude.ai) se precisar de detalhes específicos.*
