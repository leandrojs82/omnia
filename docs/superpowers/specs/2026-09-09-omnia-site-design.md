# OmnIA — Canivete Suíço Digital

**Design doc** — 2026-09-09

## Objetivo

Um site que reúne 90+ ferramentas utilitárias (texto, cálculo, data, cripto, dev,
saúde, finanças) em uma única página, sem backend, sem conta de usuário e sem
build step. O usuário abre o arquivo e usa.

## Decisões tomadas

| Decisão | Escolha |
|---|---|
| Arquitetura | Single-file `index.html` (HTML + CSS + JS inline) |
| Entrega | Em fases — Fase 1 entrega core + 24 ferramentas |
| Visual | Dark tech / neon, com toggle para tema claro |
| Backend | Nenhum. Persistência via `localStorage` |
| Dependências | Nenhuma obrigatória. APIs públicas apenas nas ferramentas "Online" |

## Arquitetura

Três camadas dentro do mesmo arquivo:

### Core (~400 linhas, escrito uma vez)

- **Registry** — array `TOOLS[]`. Cada ferramenta é
  `{ id, cat, nome, desc, tags, tipo, render() }`.
  - `id`: slug único, usado na URL (`gerador-senha`)
  - `cat`: id da categoria (uma das 14)
  - `tags`: termos extras para a busca encontrar
  - `tipo`: `nativa` | `online` | `script` (ver Classes de ferramenta)
  - `render(el)`: recebe o container e monta a UI dentro dele
- **Router** — baseado em `location.hash`. `#/gerador-senha` renderiza a
  ferramenta de id `gerador-senha`. Hash vazio ou desconhecido → home.
  URLs são favoritáveis e compartilháveis.
- **Busca** — campo no header, atalho `Ctrl+K`. Filtro por substring
  case/acento-insensitive sobre `nome`, `desc` e `tags`. Enter abre o primeiro
  resultado.
- **UI kit** — helpers que padronizam a aparência e reduzem cada ferramenta a
  15–30 linhas:
  - `ui.input(label, attrs)` / `ui.textarea(label, attrs)` / `ui.select(label, opts)`
  - `ui.btn(texto, onClick, variante)`
  - `ui.out(conteudo)` — bloco de resultado em fonte mono, com botão copiar
  - `ui.copy(texto)` — clipboard + toast de confirmação
  - `ui.row(...)` / `ui.grid(...)` — layout

### Shell

- **Sidebar** — 14 categorias colapsáveis, cada uma listando suas ferramentas.
  Seção fixa no topo com Favoritos e Recentes. No mobile vira drawer com overlay.
- **Header** — logo OmnIA, campo de busca, toggle de tema.
- **Main** — área onde `render()` da ferramenta ativa escreve. Limpa a cada
  troca de rota.
- **Home** — grade das 14 categorias + bloco de Recentes.

### Tools

Cada ferramenta é uma função isolada e independente. Nenhuma ferramenta importa,
chama ou conhece outra. Adicionar a ferramenta nº 91 custa o mesmo que a nº 5:
um objeto novo no array `TOOLS[]`.

Essa é a propriedade central do design — é ela que torna as fases seguintes
viáveis sem tocar no core.

## Classes de ferramenta

Cada ferramenta exibe um selo honesto sobre como opera:

| Selo | `tipo` | Comportamento | Exemplos |
|---|---|---|---|
| 🟢 Nativa | `nativa` | Roda 100% offline no browser, resultado real | CPF, senha, Morse, IMC, dias úteis, churrasco |
| 🔵 Online | `online` | Chama API pública real | CEP (ViaCEP), CNPJ (BrasilAPI), QR Code (QR Server) |
| 🟡 Script | `script` | Browser não consegue; a ferramenta gera código Python/JS pronto para copiar e rodar localmente | Comprimir PDF, OCR, Whois, teste de portas |

Regra: nenhum resultado falso ou placeholder. Se o browser não faz, o site
entrega o script que faz.

Ferramentas `online` tratam falha de rede exibindo mensagem de erro clara no
bloco de resultado, nunca travando a UI.

## Persistência (`localStorage`)

| Chave | Conteúdo |
|---|---|
| `omnia.favoritos` | array de ids de ferramentas |
| `omnia.recentes` | array de ids, máximo 6, mais recente primeiro |
| `omnia.tema` | `"dark"` \| `"light"` |
| `omnia.notepad` | conteúdo do Notepad Online |

Todo acesso a `localStorage` é envolvido em `try/catch` — o site funciona
normalmente se o storage estiver bloqueado.

## Visual

- Tema padrão: escuro. Fundo grafite, superfícies elevadas em cinza-azulado.
- Acento: ciano para ações primárias, violeta para destaques secundários.
- Tipografia: Inter (ou system stack) na interface; fonte monoespaçada nos
  blocos de resultado e nos scripts gerados.
- Cantos arredondados, bordas sutis, sem sombras pesadas.
- Toggle claro/escuro no header, persistido.
- Cores definidas como custom properties CSS em `:root`, redefinidas sob
  `[data-tema="light"]`. Nenhuma cor hardcoded fora dos tokens.

## Responsividade

- Desktop: sidebar fixa à esquerda, main ao lado.
- Mobile (< 768px): sidebar vira drawer aberto por botão hambúrguer, com
  overlay; fecha ao escolher uma ferramenta.
- Blocos de resultado largos (tabelas, scripts) rolam horizontalmente dentro do
  próprio container. A página nunca rola na horizontal.

## Escopo da Fase 1

Core + shell + tema completos, mais 24 ferramentas totalmente funcionais.
As 24 foram escolhidas por cobrirem todos os padrões de UI que as fases
seguintes vão reaproveitar: entrada de texto livre, formulário com múltiplos
campos, lista, e resultado copiável.

| Categoria | Ferramentas |
|---|---|
| Texto | inverter · maiúsculas/minúsculas · remover duplicadas · ordem alfabética · localizar e substituir |
| Análise | contador de caracteres |
| Aleatório | sorteio · gerador de senha · Mega-Sena · dado |
| Cripto | binário · Morse · cifra de César |
| Data e Hora | contar dias entre datas · idade exata · dias úteis |
| Matemática | regra de 3 · porcentagem · números romanos · conversor de base |
| Dev | gerador de CPF · gerador de CNPJ · UUID v4 · Lorem Ipsum |
| Saúde | IMC |

As 14 categorias aparecem na sidebar desde a Fase 1. Categorias ainda sem
ferramentas exibem "em breve" — a estrutura de navegação já é a final.

## Fases seguintes

Cada fase adiciona um lote de ferramentas por categoria, apenas acrescentando
objetos ao `TOOLS[]`. O core, o shell e o CSS não são alterados.

Ordem sugerida: Finanças → Documentos e Dados Públicos → Multimídia (scripts) →
Internet e Redes → Diversos e Viagem → restante de Texto/Análise/Data.

## Não faz parte deste projeto

- Backend, banco de dados ou autenticação
- Contas de usuário ou sincronização entre dispositivos
- Upload de arquivos para servidor
- Analytics ou rastreamento
- Build step, bundler ou gerenciador de pacotes

## Critérios de sucesso da Fase 1

1. `index.html` abre com duplo clique, sem servidor, e funciona.
2. As 24 ferramentas produzem resultado correto — verificado caso a caso.
3. Cada ferramenta tem URL própria que funciona ao recarregar a página.
4. Busca encontra qualquer uma das 24 pelo nome ou por um sinônimo.
5. Favoritos, recentes e tema sobrevivem ao reload.
6. Layout íntegro em 375px de largura, sem rolagem horizontal.
7. Adicionar uma ferramenta nova exige tocar em um único lugar do arquivo.
