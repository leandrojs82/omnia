<p align="center">
  <img src="marca/omnia-logo.svg" width="240" alt="OmnIA">
</p>

<p align="center"><strong>Canivete suíço digital</strong> — 27 ferramentas em um único arquivo HTML.<br>
Abre com duplo clique. Sem instalar nada, sem servidor, sem conta. Nada é enviado a lugar nenhum.</p>

---

## Como usar

1. Baixe o [`index.html`](index.html).
2. Abra no navegador (duplo clique).

Pronto. Funciona offline. A busca (`Ctrl+K`) acha por nome ou sinônimo — `senha`, `cpf`, `regex`, `timer`.

Tema escuro por padrão, claro no botão do canto. Favoritos e ferramentas recentes ficam salvos no próprio navegador.

## Ferramentas

| Categoria | Ferramentas |
|---|---|
| **Manipulação de Texto** | Inverter texto · Maiúsculas e minúsculas · Remover linhas duplicadas · Ordem alfabética · Localizar e substituir (com regex) |
| **Análise de Texto** | Contador de caracteres, palavras, linhas e parágrafos |
| **Aleatório** | Sorteio · Gerador de senha · Números da Mega-Sena · Dado |
| **Criptografia** | Código binário · Código Morse · Cifra de César |
| **Data e Hora** | Contar dias entre datas · Idade exata · Dias úteis · Formatar data e hora · Pomodoro com sons |
| **Números e Matemática** | Regra de 3 · Porcentagem · Números romanos · Conversor de base |
| **Desenvolvedor** | Gerador de CPF · Gerador de CNPJ · UUID v4 · Lorem Ipsum |
| **Saúde** | Calculadora de IMC |

Os geradores de CPF e CNPJ produzem números matematicamente válidos pelo algoritmo oficial, para testar sistemas. Não pertencem a ninguém.

O Pomodoro tem chuva, natureza e música ambiente **sintetizadas no navegador** (Web Audio), mais seis peças clássicas de domínio público e a opção de colar um link do YouTube.

## O que precisa de internet

Quase nada. Três coisas usam a rede quando disponível e degradam com elegância sem ela:

- **Fontes** (Google Fonts) — sem internet, cai em Georgia / Segoe UI / Consolas.
- **Clássicas no Pomodoro** — gravações de Bach, Satie e Debussy hospedadas no Wikimedia Commons (CC0 / CC BY / domínio público).
- **YouTube no Pomodoro** — só se você colar um link.

Tudo o mais roda local, incluindo os sons sintetizados.

## Testes

O arquivo carrega seu próprio conjunto de testes. Abra:

```
index.html#/_testes
```

Aparece o placar (hoje **82/82**). Os testes cobrem a lógica pura de cada ferramenta — os dígitos verificadores de CPF e CNPJ são conferidos contra 200 documentos gerados por execução, e a idade exata é validada por um teste de propriedade com 500 pares de datas aleatórias. A suíte fica verde também com o `localStorage` bloqueado.

## Como adicionar uma ferramenta

Uma ferramenta é um único `TOOLS.push({...})` — sidebar, busca, home e roteamento derivam do registro sozinhos.

```js
TOOLS.push({
  id: "minha-ferramenta",          // vira a URL: #/minha-ferramenta
  cat: "texto",                    // uma das 8 categorias
  tipo: "nativa",                  // nativa | online | script
  nome: "Minha Ferramenta",
  desc: "O que ela faz, em uma linha",
  tags: ["sinonimo", "outro nome"],
  render(el) {
    const entrada = ui.area("Texto");
    const saida = ui.saida();
    ui.montar(el, entrada, ui.btn("Rodar", () => saida.set(lib.minhaFuncao(entrada.value))), saida);
  }
});
```

Regras da casa:

- **Lógica em `lib`, interface em `render`.** `lib.minhaFuncao` é pura (entra valor, sai valor, sem DOM) e ganha um `TESTES.push`. `render` só monta a tela com o kit `ui`.
- **Campos numéricos passam por `lib.inteiro(valor, min, max, padrao)`** — um `-5` digitado não pode derrubar nada.
- **Nenhum resultado falso.** Entrada inválida vira mensagem clara, nunca um número errado com cara de certo.
- **Cores só por token** (`var(--acento)`, `var(--texto-fraco)`…). Nenhuma cor literal fora do bloco `:root`.

Kit `ui` disponível: `campo`, `area`, `select`, `btn`, `saida`, `linha`, `montar`, `nota`, `separador`, `toast`, `copiar`.

## Estrutura do arquivo

Um `index.html` de ~2.300 linhas, nesta ordem: tokens e CSS → markup do shell → `lib` (funções puras) → `TESTES` → `store` (localStorage) → `ui` (kit) → `TOOLS` (registro) → router → busca → init.

Documentos de projeto em [`docs/superpowers/`](docs/superpowers/): o spec de design e o plano de implementação da Fase 1. A marca está em [`marca/`](marca/), com folha de apresentação.

## Marca

Quatro lâminas abrindo em leque a partir de um eixo — a mais aberta, em terracota, é a ferramenta em uso. Arquivos SVG para fundo escuro, claro e monocromático em [`marca/`](marca/).

## Licença

A definir.

As gravações clássicas usadas no Pomodoro têm licenças próprias, creditadas na própria ferramenta enquanto tocam: Kimiko Ishizaka (CC0), Kevin MacLeod (CC BY 3.0), Laurens Goedhart (domínio público).
