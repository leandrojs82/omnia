# Vários arquivos, nome em campo próprio e arquivo vazio — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A ferramenta Criar Arquivo passa a ter um campo de nomes (um por linha), baixa vários arquivos num `.zip`, e aceita baixar um arquivo vazio.

**Architecture:** Duas funções puras novas em `lib` (`zip`, `nomesDeLinhas`) mais a reescrita do `render` da ferramenta. `lib.detectarTipo`, `lib.ambiguo` e `lib.nomeSeguro` ficam como estão.

**Tech Stack:** JavaScript ES2020 vanilla dentro do `index.html` único. Zero dependências novas. `Blob` + `<a download>` para o download; o ZIP é escrito à mão pelo método *store*.

## Global Constraints

- **Um único arquivo entregável:** `index.html`. Todo CSS em `<style>`, todo JS em `<script>`. Nenhum `import`, nenhum `<script src>` externo, nenhum `type="module"`.
- **Abre com duplo clique:** tudo funciona via `file://`, sem servidor.
- **Idioma:** interface em português do Brasil.
- **Cores:** nenhuma cor hardcoded fora dos tokens `:root`.
- **Nenhum resultado falso.** A ferramenta nunca afirma um tipo sem mostrar os sinais em que se baseou.
- **Sem rolagem horizontal** em 375px; conteúdo largo rola dentro do próprio container.
- **Ordem interna do `<script>`:** `lib` → `TESTES` → `store` → `ui` → `TOOLS` → `router` → `busca` → `init`.
- **Lógica pura em `lib`, interface em `render`.** `lib` nunca toca no DOM.
- **Campos numéricos** passam por `lib.inteiro(valor, min, max, padrao)`. Esta feature não tem campo numérico — a regra fica registrada.

## Estratégia de teste

Não há npm. A suíte é o array `TESTES` renderizado na rota `#/_testes`. Helpers: `assertEq`, `assertTrue`, `assertThrows`.

**Como rodar:** ler `.claude/launch.json` (configuração `omnia-static`, python http.server na porta 8642), subir com `mcp__Claude_Browser__preview_start`, navegar até `<url>/index.html#/_testes` e ler o placar com `get_page_text`. O navegador interno **não executa JavaScript em URLs `file://`** — por isso o servidor local.

Placar atual: **91/91**. Ao fim deste plano: **95/95**.

## O gerador de ZIP já foi verificado

O spec exigia provar o gerador antes do plano. Feito: o protótipo rodou em Node, gravou um `.zip` em disco, e o `Expand-Archive` do Windows — uma implementação independente — extraiu os quatro arquivos corretamente, incluindo um `.env` de **zero byte** e um nome com acentos (`acentos-çãé.txt`). O CRC-32 bate com o vetor conhecido (`"123456789"` → `0xCBF43926`).

O código da Task 1 é exatamente o que foi verificado. Não reescreva os cabeçalhos por conta própria: os deslocamentos de bytes são fixos pelo formato.

## File Structure

| Arquivo | Responsabilidade |
|---|---|
| `index.html` | Todo o produto. Inserções em três pontos: funções na seção `lib`, casos em `TESTES`, reescrita do `render` da ferramenta `criar-arquivo`. |
| `README.md` | Descrição da ferramenta e placar de testes. |

Âncoras no `index.html`:
- `lib`: imediatamente antes de `/* ---------- pomodoro ---------- */`
- `TESTES`: imediatamente antes de `function renderTestes(el) {`
- ferramenta: o bloco `TOOLS.push({ id: "criar-arquivo", ... })`, que começa na linha 2467

---

### Task 1: `lib.zip` e `lib.crc32`

Escreve um arquivo ZIP em memória. Nenhuma mudança de interface.

**Files:**
- Modify: `index.html` — seção `lib` e seção `TESTES`

**Interfaces:**
- Consumes: nada
- Produces:
  - `lib.crc32(bytes)` → número de 32 bits sem sinal. `bytes` é `Uint8Array`.
  - `lib.zip(entradas, agora = new Date())` → `Uint8Array` com o conteúdo de um `.zip`. `entradas` é `[{ nome, conteudo }]`, `conteudo` string (vazia para arquivo vazio). `agora` é injetável para o teste ser determinístico.

- [ ] **Step 1: Escrever os testes**

Inserir antes de `function renderTestes(el) {`:

```js
TESTES.push({ nome: "crc32 bate com os vetores conhecidos", fn: () => {
  const b = s => new TextEncoder().encode(s);
  assertEq(lib.crc32(b("123456789")), 0xCBF43926, "vetor padrão do CRC-32");
  assertEq(lib.crc32(b("")), 0, "vazio é zero");
  assertEq(lib.crc32(b("a")), 0xE8B7BE43);
  assertTrue(lib.crc32(b("abc")) >= 0, "nunca negativo");
}});
TESTES.push({ nome: "zip monta um arquivo com a estrutura certa", fn: () => {
  const data = new Date(2026, 8, 22, 14, 30, 0);
  const bytes = lib.zip([
    { nome: "docker-compose.yml", conteudo: "services:\n" },
    { nome: ".env", conteudo: "" }
  ], data);
  assertTrue(bytes instanceof Uint8Array, "devolve bytes");
  const v = new DataView(bytes.buffer);
  assertEq(v.getUint32(0, true), 0x04034b50, "assinatura PK\\x03\\x04 no início");
  // registro de fim: ultimos 22 bytes
  const fim = bytes.length - 22;
  assertEq(v.getUint32(fim, true), 0x06054b50, "assinatura do registro de fim");
  assertEq(v.getUint16(fim + 8, true), 2, "duas entradas no diretório central");
  assertEq(v.getUint16(fim + 10, true), 2, "duas entradas no total");
  // o nome de cada arquivo aparece legivel nos bytes
  const texto = new TextDecoder().decode(bytes);
  assertTrue(texto.includes("docker-compose.yml"), "nome do primeiro arquivo");
  assertTrue(texto.includes(".env"), "nome do segundo arquivo");
  assertTrue(texto.includes("services:"), "conteúdo do primeiro arquivo");
}});
TESTES.push({ nome: "zip grava tamanho e crc corretos por entrada", fn: () => {
  const bytes = lib.zip([{ nome: "a.txt", conteudo: "123456789" }], new Date(2026, 0, 1, 0, 0, 0));
  const v = new DataView(bytes.buffer);
  assertEq(v.getUint32(14, true), 0xCBF43926, "crc do conteúdo no cabeçalho local");
  assertEq(v.getUint32(18, true), 9, "tamanho comprimido = original no método store");
  assertEq(v.getUint32(22, true), 9, "tamanho original");
  assertEq(v.getUint16(8, true), 0, "método 0 = store, sem compressão");
  assertEq(v.getUint16(6, true), 0x0800, "flag de nome em UTF-8 ligada");
  // arquivo vazio: crc zero e tamanho zero
  const vazio = lib.zip([{ nome: "x", conteudo: "" }], new Date(2026, 0, 1));
  const vv = new DataView(vazio.buffer);
  assertEq(vv.getUint32(14, true), 0, "crc de vazio é zero");
  assertEq(vv.getUint32(22, true), 0, "tamanho de vazio é zero");
}});
```

- [ ] **Step 2: Rodar e verificar que falha**

Abrir `#/_testes`. Esperado: 3 linhas vermelhas, a primeira dizendo `lib.crc32 is not a function`.

- [ ] **Step 3: Implementar**

Inserir antes de `/* ---------- pomodoro ---------- */`. Este código foi verificado em Node e extraído com uma ferramenta de ZIP independente — copie como está:

```js
/* ---------- zip (método store, sem compressão) ---------- */
const TABELA_CRC = (() => {
  const t = new Uint32Array(256);
  for (let n = 0; n < 256; n++) {
    let c = n;
    for (let k = 0; k < 8; k++) c = c & 1 ? 0xedb88320 ^ (c >>> 1) : c >>> 1;
    t[n] = c >>> 0;
  }
  return t;
})();

lib.crc32 = bytes => {
  let c = 0xffffffff;
  for (let i = 0; i < bytes.length; i++) c = TABELA_CRC[(c ^ bytes[i]) & 0xff] ^ (c >>> 8);
  return (c ^ 0xffffffff) >>> 0;
};

// Data e hora no formato DOS: 7 bits de ano desde 1980, e os segundos em passos de 2.
const dataDos = d => ({
  data: ((d.getFullYear() - 1980) << 9) | ((d.getMonth() + 1) << 5) | d.getDate(),
  hora: (d.getHours() << 11) | (d.getMinutes() << 5) | (d.getSeconds() >> 1)
});

// Os deslocamentos de bytes abaixo são fixados pelo formato ZIP — não mexer.
lib.zip = (entradas, agora = new Date()) => {
  const cod = new TextEncoder();
  const { data, hora } = dataDos(agora);
  const locais = [], centrais = [];
  let deslocamento = 0;

  for (const e of entradas) {
    const nome = cod.encode(e.nome);
    const dados = cod.encode(e.conteudo == null ? "" : String(e.conteudo));
    const crc = lib.crc32(dados);

    const local = new Uint8Array(30 + nome.length + dados.length);
    const vl = new DataView(local.buffer);
    vl.setUint32(0, 0x04034b50, true);    // assinatura PK\x03\x04
    vl.setUint16(4, 20, true);            // versão mínima
    vl.setUint16(6, 0x0800, true);        // flag: nome em UTF-8
    vl.setUint16(8, 0, true);             // método 0 = store
    vl.setUint16(10, hora, true);
    vl.setUint16(12, data, true);
    vl.setUint32(14, crc, true);
    vl.setUint32(18, dados.length, true); // tamanho comprimido
    vl.setUint32(22, dados.length, true); // tamanho original
    vl.setUint16(26, nome.length, true);
    vl.setUint16(28, 0, true);            // sem campo extra
    local.set(nome, 30);
    local.set(dados, 30 + nome.length);
    locais.push(local);

    const central = new Uint8Array(46 + nome.length);
    const vc = new DataView(central.buffer);
    vc.setUint32(0, 0x02014b50, true);    // assinatura PK\x01\x02
    vc.setUint16(4, 20, true);            // versão de quem criou
    vc.setUint16(6, 20, true);            // versão mínima
    vc.setUint16(8, 0x0800, true);
    vc.setUint16(10, 0, true);
    vc.setUint16(12, hora, true);
    vc.setUint16(14, data, true);
    vc.setUint32(16, crc, true);
    vc.setUint32(20, dados.length, true);
    vc.setUint32(24, dados.length, true);
    vc.setUint16(28, nome.length, true);
    vc.setUint32(42, deslocamento, true); // onde começa o cabeçalho local
    central.set(nome, 46);
    centrais.push(central);

    deslocamento += local.length;
  }

  const tamCentral = centrais.reduce((s, c) => s + c.length, 0);
  const fim = new Uint8Array(22);
  const vf = new DataView(fim.buffer);
  vf.setUint32(0, 0x06054b50, true);      // assinatura PK\x05\x06
  vf.setUint16(8, entradas.length, true);
  vf.setUint16(10, entradas.length, true);
  vf.setUint32(12, tamCentral, true);
  vf.setUint32(16, deslocamento, true);

  const saida = new Uint8Array(deslocamento + tamCentral + 22);
  let p = 0;
  for (const l of locais) { saida.set(l, p); p += l.length; }
  for (const c of centrais) { saida.set(c, p); p += c.length; }
  saida.set(fim, p);
  return saida;
};
```

- [ ] **Step 4: Rodar e verificar que passa**

Abrir `#/_testes`. Esperado: **94/94**.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: lib.zip escreve um arquivo zip pelo metodo store

Verificado em Node: o zip gerado foi extraido pelo Expand-Archive do
Windows com os arquivos corretos, inclusive um de zero byte.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: `lib.nomesDeLinhas`

Transforma o texto do campo de nomes numa lista limpa, sem duplicatas.

**Files:**
- Modify: `index.html` — seção `lib` (depois de `lib.zip`) e seção `TESTES`

**Interfaces:**
- Consumes: `lib.nomeSeguro(nome)` — já existe; limpa caracteres que o Windows recusa e nomes reservados, nunca devolve vazio
- Produces: `lib.nomesDeLinhas(texto)` → `{ nomes: string[], repetidos: string[] }`. `nomes` já passou por `nomeSeguro`, sem duplicatas, na ordem da primeira aparição. `repetidos` lista os que apareceram mais de uma vez, uma entrada por nome repetido.

- [ ] **Step 1: Escrever o teste**

```js
TESTES.push({ nome: "nomesDeLinhas limpa, ordena e aponta repetidos", fn: () => {
  const r = lib.nomesDeLinhas("docker-compose.yml\n.env\n.gitignore");
  assertEq(r.nomes, ["docker-compose.yml", ".env", ".gitignore"], "ordem preservada");
  assertEq(r.repetidos, []);

  const vazias = lib.nomesDeLinhas("\n\n  a.txt  \n\n   \nb.txt\n\n");
  assertEq(vazias.nomes, ["a.txt", "b.txt"], "linhas em branco somem e espaços são cortados");

  const rep = lib.nomesDeLinhas("a.txt\nb.txt\na.txt\nb.txt\na.txt");
  assertEq(rep.nomes, ["a.txt", "b.txt"], "duplicata entra uma vez só");
  assertEq(rep.repetidos, ["a.txt", "b.txt"], "cada repetido aparece uma vez na lista de avisos");

  const sujo = lib.nomesDeLinhas("pasta/arquivo.txt\nCON.tar.gz");
  assertEq(sujo.nomes, ["pasta-arquivo.txt", "_CON.tar.gz"], "passa pelo nomeSeguro");

  assertEq(lib.nomesDeLinhas("").nomes, []);
  assertEq(lib.nomesDeLinhas("   \n  \n").nomes, [], "só espaços não vira nome");
  assertEq(lib.nomesDeLinhas(null).nomes, [], "null não derruba");
  assertEq(lib.nomesDeLinhas(undefined).nomes, []);
}});
```

- [ ] **Step 2: Rodar e verificar que falha**

Abrir `#/_testes`. Esperado: `lib.nomesDeLinhas is not a function`.

- [ ] **Step 3: Implementar**

```js
// Uma linha, um nome. Linha em branco é ruído de quem colou e some sem aviso;
// nome repetido provavelmente é engano, então volta em "repetidos" para a interface avisar.
lib.nomesDeLinhas = texto => {
  const linhas = String(texto == null ? "" : texto).split("\n");
  const nomes = [], vistos = new Set(), repetidos = [], jaAvisados = new Set();
  for (const linha of linhas) {
    const cru = linha.trim();
    if (!cru) continue;
    const limpo = lib.nomeSeguro(cru);
    if (vistos.has(limpo)) {
      if (!jaAvisados.has(limpo)) { repetidos.push(limpo); jaAvisados.add(limpo); }
      continue;
    }
    vistos.add(limpo);
    nomes.push(limpo);
  }
  return { nomes, repetidos };
};
```

- [ ] **Step 4: Rodar e verificar que passa**

Abrir `#/_testes`. Esperado: **95/95**.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: lib.nomesDeLinhas le a lista de nomes do campo

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 3: A ferramenta com dois campos

Reescreve o `render` de `criar-arquivo`: campo de nomes, campo de conteúdo condicional, e o download que escolhe entre arquivo solto e zip.

**Files:**
- Modify: `index.html` — o bloco `TOOLS.push({ id: "criar-arquivo", ... })` (começa na linha 2467) e o `<style>`
- Modify: `README.md`

**Interfaces:**
- Consumes: `lib.zip(entradas, agora)`, `lib.nomesDeLinhas(texto)`, `lib.detectarTipo(texto)`, `lib.ambiguo(candidatos)`, `lib.nomeSeguro(nome)`, e o kit `ui` (`campo`, `area`, `btn`, `linha`, `montar`, `nota`, `toast`)
- Produces: nada que outra task consuma

- [ ] **Step 1: Acrescentar o CSS**

Inserir no `<style>`, logo depois da regra `.arq-alt:hover`:

```css
.arq-aviso { margin-top: 8px; font-size: 13px; color: var(--acento); }
.campo textarea.arq-nomes { min-height: 90px; }
.campo textarea:disabled { opacity: .45; cursor: default; }
```

- [ ] **Step 2: Substituir o `render` inteiro**

Trocar todo o corpo de `render(el) { ... }` da ferramenta `criar-arquivo` por:

```js
  render(el) {
    const nomes = ui.area("Nomes dos arquivos", { placeholder: "Um por linha:\ndocker-compose.yml\n.env\n.gitignore" });
    nomes.classList.add("arq-nomes");
    const conteudo = ui.area("Conteúdo", { placeholder: "Opcional. Cole aqui o conteúdo do arquivo." });
    const palpite = document.createElement("div"); palpite.className = "arq-palpite"; palpite.hidden = true;
    const aviso = document.createElement("div"); aviso.className = "arq-aviso"; aviso.hidden = true;
    const btnBaixar = ui.btn("Baixar", () => baixar());
    let candidatos = [], escolhido = null, nomeEditado = false;

    nomes.addEventListener("input", () => { nomeEditado = nomes.value.trim().length > 0; pintar(); });

    const lidos = () => lib.nomesDeLinhas(nomes.value);

    const pintar = () => {
      const { nomes: lista, repetidos } = lidos();
      const varios = lista.length > 1;

      // Conteúdo só faz sentido com um arquivo: com vários, não há para qual deles ele iria.
      conteudo.disabled = varios;
      conteudo.wrapper.querySelector("label").textContent =
        varios ? "Conteúdo (indisponível — vários arquivos saem vazios)" : "Conteúdo";

      const partes = [];
      if (repetidos.length) partes.push("Nome repetido, vai entrar uma vez só: " + repetidos.join(", "));
      if (lista.length === 1 && !conteudo.value) partes.push("Sem conteúdo — vai baixar um arquivo vazio.");
      aviso.textContent = partes.join(" ");
      aviso.hidden = !partes.length;

      btnBaixar.textContent = varios
        ? "Baixar .zip com " + lista.length + " arquivos"
        : (lista.length === 1 ? "Baixar " + lista[0] : "Baixar");
      btnBaixar.disabled = lista.length === 0;

      // O painel de reconhecimento só aparece quando há conteúdo para reconhecer.
      if (!escolhido || varios) { palpite.hidden = true; return; }
      const alternativas = candidatos.filter(c => c.id !== escolhido.id).slice(0, 3);
      const rotuloEmpate = alternativas.length && (lib.ambiguo(candidatos) || candidatos[0].id !== escolhido.id);
      palpite.innerHTML =
        '<div class="arq-titulo">Parece um <strong>' + escolhido.nome + '</strong></div>' +
        '<div class="arq-sinais">Bateu: ' + escolhido.sinais.join(" · ") + '</div>' +
        (alternativas.length ? '<div class="arq-alternativas"><span>' +
          (rotuloEmpate ? "Ou talvez" : "Outras opções") + '</span></div>' : '');
      const caixa = palpite.querySelector(".arq-alternativas");
      if (caixa) alternativas.forEach(c => {
        const b = document.createElement("button");
        b.type = "button"; b.className = "arq-alt"; b.textContent = c.nome;
        // Clicar numa alternativa é escolha deliberada: escreve o nome direto.
        b.onclick = () => { escolhido = c; nomes.value = c.nome; nomeEditado = true; pintar(); };
        caixa.appendChild(b);
      });
      palpite.hidden = false;
    };

    let timer = null;
    conteudo.addEventListener("input", () => {
      clearTimeout(timer);
      timer = setTimeout(() => {
        if (!conteudo.value.trim()) { candidatos = []; escolhido = null; pintar(); return; }
        candidatos = lib.detectarTipo(conteudo.value);
        escolhido = candidatos[0];
        // O reconhecimento preenche o campo de nomes, mas nunca por cima do que o usuário digitou.
        if (!nomeEditado) nomes.value = escolhido.nome;
        pintar();
      }, 250);
    });

    const entregar = (bytes, mime, nomeArquivo) => {
      const blob = new Blob([bytes], { type: mime });
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url; a.download = nomeArquivo;
      document.body.appendChild(a); a.click(); a.remove();
      setTimeout(() => URL.revokeObjectURL(url), 1000);
    };

    const baixar = () => {
      const lista = lidos().nomes;
      if (!lista.length) return;
      try {
        if (lista.length === 1) {
          entregar(conteudo.value, (escolhido && escolhido.mime) || "text/plain", lista[0]);
          ui.toast("Baixando " + lista[0]);
        } else {
          const bytes = lib.zip(lista.map(n => ({ nome: n, conteudo: "" })));
          entregar(bytes, "application/zip", "arquivos.zip");
          ui.toast("Baixando arquivos.zip com " + lista.length + " arquivos");
        }
      } catch (e) {
        ui.toast("Não foi possível baixar — copie o conteúdo manualmente");
      }
    };

    ui.montar(el, nomes, conteudo, palpite, aviso, ui.linha(btnBaixar),
      ui.nota("Um nome baixa o arquivo direto; dois ou mais vêm num .zip, vazios. O reconhecimento preenche o campo de nomes quando ele está vazio — o nome final é sempre seu. O conteúdo baixa exatamente como está, exceto quebras de linha: o navegador sempre grava só LF, mesmo que o original tivesse CRLF."));
    pintar();
  }
```

Também trocar a `desc` da ferramenta, na linha logo acima do `tags`:

```js
  nome: "Criar Arquivo", desc: "Digite os nomes e baixe os arquivos — um direto, vários num zip",
```

E acrescentar `"zip"`, `"varios arquivos"` e `"arquivo vazio"` ao array `tags`.

- [ ] **Step 3: Rodar a suíte**

Abrir `#/_testes`. Esperado: **95/95** — esta task não acrescenta testes, e não pode derrubar nenhum. `read_console_messages` com `onlyErrors: true`: nenhum.

- [ ] **Step 4: Verificar na tela, citando o que apareceu**

Em `#/criar-arquivo`:

- Digitar `docker-compose.yml` no campo de nomes → o botão diz **"Baixar docker-compose.yml"**, e o aviso diz "Sem conteúdo — vai baixar um arquivo vazio."
- Colar `services:\n  web:\n    image: nginx\n` no conteúdo → o aviso some e o painel diz "Parece um **docker-compose.yml**".
- Apagar o campo de nomes e colar o mesmo conteúdo → o campo de nomes se preenche sozinho com `docker-compose.yml`.
- Digitar `meu.yml` à mão e depois mudar o conteúdo → o nome digitado **não** é sobrescrito.
- Digitar três nomes (`docker-compose.yml`, `.env`, `.gitignore`) → o botão vira **"Baixar .zip com 3 arquivos"**, e o campo de conteúdo fica desabilitado com o rótulo explicando o porquê.
- Digitar `a.txt` duas vezes → o aviso diz "Nome repetido, vai entrar uma vez só: a.txt", e o botão diz "Baixar a.txt".
- Esvaziar o campo de nomes → botão desabilitado.
- Baixar o zip de três nomes, descompactar, e confirmar os três arquivos vazios com os nomes certos.

- [ ] **Step 5: Verificar em 375px**

`resize_window` para mobile. Confirmar `document.documentElement.scrollWidth === clientWidth`. Voltar para desktop.

- [ ] **Step 6: Atualizar o README**

- Placar: `**91/91**` → `**95/95**`.
- Na tabela de ferramentas, a linha **Desenvolvedor**: trocar `Criar arquivo` por `Criar arquivos (um ou vários, em zip)`.

- [ ] **Step 7: Commit**

```bash
git add index.html README.md
git commit -m "feat: campo de nomes, varios arquivos em zip e arquivo vazio

Um nome baixa direto; dois ou mais vao num zip. O reconhecimento passa a
preencher o campo de nomes em vez de ser o unico caminho para ele.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Auto-revisão do plano

**Cobertura do spec:**

| Requisito do spec | Task |
|---|---|
| `lib.zip(entradas)` pelo método store | Task 1 |
| CRC-32 e data/hora DOS | Task 1 |
| Flag de UTF-8 no nome | Task 1 (testado: `getUint16(6) === 0x0800`) |
| `lib.nomesDeLinhas` com `nomes` e `repetidos` | Task 2 |
| Linha em branco ignorada sem aviso | Task 2 (testado) |
| Nome repetido avisa e entra uma vez | Task 2 (testado) + Task 3 (aviso na tela) |
| Nomes passam por `nomeSeguro` | Task 2 (testado com `pasta/arquivo.txt` e `CON.tar.gz`) |
| Campo de nomes, um por linha | Task 3 |
| Conteúdo só com um nome; desabilitado com vários | Task 3 |
| Um arquivo baixa direto; dois ou mais em zip | Task 3 |
| Arquivo vazio com um nome | Task 3 (aviso na tela + `baixar` sem guarda de conteúdo) |
| Reconhecimento preenche o campo de nomes | Task 3 |
| Reconhecimento nunca sobrescreve nome digitado | Task 3 (`nomeEditado`) |
| Botão diz o que vai fazer | Task 3 (três estados) |
| Falha ao montar o zip → toast | Task 3 (`try/catch`) |
| Critérios de sucesso 1–8 | Tasks 1–3, verificação no Step 4 |

Sem lacunas.

**Consistência de nomes entre tasks:** `lib.crc32` e `lib.zip` (Task 1) são consumidos por `lib.zip` e pela Task 3. `lib.nomesDeLinhas` (Task 2) devolve `{ nomes, repetidos }`, e a Task 3 desestrutura exatamente esses dois campos. `lib.nomeSeguro` já existe e é consumido pela Task 2. `ui.area(label, attrs)` e `ui.campo(label, tipo, attrs)` são as assinaturas reais do kit; a Task 3 usa `ui.area` com dois argumentos, que confere.

**Contagem de testes:** 91 atuais + 3 (T1) + 1 (T2) = **95**. Bate com o esperado na Task 2, Step 4 e no README.

**Uma decisão que mudou em relação ao código atual, e por quê:** hoje `nomeEditado` vira `true` na primeira tecla e nunca mais volta. Com o campo de nomes sendo o principal, isso prenderia o usuário: apagar tudo do campo deveria devolver o preenchimento automático. Por isso a Task 3 passa a calcular `nomeEditado` a cada digitada — ele é `true` enquanto houver algo escrito, e `false` quando o campo está vazio. O clique numa alternativa continua escrevendo o nome e marcando como editado.

**Um risco que o plano aceita conscientemente:** `lib.zip` monta o arquivo inteiro em memória. Para os nomes vazios deste caso de uso isso é irrelevante — três arquivos de zero byte dão menos de 500 bytes. Se um dia o zip passar a carregar conteúdo, vale revisitar.
