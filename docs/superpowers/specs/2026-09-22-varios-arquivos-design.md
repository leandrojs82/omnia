# Criar Arquivo — nomes em campo próprio, vários arquivos e arquivo vazio

**Design doc** — 2026-09-22

Segunda rodada sobre a ferramenta **Criar Arquivo**, que hoje reconhece 18
tipos de conteúdo colado e baixa um arquivo com o nome e a extensão certos.
Spec da primeira rodada: `2026-09-22-criar-arquivo-design.md`.

## Problema

Três buracos apareceram no uso real:

1. **O nome não tem um campo de verdade.** Ele só existe como sugestão do
   reconhecimento. Quem já sabe o nome que quer precisa colar conteúdo antes
   para o campo aparecer preenchido.
2. **Não dá para pedir vários arquivos de uma vez.** Criar
   `docker-compose.yml`, `.env` e `.gitignore` exige três passagens pela
   ferramenta.
3. **Não dá para baixar um arquivo vazio.** O botão desabilita com o
   conteúdo em branco, então um `.gitkeep` ou um `.env` vazio é impossível.

## Fora de escopo

**Estrutura de pastas.** Uma versão anterior deste spec previa colar uma
árvore (`├── sql/`) e receber a hierarquia num zip. Foi retirada a pedido:
a ferramenta cria arquivos, não árvores. O parser de árvore, a inferência
de profundidade e as pastas vazias saem inteiros. O gerador de zip
permanece, agora com o papel de empacotar vários arquivos lado a lado.

## Decisões

| Decisão | Escolha |
|---|---|
| Nomes | Campo próprio, um nome por linha |
| Um arquivo | Baixa direto, sem zip |
| Dois ou mais | Um `arquivos.zip` com todos, sem pastas dentro |
| Conteúdo | Só se aplica quando há exatamente um nome |
| Arquivo vazio | Permitido: basta o nome |
| Reconhecimento | Continua, agora preenchendo o campo de nomes |
| Dependências novas | Nenhuma |

### Por que zip a partir de dois, e não sempre

O navegador entrega um arquivo por vez. Vários downloads seguidos fazem
Chrome e Edge pedirem permissão numa barra — negar ou ignorar deixa o
usuário com só o primeiro arquivo, sem entender por quê. O zip evita isso
por completo.

Um arquivo sozinho dentro de um zip, por outro lado, é só um passo a mais
para o usuário desfazer. Por isso o corte fica em dois.

### Por que o reconhecimento sobrevive

Ele deixa de ser o único caminho para o nome e passa a ser um atalho: cola
o conteúdo, o campo de nomes se preenche sozinho. Quem já sabe o nome
digita e ignora a sugestão.

## Arquitetura

Uma função pura nova em `lib`, mais a reescrita do `render` da ferramenta.
`lib.detectarTipo`, `lib.ambiguo` e `lib.nomeSeguro` continuam como estão.

### `lib.zip(entradas)` → `Uint8Array`

`entradas` é `[{ nome, conteudo }]`, `conteudo` string (vazia para arquivo
vazio).

Escreve um ZIP pelo método *store*, sem compressão: cabeçalho local por
entrada, diretório central, registro de fim. Precisa de CRC-32 (zero para
conteúdo vazio) e da data/hora no formato DOS. Nomes em UTF-8, com a flag
de UTF-8 ligada no campo de propósito geral.

Sem pastas: todo nome é uma entrada na raiz do zip. Uma barra num nome é
responsabilidade do `lib.nomeSeguro`, que já a transforma em hífen.

Função pura: recebe dados, devolve bytes, não toca no DOM.

### `lib.nomesDeLinhas(texto)` → `{ nomes, repetidos }`

Quebra o texto em linhas, descarta linhas vazias e espaços nas pontas,
passa cada nome por `lib.nomeSeguro`, e remove duplicatas preservando a
primeira ocorrência. `repetidos` lista os nomes que apareceram mais de uma
vez, para a interface avisar em vez de descartar em silêncio.

### O `render`

Dois campos, nesta ordem:

1. **Nomes dos arquivos** — textarea, um por linha.
2. **Conteúdo** — textarea. Habilitado apenas quando há exatamente um nome;
   com dois ou mais fica desabilitado, com a explicação ao lado.

O painel de reconhecimento continua onde está, abaixo do conteúdo, e
preenche o campo de nomes quando ele está vazio — nunca sobrescreve o que o
usuário digitou.

O botão diz o que vai fazer:

| estado | botão |
|---|---|
| um nome | **Baixar docker-compose.yml** |
| dois ou mais | **Baixar .zip com 3 arquivos** |
| nenhum nome | desabilitado |

## Erros

| situação | o que acontece |
|---|---|
| Nome repetido na lista | O zip leva um só, e um aviso diz qual se repetiu |
| Linha em branco no meio | Ignorada, sem aviso — é ruído comum ao colar |
| Nome com caractere proibido | `lib.nomeSeguro` limpa, com o aviso que já existe |
| Nenhum nome e nenhum conteúdo | Botão desabilitado |
| Falha ao montar o zip | Toast nomeando a falha, como no download comum |

## Arquivo vazio

Um nome sem conteúdo baixa um arquivo de zero byte. O painel diz **"Sem
conteúdo — vai baixar um arquivo vazio"**. Nenhum reconhecimento de tipo:
não há o que reconhecer, e inventar um seria resultado falso.

## Testes

Na suíte embutida (`#/_testes`):

- `lib.nomesDeLinhas`: três nomes; linhas em branco no meio e nas pontas;
  nome repetido listado em `repetidos` e presente uma vez só em `nomes`;
  nome com `/` limpo pelo `nomeSeguro`; texto vazio devolvendo lista vazia
- `lib.zip` com um arquivo vazio, um com conteúdo e dois arquivos:
  assinatura `PK\x03\x04` no início, contagem de entradas no diretório
  central, CRC de um conteúdo conhecido, e o nome de cada entrada legível
  nos bytes

## Verificação que o plano deve fazer antes da implementação

O gerador de zip é a única parte deste spec que dá para conferir fora do
navegador: rodar em Node, gravar o `.zip` em disco, descompactar e comparar
os arquivos com os esperados. Obrigatório antes de escrever o plano — a
mesma disciplina que pegou o erro de ordenação na primeira rodada.

## Critérios de sucesso

1. Digitar `docker-compose.yml` e colar conteúdo baixa esse arquivo com
   esse conteúdo.
2. Digitar `.gitkeep` sem conteúdo baixa um arquivo de zero byte.
3. Digitar três nomes baixa um zip que, descompactado, produz os três
   arquivos vazios com os nomes certos.
4. Colar conteúdo com o campo de nomes vazio ainda preenche o nome
   sugerido, como antes.
5. Um nome digitado à mão nunca é sobrescrito pelo reconhecimento.
6. Com dois ou mais nomes, o campo de conteúdo fica desabilitado e a razão
   aparece na tela.
7. A suíte segue verde, incluindo com `localStorage` bloqueado.
8. Nenhuma dependência nova; tudo funciona sem internet.
