# Criar Arquivo — árvore de pastas e arquivo vazio

**Design doc** — 2026-09-22

Segunda rodada sobre a ferramenta **Criar Arquivo**, que hoje reconhece 18
tipos de conteúdo colado e baixa um arquivo com o nome e a extensão certos.
Spec da primeira rodada: `2026-09-22-criar-arquivo-design.md`.

## Problema

Dois buracos apareceram no uso real:

1. **Não dá para baixar um arquivo vazio.** O botão desabilita quando o
   conteúdo está em branco, então criar um `.gitkeep` ou um `.env` vazio é
   impossível pela ferramenta.
2. **Não dá para criar uma estrutura de pastas.** Uma árvore como a que se
   cola de um README precisa virar pastas e arquivos de verdade:

```
meu-laboratorio/
├── docker-compose.yml
├── .env
├── .env.example
├── .gitignore
├── sql/
│   └── init/
├── pipelines/
├── dados/
│   ├── entrada/
│   └── amostras/
└── docs/
```

## A restrição que define a solução

**O navegador não cria pastas no disco.** Não existe API que faça isso a
partir de uma página comum; `<a download>` entrega um arquivo, e só. As
alternativas seriam a File System Access API (só Chrome, e indisponível em
`file://`) ou pedir ao usuário que rode um script.

A saída é um **.zip**: o formato carrega a hierarquia inteira, inclusive
pastas vazias, e descompactar é um gesto que todo mundo já conhece. O
gerador cabe em JavaScript puro sem biblioteca, então a regra de arquivo
único e offline continua valendo.

## Decisões

| Decisão | Escolha |
|---|---|
| Onde mora | A mesma ferramenta `criar-arquivo` — a árvore vira o 19º tipo reconhecido |
| Saída da árvore | Um `.zip` baixado, nomeado pela raiz da árvore |
| Compressão | Método *store* (sem compressão) — simples, e o conteúdo é vazio mesmo |
| Arquivo vazio | Permitido: basta um nome |
| Dependências novas | Nenhuma |

### Por que a mesma ferramenta, e não uma nova

Colar e descobrir o que é já é o gesto da ferramenta. Uma ferramenta
separada obrigaria o usuário a saber o que tem em mãos antes de colar —
exatamente o trabalho que esta ferramenta existe para evitar.

### Por que zip, e não um script

Um script `.sh`/`.ps1` foi considerado: seria honesto, funcionaria por SSH
e o usuário leria antes de rodar. Perdeu porque exige um segundo passo num
terminal, enquanto o zip resolve com um clique e um duplo clique. O script
continua sendo a saída certa se algum dia o destino for uma máquina remota
— fica registrado aqui como a alternativa considerada, não como dívida.

## Arquitetura

Duas funções puras novas em `lib`, uma entrada nova em `TIPOS`, e um ramo
no `render` da ferramenta. Nada mais do site muda.

### `lib.arvoreParaCaminhos(texto)` → `Caminho[]`

```js
{ caminho: "meu-laboratorio/sql/init/", pasta: true }
{ caminho: "meu-laboratorio/docker-compose.yml", pasta: false }
```

Lista na ordem em que aparecem, com o caminho completo desde a raiz.
Devolve lista vazia quando não consegue ler nada.

Entende:

- **Traços de caixa** — `├──`, `└──`, `│`, e a variante ASCII `|--`, `` `-- ``
- **Só indentação**, sem traço nenhum (2 ou 4 espaços por nível)
- **Profundidade** pela largura do prefixo antes do nome, em passos de 4
  colunas para as variantes com traço
- **Barra no fim** marca pasta; sem barra, arquivo
- **Nome sem barra que tem filhos é pasta assim mesmo** — a barra esquecida
  é o erro mais comum ao escrever uma árvore à mão, e tratar como arquivo
  produziria uma estrutura errada em silêncio
- **Comentário depois do nome** — tudo a partir de `#` ou de dois espaços
  seguidos é descartado, então `.env  # copie de .env.example` vira `.env`
- **Linhas vazias** no meio são ignoradas
- **CRLF** tratado como LF

Recusa, devolvendo lista vazia: um caminho com `..` em qualquer segmento
(tentativa de escapar da raiz) ou um caminho absoluto.

### `lib.zip(entradas)` → `Uint8Array`

`entradas` é `[{ caminho, conteudo }]`, onde `conteudo` é string (vazia para
arquivo vazio) e um caminho terminado em `/` é uma pasta.

Escreve um ZIP pelo método *store*: cabeçalho local por entrada, diretório
central, e o registro de fim. Precisa de CRC-32 (zero para conteúdo vazio) e
da codificação de data/hora no formato DOS. Nomes em UTF-8, com a flag de
UTF-8 ligada no campo de propósito geral.

Pasta é uma entrada de tamanho zero cujo nome termina em `/` — é assim que o
formato representa diretório, e é o que faz `sql/init/` chegar ao disco
mesmo sem nada dentro.

Função pura: recebe dados, devolve bytes, não toca no DOM.

### O tipo `arvore` em `TIPOS`

Sinais, do mais para o menos decisivo:

| sinal | peso | o que reconhece |
|---|---|---|
| `├──` ou `└──` no início de uma linha (após espaços) | 8 | traço de caixa |
| `│` em qualquer linha | 4 | haste de continuação |
| `\|--` ou `` `-- `` | 6 | variante ASCII |
| primeira linha termina em `/` | 3 | raiz declarada |
| três ou mais linhas terminando em `/` | 3 | várias pastas |

Peso alto de propósito: uma árvore é inconfundível quando tem traços, e
precisa ganhar do `md` (que também vê listas) e do `txt`.

### O ramo no `render`

Quando o tipo escolhido é `arvore`, a ferramenta troca de modo:

- O painel diz **"Parece uma árvore de pastas — 7 pastas, 4 arquivos"** e
  lista **todos** os caminhos que vai criar, marcando as pastas vazias.
- O campo de nome vem com a raiz da árvore mais `.zip`
  (`meu-laboratorio.zip`), editável como sempre.
- O botão vira **"Baixar .zip"**.

Nos demais tipos, nada muda.

## Mostrar o que entendeu

Ler uma árvore em ASCII é palpite, e um palpite errado aqui não produz um
nome estranho — produz uma estrutura errada no disco do usuário. Por isso a
lista completa de caminhos aparece **antes** do download, não depois. O
usuário confere e corrige a árvore se algo saiu torto.

É a mesma regra do resto do projeto (*nenhum resultado falso*), aplicada ao
caso em que o custo de errar é mais alto.

## Arquivo vazio

O botão passa a habilitar quando existe um nome, mesmo sem conteúdo. Com o
campo em branco, o painel diz **"Sem conteúdo — vai baixar um arquivo
vazio"** e o nome sugerido é `arquivo.txt`. Nada de reconhecimento de tipo:
não há o que reconhecer, e inventar um seria resultado falso.

## Erros

| situação | o que acontece |
|---|---|
| Árvore que não produz nenhum caminho | Mensagem dizendo o formato esperado, sem zip |
| Caminho com `..` ou absoluto | A árvore inteira é recusada, com aviso do motivo |
| Nome do zip com caractere proibido | `lib.nomeSeguro` limpa, com o aviso que já existe |
| Falha ao montar o zip | Toast nomeando a falha, como no download comum |

## Testes

Na suíte embutida (`#/_testes`):

- A árvore do exemplo acima, conferindo os caminhos um a um e quais são
  pasta
- Variante ASCII (`|--`, `` `-- ``)
- Árvore só com indentação, sem traço
- Barra esquecida num nome que tem filhos
- Comentário depois do nome
- Linha vazia no meio
- `..` num segmento → lista vazia
- Texto que não é árvore → lista vazia
- `lib.zip` com uma pasta vazia, um arquivo vazio e um arquivo com conteúdo:
  conferir a assinatura `PK\x03\x04`, a contagem de entradas no diretório
  central e o CRC de um conteúdo conhecido
- O tipo `arvore` vence `md` e `txt` na árvore do exemplo

## Verificação que o plano deve fazer antes da implementação

O gerador de zip é a única parte deste spec que dá para conferir de
verdade fora do navegador: rodar em Node, gravar o `.zip` em disco,
descompactar e comparar a estrutura com a esperada. Essa verificação é
obrigatória antes de escrever o plano — a mesma disciplina que pegou o erro
de ordenação na primeira rodada.

## Critérios de sucesso

1. Colar a árvore do exemplo lista 11 caminhos, com `sql/init/`,
   `dados/entrada/`, `dados/amostras/`, `pipelines/` e `docs/` marcadas como
   pasta.
2. O zip baixado, descompactado, produz exatamente essa estrutura —
   incluindo as pastas vazias.
3. Uma árvore com a barra esquecida em `sql` ainda cria `sql` como pasta.
4. Digitar `.gitkeep` sem conteúdo baixa um arquivo de zero byte com esse
   nome.
5. Colar conteúdo comum continua se comportando como antes — a árvore não
   rouba nenhum dos 18 tipos.
6. A suíte segue verde, incluindo com `localStorage` bloqueado.
7. Nenhuma dependência nova; tudo funciona sem internet.
