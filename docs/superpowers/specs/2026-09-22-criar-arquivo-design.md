# Criar Arquivo — reconhecer o tipo do conteúdo e baixar

**Design doc** — 2026-09-22

## Problema

Você recebe um conteúdo pronto — de uma resposta de IA, de um tutorial, de um
colega — e precisa dele como arquivo no disco, com o nome e a extensão certos.
Hoje isso é: abrir o editor, criar arquivo, colar, lembrar se é `.yml` ou
`.yaml`, salvar na pasta certa.

A ferramenta encurta isso: cola o conteúdo, ela reconhece o que é, sugere o
nome, você baixa.

## O que ela não é

- **Não interpreta pedidos.** Colar "quero um docker-compose com nginx" não
  gera nada — ela lê o conteúdo que você colou, não a intenção.
- **Não extrai vários arquivos de um texto.** Um conteúdo colado, um arquivo
  baixado.
- **Não lê imagens.** Só texto.

Essas três foram consideradas e descartadas: as duas primeiras exigiriam uma
IA, a terceira exigiria OCR. Ambas quebrariam a regra de arquivo único e
offline.

## Decisões

| Decisão | Escolha |
|---|---|
| Categoria | Desenvolvedor (`dev`) |
| Tipo | `nativa` — roda 100% offline |
| Reconhecimento | Pontuação por sinais com peso; maior soma vence |
| Ambiguidade | Mostra os candidatos próximos como alternativas clicáveis |
| Fallback | `.txt` sempre presente como piso — a lista nunca volta vazia |
| Dependências novas | Nenhuma |

### Por que pontuação, e não a primeira regra que casar

Um `docker-compose.yml` **é** um YAML válido; um `package.json` **é** um JSON
válido. Com uma cascata de regras a ordem vira regra oculta e o segundo
colocado desaparece. Com pontuação, os dois aparecem e o usuário escolhe —
e cada regra continua sendo uma função pura testável isoladamente.

## Arquitetura

Duas funções puras em `lib`, uma ferramenta em `TOOLS`. Nenhuma outra parte
do site muda.

### `lib.detectarTipo(texto)` → `Candidato[]`

Devolve a lista ordenada por confiança decrescente. Nunca vazia.

```js
{ id: "docker-compose",        // identificador estável
  nome: "docker-compose.yml",  // nome sugerido do arquivo
  ext: ".yml",                 // extensão
  mime: "text/yaml",           // usado no Blob do download
  confianca: 0.86,             // 0–1
  sinais: ["services: no início", "image:", "ports:"] }
```

Cada tipo é uma entrada numa tabela `TIPOS`, com sinais e pesos:

```js
{ id: "docker-compose", nome: "docker-compose.yml", ext: ".yml", mime: "text/yaml",
  sinais: [
    { re: /^services:\s*$/m, peso: 5, label: "services: no início" },
    { re: /^\s+image:\s/m,   peso: 3, label: "image:" },
    { re: /^\s+ports:\s*$/m, peso: 2, label: "ports:" }
  ] }
```

A confiança é a soma dos pesos que bateram, dividida pela soma de todos os
pesos daquele tipo. Tipos com confiança zero ficam de fora da lista. O `.txt`
entra sempre, com confiança mínima fixa, para que exista sempre uma saída.

**Ambíguo** significa: o segundo colocado tem confiança de pelo menos 70% da
do primeiro. Nesse caso a interface mostra os dois; caso contrário, mostra o
primeiro e deixa os outros num rodapé discreto.

### Os tipos reconhecidos

`docker-compose.yml` · `Dockerfile` · `package.json` · `tsconfig.json` ·
`.env` · `.sql` · `.json` · `.yaml` · `.xml` · `.html` · `.css` · `.js` ·
`.py` · `.sh` · `.md` · `.csv` · `.ini`/`.conf` · `.gitignore` · `.txt`

### `lib.nomeSeguro(nome)` → `string`

Remove `/ \ : * ? " < > |`, corta espaços e pontos nas pontas, recusa os nomes
reservados do Windows (`CON`, `PRN`, `AUX`, `NUL`, `COM1`–`COM9`,
`LPT1`–`LPT9`) prefixando com `_`. Nome vazio depois da limpeza vira
`arquivo.txt`. Não mexe na extensão.

### A ferramenta

`render(el)` monta:

1. **Textarea** "Conteúdo do arquivo". O reconhecimento roda enquanto se
   digita, com atraso de 250 ms para não recalcular a cada tecla.
2. **Resultado do reconhecimento** — "Parece um `docker-compose.yml`" seguido
   dos sinais que bateram. Alternativas próximas viram botões: clicar troca o
   nome sugerido.
3. **Campo de nome**, preenchido com a sugestão e editável. Editar não
   desfaz o reconhecimento — o usuário manda no nome final.
4. **Botão Baixar** — `Blob` + `<a download>`, `URL.revokeObjectURL` depois.

Conteúdo vazio: nenhum reconhecimento, botão desabilitado, mensagem dizendo
o que fazer. Nome inválido: sanitizado ao baixar, com aviso do que ficou.

## Mostrar o porquê

A ferramenta nunca afirma um tipo sem dizer no que se baseou. "Parece um
`docker-compose.yml` — bateu `services:` no início, `image:`, `ports:`" é
verificável; "é um docker-compose" não é. Isso é a regra do projeto
(*nenhum resultado falso*) aplicada a um palpite: o palpite é honesto sobre
ser um palpite.

## Testes

Na suíte embutida (`#/_testes`):

- Uma amostra real de cada um dos 18 tipos; o candidato correto tem de vir em
  primeiro lugar.
- `docker-compose.yml` vence `.yaml` genérico, e `.yaml` genérico aparece
  como alternativa.
- `package.json` vence `.json` genérico.
- Texto vazio e texto sem sinal nenhum caem em `.txt` com lista de tamanho 1.
- `nomeSeguro`: caracteres proibidos, espaços nas pontas, nome reservado do
  Windows, nome que fica vazio depois da limpeza.
- Toda entrada de `TIPOS` tem `id` único, `mime` preenchido e pelo menos um
  sinal — teste de integridade da tabela, como já existe para `TOOLS`.

## Risco a verificar na implementação

`<a download>` com `Blob` em página aberta por `file://`. Funciona em Chrome e
Firefox atuais, mas precisa de confirmação prática antes de ser dado como
certo. Se falhar, o plano B é abrir o conteúdo em nova aba com o tipo MIME
correto, para o usuário salvar com Ctrl+S — e a ferramenta explica isso em vez
de simplesmente não fazer nada.

## Critérios de sucesso

1. Colar um `docker-compose.yml` de verdade sugere `docker-compose.yml`, não
   `.yaml` genérico.
2. Colar um `package.json` de verdade sugere `package.json`, não `.json`.
3. Colar prosa em português sugere `.txt` sem inventar tipo.
4. Os sinais que justificam o palpite aparecem na tela.
5. O arquivo baixado abre com o conteúdo idêntico ao colado, byte a byte.
6. Nome com `/` ou `:` não impede o download — é limpo, com aviso.
7. A suíte segue verde, incluindo com `localStorage` bloqueado.
8. Nenhuma dependência nova; a ferramenta funciona sem internet.
