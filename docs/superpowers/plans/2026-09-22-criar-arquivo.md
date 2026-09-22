# Criar Arquivo — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Adicionar a 28ª ferramenta ao OmnIA: cola-se um conteúdo, ela reconhece o tipo de arquivo por pontuação de sinais, sugere nome e extensão, e baixa.

**Architecture:** Duas funções puras novas em `lib` (`detectarTipo`, `nomeSeguro`) alimentadas por uma tabela de tipos `TIPOS`, mais um `TOOLS.push` com a interface. Nenhuma outra parte do site muda.

**Tech Stack:** JavaScript ES2020 vanilla dentro do `index.html` único. Zero dependências novas. `Blob` + `<a download>` para o download.

## Global Constraints

- **Um único arquivo entregável:** `index.html`. Todo CSS em `<style>`, todo JS em `<script>`. Nenhum `import`, nenhum `<script src>` externo, nenhum `type="module"`.
- **Abre com duplo clique:** tudo funciona via `file://`, sem servidor.
- **Idioma:** interface em português do Brasil.
- **Cores:** nenhuma cor hardcoded fora dos tokens `:root`.
- **Selos:** `tipo` é exatamente `'nativa'`, `'online'` ou `'script'`.
- **Nenhum resultado falso.** A ferramenta nunca afirma um tipo sem mostrar os sinais em que se baseou.
- **Sem rolagem horizontal** em 375px; conteúdo largo rola dentro do próprio container.
- **Ordem interna do `<script>`:** `lib` → `TESTES` → `store` → `ui` → `TOOLS` → `router` → `busca` → `init`.
- **Lógica pura em `lib`, interface em `render`.** `lib` nunca toca no DOM.
- **Campos numéricos** passam por `lib.inteiro(valor, min, max, padrao)` (index.html:402). Esta ferramenta não tem campo numérico — a regra fica registrada para o caso de surgir um.

## Estratégia de teste

Não há npm. A suíte é o array `TESTES` renderizado na rota `#/_testes`. Helpers disponíveis: `assertEq` (index.html:865), `assertTrue` (index.html:870), `assertThrows` (index.html:874).

**Como rodar:** ler `.claude/launch.json` para o nome da configuração, subir com `mcp__Claude_Browser__preview_start`, navegar até `<url>/index.html#/_testes` e ler o placar com `get_page_text`. O navegador interno **não executa JavaScript em URLs `file://`** fora da raiz do projeto — por isso o servidor local.

Placar atual: **82/82**. Ao fim deste plano: **88/88**.

## File Structure

| Arquivo | Responsabilidade |
|---|---|
| `index.html` | Todo o produto. Inserções em três pontos: tabela + funções na seção `lib`, casos em `TESTES`, a ferramenta em `TOOLS`. |
| `README.md` | Contagem de ferramentas, linha da categoria Desenvolvedor, placar de testes. |
| `docs/superpowers/specs/2026-09-22-criar-arquivo-design.md` | Spec (já existe, não modificar). |

Âncoras de inserção no `index.html`:
- `lib`: imediatamente antes de `/* ---------- pomodoro ---------- */`
- `TESTES`: imediatamente antes de `function renderTestes(el) {`
- `TOOLS`: imediatamente antes de `TOOLS.push({\n  id: "pomodoro",`

---

### Task 1: A tabela de tipos e o teste de integridade

Entrega a tabela `TIPOS` com os 18 tipos reconhecíveis, validada por um teste de integridade. Sem detecção ainda — só os dados.

**Files:**
- Modify: `index.html` — seção `lib` e seção `TESTES`

**Interfaces:**
- Consumes: `assertTrue`, `assertEq` (helpers da suíte)
- Produces:
  - `TIPOS` — array de `{ id, nome, ext, mime, sinais: [{re, peso, label}] }`, 18 entradas
  - `TIPO_TXT` — a entrada de fallback, `{ id: "txt", nome: "arquivo.txt", ext: ".txt", mime: "text/plain" }`

- [ ] **Step 1: Escrever o teste de integridade**

Inserir antes de `function renderTestes(el) {`:

```js
TESTES.push({ nome: "tabela TIPOS está íntegra", fn: () => {
  const vistos = new Set();
  for (const t of TIPOS) {
    assertTrue(!vistos.has(t.id), "id repetido: " + t.id); vistos.add(t.id);
    assertTrue(typeof t.nome === "string" && t.nome.length > 0, t.id + " precisa de nome");
    assertTrue(/^\.[a-z0-9]+$/.test(t.ext) || t.ext === "", t.id + " extensão inválida: " + t.ext);
    assertTrue(typeof t.mime === "string" && t.mime.includes("/"), t.id + " precisa de mime");
    assertTrue(Array.isArray(t.sinais) && t.sinais.length > 0, t.id + " precisa de pelo menos um sinal");
    for (const s of t.sinais) {
      assertTrue(s.re instanceof RegExp, t.id + ": sinal sem regex");
      assertTrue(s.peso > 0, t.id + ": peso tem que ser positivo");
      assertTrue(typeof s.label === "string" && s.label.length > 0, t.id + ": sinal sem label legível");
    }
  }
  assertEq(TIPOS.length, 18, "os 18 tipos do spec");
  assertEq(TIPO_TXT.ext, ".txt");
  assertTrue(!vistos.has(TIPO_TXT.id), "txt é fallback, não entra em TIPOS");
}});
```

- [ ] **Step 2: Rodar e verificar que falha**

Abrir `#/_testes`. Esperado: uma linha vermelha `tabela TIPOS está íntegra — TIPOS is not defined`.

- [ ] **Step 3: Escrever a tabela**

Inserir antes de `/* ---------- pomodoro ---------- */`:

```js
/* ---------- criar arquivo: reconhecer tipo pelo conteúdo ---------- */
// Cada tipo soma os pesos dos sinais que batem. Maior soma relativa vence.
// Sinais específicos (services:, "dependencies") pesam mais que genéricos (key:).
const TIPOS = [
  { id: "docker-compose", nome: "docker-compose.yml", ext: ".yml", mime: "text/yaml", sinais: [
    { re: /^services:\s*$/m, peso: 6, label: "services: no início de linha" },
    { re: /^\s+image:\s*\S/m, peso: 4, label: "image:" },
    { re: /^\s+ports:\s*$/m, peso: 2, label: "ports:" },
    { re: /^\s+volumes:\s*$/m, peso: 2, label: "volumes:" },
    { re: /^\s+environment:\s*$/m, peso: 2, label: "environment:" }
  ]},
  { id: "dockerfile", nome: "Dockerfile", ext: "", mime: "text/plain", sinais: [
    { re: /^FROM\s+\S+/m, peso: 6, label: "FROM" },
    { re: /^RUN\s+\S/m, peso: 3, label: "RUN" },
    { re: /^(CMD|ENTRYPOINT)\s/m, peso: 3, label: "CMD ou ENTRYPOINT" },
    { re: /^(COPY|ADD)\s/m, peso: 2, label: "COPY ou ADD" },
    { re: /^WORKDIR\s/m, peso: 2, label: "WORKDIR" }
  ]},
  { id: "package-json", nome: "package.json", ext: ".json", mime: "application/json", sinais: [
    { re: /"(dependencies|devDependencies)"\s*:/, peso: 6, label: '"dependencies"' },
    { re: /"scripts"\s*:/, peso: 4, label: '"scripts"' },
    { re: /"name"\s*:/, peso: 2, label: '"name"' },
    { re: /"version"\s*:/, peso: 2, label: '"version"' }
  ]},
  { id: "tsconfig", nome: "tsconfig.json", ext: ".json", mime: "application/json", sinais: [
    { re: /"compilerOptions"\s*:/, peso: 8, label: '"compilerOptions"' },
    { re: /"(target|module|strict|outDir)"\s*:/, peso: 3, label: "opção de compilador" }
  ]},
  { id: "env", nome: ".env", ext: "", mime: "text/plain", sinais: [
    { re: /^[A-Z][A-Z0-9_]*=/m, peso: 5, label: "VARIAVEL=valor em maiúsculas" },
    { re: /^(DB_|DATABASE_|API_|APP_|SECRET|TOKEN|PORT|NODE_ENV)/m, peso: 4, label: "nome típico de variável de ambiente" }
  ]},
  { id: "sql", nome: "consulta.sql", ext: ".sql", mime: "application/sql", sinais: [
    { re: /\b(CREATE\s+(TABLE|INDEX|VIEW)|ALTER\s+TABLE)\b/i, peso: 6, label: "CREATE ou ALTER TABLE" },
    { re: /\bSELECT\b[\s\S]*\bFROM\b/i, peso: 5, label: "SELECT ... FROM" },
    { re: /\b(INSERT\s+INTO|UPDATE\s+\w+\s+SET|DELETE\s+FROM)\b/i, peso: 5, label: "INSERT, UPDATE ou DELETE" },
    { re: /\b(WHERE|JOIN|GROUP\s+BY|ORDER\s+BY)\b/i, peso: 2, label: "cláusula SQL" }
  ]},
  { id: "json", nome: "dados.json", ext: ".json", mime: "application/json", sinais: [
    { re: /^\s*[{[][\s\S]*[}\]]\s*$/, peso: 5, label: "abre e fecha com chave ou colchete" },
    { re: /"[^"]+"\s*:\s*("|\d|true|false|null|\{|\[)/, peso: 4, label: '"chave": valor' }
  ]},
  { id: "yaml", nome: "config.yaml", ext: ".yaml", mime: "text/yaml", sinais: [
    { re: /^[a-zA-Z_][\w-]*:\s*$/m, peso: 3, label: "chave: em linha própria" },
    { re: /^\s*-\s+\S/m, peso: 3, label: "- item de lista" },
    { re: /^[a-zA-Z_][\w-]*:\s+\S/m, peso: 2, label: "chave: valor" },
    { re: /^---\s*$/m, peso: 2, label: "--- separador de documento" }
  ]},
  { id: "xml", nome: "documento.xml", ext: ".xml", mime: "application/xml", sinais: [
    { re: /^\s*<\?xml\s/, peso: 7, label: "declaração <?xml" },
    { re: /<([a-zA-Z][\w:-]*)[^>]*>[\s\S]*<\/\1>/, peso: 4, label: "tag que abre e fecha" }
  ]},
  { id: "html", nome: "pagina.html", ext: ".html", mime: "text/html", sinais: [
    { re: /<!doctype\s+html/i, peso: 7, label: "<!doctype html>" },
    { re: /<html[\s>]/i, peso: 5, label: "<html>" },
    { re: /<(body|head|div|p|span|a|script|style)[\s>]/i, peso: 3, label: "tag de HTML" }
  ]},
  { id: "css", nome: "estilo.css", ext: ".css", mime: "text/css", sinais: [
    { re: /[.#]?[\w-]+\s*\{[^}]*:[^}]*;[^}]*\}/, peso: 5, label: "seletor { propriedade: valor; }" },
    { re: /^\s*(@media|@import|@font-face|:root)/m, peso: 4, label: "regra @ ou :root" },
    { re: /(color|margin|padding|display|font-size)\s*:/, peso: 3, label: "propriedade CSS" }
  ]},
  { id: "js", nome: "script.js", ext: ".js", mime: "text/javascript", sinais: [
    { re: /\b(const|let)\s+\w+\s*=/, peso: 4, label: "const ou let" },
    { re: /=>\s*[{(\w]/, peso: 3, label: "arrow function" },
    { re: /\b(function\s+\w+\s*\(|require\s*\(|module\.exports)/, peso: 4, label: "function, require ou module.exports" },
    { re: /^\s*(import|export)\s/m, peso: 3, label: "import ou export" }
  ]},
  { id: "py", nome: "script.py", ext: ".py", mime: "text/x-python", sinais: [
    { re: /^\s*def\s+\w+\s*\(.*\)\s*:/m, peso: 6, label: "def função():" },
    { re: /^if\s+__name__\s*==/m, peso: 5, label: 'if __name__ ==' },
    { re: /^\s*(from\s+[\w.]+\s+)?import\s+\w/m, peso: 3, label: "import" },
    { re: /^\s*(class\s+\w+|print\s*\()/m, peso: 2, label: "class ou print(" }
  ]},
  { id: "sh", nome: "script.sh", ext: ".sh", mime: "application/x-sh", sinais: [
    { re: /^#!\s*\/(usr\/)?bin\/(ba|z|k)?sh/, peso: 8, label: "shebang #!/bin/bash" },
    { re: /^\s*(echo|cd|mkdir|rm|cp|mv|export|chmod)\s/m, peso: 2, label: "comando de shell" },
    { re: /\$\{?\w+\}?/, peso: 1, label: "$variável" }
  ]},
  { id: "md", nome: "documento.md", ext: ".md", mime: "text/markdown", sinais: [
    { re: /^#{1,6}\s+\S/m, peso: 5, label: "# título" },
    { re: /^```/m, peso: 4, label: "``` bloco de código" },
    { re: /\*\*[^*\n]+\*\*/, peso: 2, label: "**negrito**" },
    { re: /^\s*[-*]\s+\S/m, peso: 2, label: "- item de lista" },
    { re: /\[[^\]\n]+\]\([^)\n]+\)/, peso: 3, label: "[link](url)" }
  ]},
  { id: "csv", nome: "dados.csv", ext: ".csv", mime: "text/csv", sinais: [
    { re: /^[^,\n]+(,[^,\n]*){2,}\r?\n[^,\n]+(,[^,\n]*){2,}/, peso: 6, label: "linhas com o mesmo número de vírgulas" },
    { re: /^[^;\n]+(;[^;\n]*){2,}\r?\n[^;\n]+(;[^;\n]*){2,}/, peso: 6, label: "linhas separadas por ponto e vírgula" }
  ]},
  { id: "ini", nome: "config.ini", ext: ".ini", mime: "text/plain", sinais: [
    { re: /^\[[\w.\s-]+\]\s*$/m, peso: 6, label: "[seção]" },
    { re: /^\s*[\w.-]+\s*=\s*\S/m, peso: 3, label: "chave = valor" },
    { re: /^\s*[;#]\s*\S/m, peso: 1, label: "comentário com ; ou #" }
  ]},
  { id: "gitignore", nome: ".gitignore", ext: "", mime: "text/plain", sinais: [
    { re: /^(node_modules|__pycache__|\.venv|dist|build|target)\/?\s*$/m, peso: 6, label: "pasta ignorada conhecida" },
    { re: /^\*\.\w+\s*$/m, peso: 4, label: "*.extensão" },
    { re: /^!\S/m, peso: 3, label: "! exceção" }
  ]}
];

// Piso: quando nada bate, ainda existe uma saída honesta.
const TIPO_TXT = { id: "txt", nome: "arquivo.txt", ext: ".txt", mime: "text/plain", sinais: [] };
```

- [ ] **Step 4: Rodar e verificar que passa**

Abrir `#/_testes`. Esperado: **83/83**, "Todos passaram."

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: tabela de tipos reconheciveis para a ferramenta Criar Arquivo

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: `lib.detectarTipo`

O motor de reconhecimento: soma os pesos, normaliza, ordena, garante o fallback.

**Files:**
- Modify: `index.html` — seção `lib` (depois de `TIPO_TXT`) e seção `TESTES`

**Interfaces:**
- Consumes: `TIPOS`, `TIPO_TXT` (Task 1)
- Produces:
  - `lib.detectarTipo(texto)` → array de `{ id, nome, ext, mime, pontos, confianca, sinais }` ordenado por `pontos` decrescente. Nunca vazio. `pontos` é a soma absoluta dos pesos que bateram e é o critério de ordenação; `confianca` (0 a 1) é a mesma soma normalizada pelo total do tipo, e serve só para exibir. `sinais` é array de strings (os `label` que bateram).
  - `lib.ambiguo(candidatos)` → `boolean` — verdadeiro quando o segundo colocado tem pelo menos 70% dos **pontos** do primeiro.

- [ ] **Step 1: Escrever os testes**

Inserir antes de `function renderTestes(el) {`:

```js
TESTES.push({ nome: "detectarTipo acerta cada um dos 18 tipos", fn: () => {
  const amostras = {
    "docker-compose": "services:\n  web:\n    image: nginx:alpine\n    ports:\n      - '80:80'\n    environment:\n      - TZ=America/Sao_Paulo\n",
    "dockerfile": "FROM node:20-alpine\nWORKDIR /app\nCOPY package.json .\nRUN npm install\nCMD [\"npm\", \"start\"]\n",
    "package-json": '{\n  "name": "meu-app",\n  "version": "1.0.0",\n  "scripts": { "dev": "vite" },\n  "dependencies": { "vue": "^3.4.0" }\n}\n',
    "tsconfig": '{\n  "compilerOptions": {\n    "target": "ES2020",\n    "strict": true,\n    "outDir": "./dist"\n  }\n}\n',
    "env": "NODE_ENV=production\nDB_HOST=localhost\nDB_PORT=5432\nAPI_KEY=abc123\n",
    "sql": "CREATE TABLE clientes (\n  id SERIAL PRIMARY KEY,\n  nome TEXT NOT NULL\n);\nSELECT nome FROM clientes WHERE id = 1;\n",
    "json": '{\n  "cidade": "Curitiba",\n  "populacao": 1963726,\n  "capital": true\n}\n',
    "yaml": "nome: relatorio\nversao: 2\nsaidas:\n  - csv\n  - pdf\nativo: true\n",
    "xml": '<?xml version="1.0" encoding="UTF-8"?>\n<nota>\n  <valor>150.00</valor>\n</nota>\n',
    "html": '<!doctype html>\n<html lang="pt-BR">\n<body>\n  <div>Olá</div>\n</body>\n</html>\n',
    "css": ":root { --cor: #333; }\n.cartao {\n  display: flex;\n  padding: 12px;\n  color: var(--cor);\n}\n",
    "js": "const soma = (a, b) => a + b;\nfunction dobro(n) { return n * 2; }\nmodule.exports = { soma, dobro };\n",
    "py": "import sys\n\ndef principal(nome):\n    print(f'Olá {nome}')\n\nif __name__ == '__main__':\n    principal(sys.argv[1])\n",
    "sh": "#!/bin/bash\nset -e\ncd /opt/app\nexport NODE_ENV=production\necho \"Publicando ${VERSAO}\"\n",
    "md": "# Relatório\n\nTexto com **negrito** e um [link](https://exemplo.com).\n\n- primeiro\n- segundo\n\n```js\nconsole.log(1);\n```\n",
    "csv": "nome,idade,cidade\nAna,34,Curitiba\nBruno,28,Recife\nCarla,41,Salvador\n",
    "ini": "[banco]\nhost = localhost\nporta = 5432\n\n[app]\ndebug = false\n",
    "gitignore": "node_modules/\ndist/\n*.log\n*.tmp\n!importante.log\n"
  };
  for (const [id, amostra] of Object.entries(amostras)) {
    const top = lib.detectarTipo(amostra)[0];
    assertEq(top.id, id, "amostra de " + id + " foi lida como " + top.id);
    assertTrue(top.sinais.length > 0, id + " precisa listar os sinais que bateram");
    assertTrue(top.confianca > 0 && top.confianca <= 1, id + " confiança fora de 0–1: " + top.confianca);
  }
}});
TESTES.push({ nome: "detectarTipo separa o específico do genérico", fn: () => {
  const compose = "services:\n  web:\n    image: nginx\n    ports:\n      - '80:80'\n";
  const cands = lib.detectarTipo(compose);
  assertEq(cands[0].id, "docker-compose");
  assertTrue(cands.some(c => c.id === "yaml"), "yaml genérico deve aparecer como alternativa");
  assertTrue(cands.findIndex(c => c.id === "yaml") > 0, "yaml não pode ganhar do docker-compose");
  // Este é o caso que derrubou a normalização: por confiança o yaml ganharia (0,80 x 0,75).
  const yaml = cands.find(c => c.id === "yaml");
  assertTrue(cands[0].pontos > yaml.pontos, "ordenação tem que ser por pontos absolutos");

  const pkg = '{ "name": "x", "version": "1.0.0", "scripts": {}, "dependencies": {} }';
  const cp = lib.detectarTipo(pkg);
  assertEq(cp[0].id, "package-json");
  assertTrue(cp.some(c => c.id === "json"), "json genérico deve aparecer como alternativa");
}});
TESTES.push({ nome: "detectarTipo cai no txt quando não reconhece nada", fn: () => {
  const prosa = "Reunião de terça ficou para quinta. Avisar o time e remarcar a sala.";
  const c1 = lib.detectarTipo(prosa);
  assertEq(c1[0].id, "txt", "prosa em português não pode virar código");
  assertEq(c1.length, 1);
  assertEq(lib.detectarTipo("")[0].id, "txt", "vazio");
  assertEq(lib.detectarTipo("   \n  \n")[0].id, "txt", "só espaços");
  assertEq(lib.detectarTipo(null)[0].id, "txt", "null não derruba");
  assertEq(lib.detectarTipo(undefined)[0].id, "txt", "undefined não derruba");
}});
TESTES.push({ nome: "ambiguo marca empate e ignora diferença grande", fn: () => {
  assertEq(lib.ambiguo([{ pontos: 10 }, { pontos: 8 }]), true, "8 de 10 é empate");
  assertEq(lib.ambiguo([{ pontos: 10 }, { pontos: 7 }]), true, "7 é o limite, conta");
  assertEq(lib.ambiguo([{ pontos: 10 }, { pontos: 5 }]), false, "5 é diferença clara");
  assertEq(lib.ambiguo([{ pontos: 10 }]), false, "sozinho nunca é ambíguo");
  assertEq(lib.ambiguo([]), false, "lista vazia não quebra");
}});
```

- [ ] **Step 2: Rodar e verificar que falha**

Abrir `#/_testes`. Esperado: 4 linhas vermelhas, a primeira delas dizendo `lib.detectarTipo is not a function`.

- [ ] **Step 3: Implementar**

Inserir logo após `const TIPO_TXT = ...`:

```js
// Soma os pesos dos sinais que batem, normaliza pelo total possível daquele tipo.
// Devolve os candidatos do mais para o menos provável. Nunca devolve lista vazia.
lib.detectarTipo = texto => {
  const t = texto == null ? "" : String(texto);
  if (!t.trim()) return [{ ...TIPO_TXT, pontos: 0, confianca: 0.1, sinais: ["conteúdo vazio"] }];

  const candidatos = [];
  for (const tipo of TIPOS) {
    const total = tipo.sinais.reduce((soma, s) => soma + s.peso, 0);
    let pontos = 0;
    const bateram = [];
    for (const s of tipo.sinais) {
      if (s.re.test(t)) { pontos += s.peso; bateram.push(s.label); }
    }
    if (pontos > 0) {
      candidatos.push({
        id: tipo.id, nome: tipo.nome, ext: tipo.ext, mime: tipo.mime,
        pontos, confianca: Math.round((pontos / total) * 100) / 100, sinais: bateram
      });
    }
  }
  if (!candidatos.length) return [{ ...TIPO_TXT, pontos: 0, confianca: 0.1, sinais: ["nenhum padrão de código reconhecido"] }];
  candidatos.sort((a, b) => b.pontos - a.pontos || b.confianca - a.confianca);
  return candidatos;
};

// Empate: o segundo colocado tem pelo menos 70% dos pontos do primeiro.
lib.ambiguo = candidatos =>
  Array.isArray(candidatos) && candidatos.length > 1 &&
  candidatos[1].pontos >= candidatos[0].pontos * 0.7;
```

**Por que ordenar por pontos absolutos, e não pela confiança normalizada.** Esta decisão foi verificada rodando a tabela contra as amostras antes de escrever o plano: com a normalizada, o `yaml` genérico vencia o `docker-compose` (0,80 contra 0,75), quebrando o critério nº 1 do spec. A causa é que normalizar premia quem tem poucos sinais fáceis — o yaml bateu 3 de 4 sinais, o docker-compose 3 de 5. Por pontos absolutos, o docker-compose faz 12 e o yaml 8, e a ordem fica certa. A normalizada continua existindo só para mostrar a confiança ao usuário.

Nota sobre `sort`: regex com flag `g` guarda estado entre chamadas de `.test()` — nenhum sinal da tabela usa `g`, e isso é proposital. Não adicionar `g` a nenhum sinal.

- [ ] **Step 4: Rodar e verificar que passa**

Abrir `#/_testes`. Esperado: **87/87**.

Se a amostra de algum tipo cair no lugar errado, o defeito está nos pesos da Task 1, não no teste — ajustar os pesos e registrar no relatório qual amostra obrigou o ajuste.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: lib.detectarTipo reconhece o tipo do conteudo por pontuacao de sinais

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 3: `lib.nomeSeguro`

Limpa o nome do arquivo para não gerar download inválido no Windows.

**Files:**
- Modify: `index.html` — seção `lib` e seção `TESTES`

**Interfaces:**
- Consumes: nada
- Produces: `lib.nomeSeguro(nome)` → string. Nunca vazia. Não altera a extensão de um nome já válido.

- [ ] **Step 1: Escrever o teste**

```js
TESTES.push({ nome: "nomeSeguro limpa o que o Windows recusa", fn: () => {
  assertEq(lib.nomeSeguro("docker-compose.yml"), "docker-compose.yml", "nome válido passa intacto");
  assertEq(lib.nomeSeguro("pasta/arquivo.txt"), "pasta-arquivo.txt", "barra vira hífen");
  assertEq(lib.nomeSeguro('re:la"to*rio?.txt'), "re-la-to-rio-.txt", "caracteres proibidos viram hífen");
  assertEq(lib.nomeSeguro("  espacos.md  "), "espacos.md", "corta espaços das pontas");
  assertEq(lib.nomeSeguro("ponto.final."), "ponto.final", "ponto no fim é removido");
  assertEq(lib.nomeSeguro("CON"), "_CON", "nome reservado do Windows ganha prefixo");
  assertEq(lib.nomeSeguro("com1.txt"), "_com1.txt", "reservado com extensão, sem diferenciar maiúsculas");
  assertEq(lib.nomeSeguro("LPT9"), "_LPT9");
  assertEq(lib.nomeSeguro("nul.log"), "_nul.log");
  assertEq(lib.nomeSeguro("///"), "arquivo.txt", "sobrou nada: cai no padrão");
  assertEq(lib.nomeSeguro(""), "arquivo.txt");
  assertEq(lib.nomeSeguro(null), "arquivo.txt");
  assertEq(lib.nomeSeguro("relatório-2026.csv"), "relatório-2026.csv", "acento é válido em nome de arquivo");
}});
```

- [ ] **Step 2: Rodar e verificar que falha**

Abrir `#/_testes`. Esperado: `lib.nomeSeguro is not a function`.

- [ ] **Step 3: Implementar**

```js
const RESERVADOS_WIN = /^(con|prn|aux|nul|com[1-9]|lpt[1-9])$/i;

// Windows recusa / \ : * ? " < > | e nomes como CON ou COM1, com ou sem extensão.
lib.nomeSeguro = nome => {
  let n = String(nome == null ? "" : nome).replace(/[/\\:*?"<>|]/g, "-").trim();
  n = n.replace(/\.+$/, "").trim();
  if (!n.replace(/[-\s.]/g, "")) return "arquivo.txt";
  const base = n.includes(".") ? n.slice(0, n.lastIndexOf(".")) : n;
  if (RESERVADOS_WIN.test(base)) n = "_" + n;
  return n;
};
```

- [ ] **Step 4: Rodar e verificar que passa**

Abrir `#/_testes`. Esperado: **88/88**.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: lib.nomeSeguro limpa nomes de arquivo invalidos no Windows

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 4: A ferramenta e a verificação do download

Monta a interface e resolve o risco técnico: confirmar que o download funciona por `file://`.

**Files:**
- Modify: `index.html` — bloco `<style>`, seção `TOOLS`
- Modify: `README.md`

**Interfaces:**
- Consumes: `lib.detectarTipo`, `lib.ambiguo`, `lib.nomeSeguro`, kit `ui` (`campo`, `area`, `btn`, `linha`, `montar`, `nota`, `toast`)
- Produces: entrada em `TOOLS` com `id: "criar-arquivo"`

- [ ] **Step 1: Verificar o risco do download ANTES de construir a ferramenta**

Este passo existe porque o spec registra o download por `file://` como risco não confirmado. Verificar primeiro evita construir a interface inteira em cima de algo que não funciona.

Criar um arquivo descartável na pasta de rascunho da sessão (fora do projeto) com:

```html
<!doctype html><meta charset="utf-8">
<button id="b">Baixar teste</button>
<script>
document.getElementById("b").onclick = () => {
  const blob = new Blob(["linha 1\nlinha 2\n"], { type: "text/plain" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url; a.download = "teste-omnia.txt";
  document.body.appendChild(a); a.click(); a.remove();
  setTimeout(() => URL.revokeObjectURL(url), 1000);
};
</script>
```

Abrir pelo servidor local, clicar, e verificar que o arquivo chega. Depois abrir o mesmo arquivo direto por `file://` e repetir.

**Registrar no relatório o que aconteceu nos dois casos.** Se `file://` falhar, implementar o plano B do spec no Step 4: abrir o conteúdo em nova aba com o MIME correto e instruir o usuário a salvar com Ctrl+S — nunca deixar o botão sem efeito e sem explicação.

- [ ] **Step 2: Adicionar o CSS**

Acrescentar ao `<style>`, antes da regra `.nota {`:

```css
.arq-palpite { margin-top: 20px; padding-top: 18px; border-top: 1px solid var(--linha); }
.arq-titulo { font-size: 17px; margin-bottom: 6px; }
.arq-titulo strong { font-family: var(--mono); color: var(--acento); font-weight: 400; }
.arq-sinais { font-size: 13px; color: var(--texto-fraco); line-height: 1.6; }
.arq-alternativas { display: flex; flex-wrap: wrap; gap: 8px; margin-top: 14px; align-items: center; }
.arq-alternativas > span { font-size: 12px; color: var(--texto-fraco); letter-spacing: 1px; text-transform: uppercase; }
.arq-alt {
  padding: 6px 12px; font: inherit; font-family: var(--mono); font-size: 13px; cursor: pointer;
  background: var(--surface); color: var(--texto); border: 1px solid var(--borda); border-radius: 6px;
}
.arq-alt:hover { border-color: var(--acento); color: var(--acento); }
```

- [ ] **Step 3: Escrever a ferramenta**

Inserir antes de `TOOLS.push({\n  id: "pomodoro",`:

```js
TOOLS.push({
  id: "criar-arquivo", cat: "dev", tipo: "nativa",
  nome: "Criar Arquivo", desc: "Reconhece o tipo do conteúdo colado e baixa com o nome e a extensão certos",
  tags: ["download", "salvar", "baixar", "extensao", "docker-compose", "package.json", "yaml", "detectar tipo"],
  render(el) {
    const conteudo = ui.area("Conteúdo do arquivo", { placeholder: "Cole aqui o conteúdo — um docker-compose, um SQL, um JSON, o que for." });
    const nome = ui.campo("Nome do arquivo", "text", { placeholder: "arquivo.txt" });
    const palpite = document.createElement("div"); palpite.className = "arq-palpite"; palpite.hidden = true;
    const btnBaixar = ui.btn("Baixar", () => baixar());
    let candidatos = [], escolhido = null, nomeEditado = false;

    nome.addEventListener("input", () => { nomeEditado = true; });

    const pintar = () => {
      if (!escolhido) { palpite.hidden = true; btnBaixar.disabled = true; return; }
      const alternativas = candidatos.filter(c => c.id !== escolhido.id).slice(0, 3);
      const mostrarAlt = alternativas.length && (lib.ambiguo(candidatos) || candidatos[0].id !== escolhido.id);
      palpite.innerHTML =
        '<div class="arq-titulo">Parece um <strong>' + escolhido.nome + '</strong></div>' +
        '<div class="arq-sinais">Bateu: ' + escolhido.sinais.join(" · ") + '</div>' +
        (alternativas.length ? '<div class="arq-alternativas"><span>' +
          (mostrarAlt ? "Ou talvez" : "Outras opções") + '</span></div>' : '');
      const caixa = palpite.querySelector(".arq-alternativas");
      if (caixa) alternativas.forEach(c => {
        const b = document.createElement("button");
        b.type = "button"; b.className = "arq-alt"; b.textContent = c.nome;
        b.onclick = () => { escolhido = c; if (!nomeEditado) nome.value = c.nome; pintar(); };
        caixa.appendChild(b);
      });
      palpite.hidden = false;
      btnBaixar.disabled = false;
    };

    let timer = null;
    conteudo.addEventListener("input", () => {
      clearTimeout(timer);
      timer = setTimeout(() => {
        if (!conteudo.value.trim()) { candidatos = []; escolhido = null; pintar(); return; }
        candidatos = lib.detectarTipo(conteudo.value);
        escolhido = candidatos[0];
        if (!nomeEditado) nome.value = escolhido.nome;
        pintar();
      }, 250);
    });

    const baixar = () => {
      if (!conteudo.value.trim()) { ui.toast("Cole algum conteúdo primeiro"); return; }
      const limpo = lib.nomeSeguro(nome.value || (escolhido && escolhido.nome));
      if (limpo !== nome.value) { nome.value = limpo; ui.toast("Nome ajustado para " + limpo); }
      const blob = new Blob([conteudo.value], { type: (escolhido && escolhido.mime) || "text/plain" });
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url; a.download = limpo;
      document.body.appendChild(a); a.click(); a.remove();
      setTimeout(() => URL.revokeObjectURL(url), 1000);
    };

    btnBaixar.disabled = true;
    ui.montar(el, conteudo, palpite, nome, ui.linha(btnBaixar),
      ui.nota("O reconhecimento é um palpite baseado nos sinais listados — o nome final é sempre seu. O conteúdo baixa exatamente como está aí em cima, sem nenhuma alteração."));
  }
});
```

- [ ] **Step 4: Ajustar o download se o Step 1 mostrou que `file://` falha**

Só executar se o Step 1 registrou falha. Substituir o corpo de `baixar()` pelo caminho que funcionou, mantendo a mensagem explicativa ao usuário. Se o Step 1 mostrou que funciona, pular este passo e dizer isso no relatório.

- [ ] **Step 5: Verificar a ferramenta na prática**

Subir o servidor e conferir, citando o que apareceu na tela:

- `#/criar-arquivo` com o docker-compose da amostra da Task 2 → "Parece um **docker-compose.yml**", sinais listados, campo de nome preenchido.
- Clicar na alternativa `config.yaml` → nome muda para `config.yaml`.
- Editar o nome à mão para `meu.yml`, depois alterar o conteúdo → o nome **não** é sobrescrito.
- Colar `Reunião de terça ficou para quinta.` → "Parece um **arquivo.txt**".
- Campo vazio → botão Baixar desabilitado, nenhum palpite na tela.
- Nome `pasta/x.yml` → baixa como `pasta-x.yml` com aviso.
- Baixar e abrir o arquivo: conteúdo idêntico ao colado.
- 375px: sem rolagem horizontal na página.
- Console sem erros.

- [ ] **Step 6: Atualizar o README**

- Contagem: `27 ferramentas` → `28 ferramentas`.
- Linha **Desenvolvedor**: acrescentar ` · Criar arquivo`.
- Placar: `**82/82**` → `**88/88**`.

- [ ] **Step 7: Commit**

```bash
git add index.html README.md
git commit -m "feat: ferramenta Criar Arquivo

Cola o conteudo, ela reconhece o tipo pelos sinais, sugere nome e
extensao e baixa. Mostra em que sinais se baseou.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Auto-revisão do plano

**Cobertura do spec:**

| Requisito do spec | Task |
|---|---|
| `lib.detectarTipo(texto)` com a forma `{id, nome, ext, mime, confianca, sinais}` | Task 2 |
| Pontuação por sinais com peso, normalizada | Task 2 |
| Nunca devolve lista vazia; `.txt` como piso | Task 2 (testado) |
| Ambíguo = segundo tem ≥70% do primeiro | Task 2 (`lib.ambiguo` sobre pontos, testado nos limites) |
| Os 18 tipos | Task 1 (tabela) + Task 2 (uma amostra real de cada) |
| `lib.nomeSeguro` com reservados do Windows | Task 3 |
| Teste de integridade da tabela | Task 1 |
| Interface: textarea, palpite com sinais, alternativas clicáveis, nome editável, baixar | Task 4 |
| Reconhecimento enquanto digita, 250 ms | Task 4 |
| Editar o nome não desfaz o reconhecimento | Task 4 (`nomeEditado`) + verificado no Step 5 |
| Vazio: sem palpite, botão desabilitado | Task 4 |
| Risco do download por `file://` | Task 4, Step 1 — verificado antes de construir |
| Mostrar o porquê | Task 4 (linha "Bateu: …") |
| Critérios de sucesso 1–8 | Tasks 2 e 4, Step 5 |

Sem lacunas.

**Consistência de nomes entre tasks:** `TIPOS` e `TIPO_TXT` (Task 1) são consumidos por `lib.detectarTipo` (Task 2). `lib.detectarTipo`, `lib.ambiguo` (Task 2) e `lib.nomeSeguro` (Task 3) são consumidos pela ferramenta (Task 4). A forma do candidato — `{id, nome, ext, mime, confianca, sinais}` — é a mesma nas três tasks. `ui.area` aceita `attrs` como segundo argumento (assinatura existente: `ui.area(label, attrs = {})`), e `ui.campo(label, tipo, attrs)` com três, que é como estão usados.

**Contagem de testes:** 82 atuais + 1 (T1) + 4 (T2) + 1 (T3) = **88**. Bate com o esperado na Task 3, Step 4 e no README.

**Uma correção feita antes de você ler o plano:** rodei a tabela da Task 1 e o motor da Task 2 contra as 18 amostras da própria Task 2, em Node, antes de considerar o plano pronto. A primeira versão ordenava pela confiança normalizada e falhava no critério nº 1 do spec — o `yaml` genérico vencia o `docker-compose`. Troquei a ordenação para pontos absolutos e reverifiquei: 18/18 corretos, `docker-compose` à frente do `yaml`, `package.json` à frente do `json`, prosa em `txt`, `null` e vazio sem derrubar.

**Um ponto que deixei explícito por ser fácil de errar:** nenhum sinal da tabela usa a flag `g`. Com `g`, `regex.test()` guarda `lastIndex` entre chamadas e o mesmo sinal passa a alternar entre bater e não bater a cada texto testado — um bug intermitente e difícil de achar. A nota está na Task 2, Step 3.
