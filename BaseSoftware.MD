# BaseSoftware.MD — CenaMestre Studio

> **Nome Oficial do Programa:** Prime Scene APP
> **Repositório:** https://github.com/iwerry/PrimeScene.git

> **Documento fundacional.** Qualquer IA (ou humano) que abrir este repositório deve
> ler este arquivo ANTES de escrever uma única linha de código. Ele explica o que o
> software é, como ele funciona hoje e para onde ele vai. Se algo aqui conflitar com
> um chute seu, este documento vence.

---

## 1. O que é este software

**CenaMestre Studio** (nome de trabalho — pode ser renomeado) é um aplicativo
**desktop multiplataforma** (Windows, macOS, Linux) para **escrita de roteiros no
padrão Master Scene** ("spec format" / formato padrão de roteiro cinematográfico),
combinado com ferramentas de planejamento, validação, estimação de duração e exportação.

Ele serve a qualquer duração e formato de tela:

| Uso | Duração típica | Observação |
|---|---|---|
| Internet curto (Reels/TikTok/Shorts) | ~1 min | vertical 9:16 |
| Internet curto (feed) | 1–5 min | horizontal 16:9 |
| YouTube / web série | 5–30 min | horizontal |
| Amador / escolar | variável | orçamento zero |
| Profissional (TV, streaming, cinema) | 20–180 min | padrão indústria |

A variação entre eles **não está na gramática** (que é única — o Master Scene),
mas no **perfil de página/mídia** (proporção, duração-alvo, convenções de mercado).
Ver seção 7.

## 2. Princípios inegociáveis (a filosofia do projeto)

1. **O programa É a Skill.** O cérebro do software é o pacote de conhecimento em
   `skills/skill-master-scene/` (YAML + Markdown + scripts JS sem dependências).
   O código do app só **lê, executa e apresenta** — nunca duplica a regra.
2. **Atualizar a skill atualiza o programa.** Os arquivos da skill são carregados
   **em runtime** (a cada inicialização), não compilados para dentro do binário.
   Trocou o YAML/MD/JS → o comportamento muda sem recompilar nada.
3. **Nomes originais preservados.** Os arquivos da skill mantêm exatamente os nomes
   do repositório de origem (`SKILL.md`, `format-profile.yaml`, `scripts/*.js`, …)
   para permitir atualização por substituição / `git pull`.
4. **Inspiração, não cópia.** Starc (story-apps/starc) e KIT Scenarist são a
   referência de UX e de matemática (eles já validaram o modelo: 117 páginas =
   117 min, estatísticas por elemento, etc.). **Nunca copiar código deles.**
   Ler para entender o comportamento; implementar do zero.
5. **Nunca travar a escrita.** Só regras de severidade `error` bloqueiam.
   `warning` e `style` viram notas laterais — o autor decide.
6. **Nunca inventar números.** Specs não verificados (perfis não-EUA) entram com
   `verified: false` / `TODO-VERIFY`. Dado inventado é bug.

## 3. Stack tecnológico (decisão fechada)

| Camada | Tecnologia | Por quê |
|---|---|---|
| Shell desktop | **Tauri 2** (Rust + webview do sistema) | build único para Win/Mac/Linux, binários pequenos (~10 MB), nativo |
| Frontend | **TypeScript** + **CodeMirror 6** (ou Monaco) | editor de página fixa sem reinventar editor de texto |
| Motor da skill | **JavaScript/TS puro** | os `scripts/*.js` da skill são sem dependências → rodam sem adaptação |
| Estado/documento | **JSON** (formato de documento próprio) + **Fountain** como intercâmbio canônico | simples, diffável, portável |
| Empacotamento | `tauri build` → `.msi`/`.exe` (Win), `.dmg` (Mac), `.AppImage` (Linux) | pedido: fácil de compilar |
| Dev local | **VS Code** + rust-analyzer + rustup + Node ≥ 20 | fluxo que o autor já usa |

Alternativas descartadas e por quê (para uma IA futura não reabrir o debate sem motivo):
- **C++/Qt (igual Starc)**: build pesadíssima (qmake, Crashpad, submodules privados) — inviável para dev solo.
- **Electron**: válido como plano B (100% JS), mas binários ~5× maiores e mais RAM.
- **Flutter/.NET/Python+Qt**: exigem interpretador JS embarcado para a skill — atrito sem ganho.

## 4. Arquitetura

```
┌────────────────────────────────────────────────────────┐
│  UI (Tauri webview, TypeScript)                        │
│  editor de página · navegador de cenas · timeline      │
│  cartões/outline · painel de notas (lint) · export     │
└───────────────┬────────────────────────────────────────┘
                │ chamadas tipadas (invoke)
┌───────────────▼────────────────────────────────────────┐
│  ENGINE (TypeScript puro, sem dependências da UI)      │
│  ├─ format   → gramática de elementos + geometria      │
│  ├─ validate → lint + matriz de severidade             │
│  ├─ export   → Fountain, PDF, estimativa de duração    │
│  └─ model    → documento/cenas/projeto (JSON)          │
└───────────────┬────────────────────────────────────────┘
                │ leitura em runtime (fs), NUNCA bundle
┌───────────────▼────────────────────────────────────────┐
│  skills/skill-master-scene/  ← O CÉREBRO (atualizável) │
│  SKILL.md · format-profile.yaml · data/ · references/  │
│  profiles/ · scripts/ · examples/                      │
└────────────────────────────────────────────────────────┘
```

### Layout de pastas (alvo)

```
cenamestre/
├── BaseSoftware.MD              ← este documento
├── skills/
│   └── skill-master-scene/      ← clonado de github.com/iwerry/skill-master-scene
│       │                           (com .git → updates via `git pull`)
│       ├── SKILL.md             ← índice dos 3 módulos
│       ├── format-profile.yaml  ← perfil canônico (geometria, elementos)
│       ├── data/                ← element-coordinates.yaml, severity-matrix.yaml
│       ├── references/          ← 01..10: gramática, geometria, quebras, fountain…
│       ├── profiles/            ← us-studio-feature (default), br-telenovela, etc.
│       ├── scripts/             ← validate-format.js, lint-style.js,
│       │                           convert-fountain.js, estimate-runtime.js
│       └── examples/            ← exemplos canônicos de página
├── app/                         ← frontend TS (UI)
├── src-tauri/                   ← shell Rust (janela, fs, dialogs, impressão)
└── docs/                        ← decisões técnicas (ADRs), contrato da engine
```

### Contrato de runtime (a regra de ouro do cérebro)

1. Na inicialização, o app **descobre** `skills/skill-master-scene/` relativo ao
   executável (ou via variável/config de override).
2. O engine **parseia** `format-profile.yaml`, `data/*.yaml` e `profiles/*.yaml`.
3. Os `scripts/*.js` são **importados/executados como módulos JS** — nunca
   reescritos, nunca renomeados, nunca "adaptados" para dentro do app.
4. A skill só é **lida**. Escrita só acontece via atualização externa (git/manual).
5. Falha ao carregar a skill → o app abre em modo degradado (só editor puro) e
   avisa; nunca cai silenciosamente.

## 5. O cérebro: o que cada módulo da skill faz

A skill vem organizada em 3 sub-módulos (ver `SKILL.md` de origem). Eles viram
diretamente as 3 camadas do engine:

| Módulo skill | Responsabilidade | Arquivos de origem | Camada do app |
|---|---|---|---|
| `skill-screenplay-format` | gramática dos 9 elementos, geometria de página, perfis culturais/mídia | `format-profile.yaml`, `data/element-coordinates.yaml`, `references/01-05,08-09`, `profiles/*` | `engine/format` |
| `skill-screenplay-validate` | lint de formato + estilo, matriz de severidade | `data/severity-matrix.yaml`, `scripts/validate-format.js`, `scripts/lint-style.js`, `references/03-04,06` | `engine/validate` |
| `skill-screenplay-export` | conversão Fountain/.fdx, estimativa de duração | `scripts/convert-fountain.js`, `scripts/estimate-runtime.js`, `references/10-fountain-mapping.md` | `engine/export` |

**Gramática dos 9 elementos** (identificados por posição de coluna + caixa, não por
markup — esta é a essência do Master Scene):

1. **Scene Heading** (slugline) — 1.5", ALL CAPS, `INT./EXT. LOCAL - MOMENTO`
2. **Action** — 1.5", misto, 3ª pessoa presente, só o que câmera/microfone captam
3. **Character** — ~3.7", ALL CAPS, extensões `(V.O.) (O.S.) (CONT'D)`
4. **Parenthetical** — ~3.1", `(...)`, máx. 1 linha, ação física breve (nunca emoção/"wryly")
5. **Dialogue** — 2.5", misto, sem aspas
6. **Transition** — alinhado à direita (6.0"), ALL CAPS, `FADE IN:`, `CUT TO:`…
7. **Page Number** — topo direito, formato `N.`
8. **Scene Number** — ambas margens, apenas modos produção/shooting
9. **Title Page** — página própria, fora da gramática do corpo

**Modelo de duração:** 1 página ≈ 1 minuto de tela (dramas correm ligeiramente mais
longos — anotar como nota `style`, nunca como correção).

**Regra de severidade:** `error` = quebra de renderização/interop → trava.
`warning`/`style` = sugestão → painel de notas.

## 6. Modelo de domínio (o documento de roteiro)

```
Projeto (título, logline, mídia-alvo, perfil, idioma)
 └── Documento (roteiro | outline | cards) + Rascunhos (drafts)
      └── Cena (nº, slugline, posição temporal, duração estimada, personagens)
           └── Elemento (tipo ∈ 9 elementos, ordem, conteúdo, atributos)
```

- Formato de persistência: **JSON próprio** (especificado em `docs/` quando criado),
  versionado, com um documento = um arquivo.
- **Fountain** é o formato de intercâmbio canônico (import/export).
- `.fdx` é apenas alvo de export documentado (factual, sem código de terceiros).
- PDF é renderização fiel da geometria (Courier 12, margens calibradas).

## 7. Perfis de mídia/mercado (curto vertical ↔ cinema)

A gramática é única; o que muda por formato é o **perfil**:

- Perfis vivem em `profiles/*.yaml` da skill (hoje: `us-studio-feature` como padrão
  **verificado**; `br-telenovela` e outros parcialmente `TODO-VERIFY`).
- **Perfis novos = arquivos YAML novos** (nunca código novo): ex. `web-vertical-9x16.yaml`,
  `web-short-16x9.yaml`, `youtube-longform.yaml`. Enquanto não calibrados, entram
  com `verified: false` e o app exibe um selo "não verificado" na UI.
- Perfis controlam: proporção da página, duração-alvo (páginas/min), convenções de
  mercado (ex.: telenovela tem quebras de capítulo), e arquétipo de estilo aplicado.

## 8. Padrão visual (as 5 peças de todo software de gênero)

Estudadas nas capturas do Starc e confirmadas pelo mercado (Final Draft, Arc Studio,
Highland). Nosso app precisa ter:

1. **Editor de página fixa** — papel branco, Courier 12, margens Master Scene,
   numeração de página, indicadores de margem/revisão.
2. **Navegador de cenas** — lista clicável de sluglines com resumo e cor por linha
   dramática.
3. **Timeline/duração** — coluna com duração por cena + duração total (vem do
   `estimate-runtime.js`).
4. **Estatísticas por elemento** — pizza/tabela com % de Action/Dialogue/etc. (a
   matemática do Starc já prova o formato do relatório — implementar do zero).
5. **Export** — Fountain, PDF; `.fdx` depois.

Autocomplete de elementos no editor: teclas `Tab`/`Enter` ciclam
`Action → Character → Dialogue` etc. — convenção universal do gênero.

## 9. Roadmap (como será no futuro)

### v1 — MVP (escrever bem e certo)
- [ ] Scaffold Tauri 2 + TS (builda nos 3 SOs)
- [ ] Loader da skill em runtime (contrato da seção 4)
- [ ] Editor Master Scene com autocomplete de elementos e geometria de página
- [ ] Navegador de cenas + duração por cena e total
- [ ] Lint em tempo real (`error` trava/navega; `warning`/`style` em painel)
- [ ] Export/import Fountain + export PDF
- [ ] Perfis: `us-studio-feature` + rascunhos `web-vertical-9x16`, `web-short-16x9`

### v2 — Planejar
- [ ] Cartões/outline por cena, mapa de atos
- [ ] Estatísticas por elemento (relatório de texto)
- [ ] Personagens e locais (relatórios de elenco/cenário)
- [ ] Multi-documento/multi-rascunho, backups automáticos, título/página de rosto

### v3 — Inteligência
- [ ] AI Assistant (rephrase, expand, shorten, insert, summarize, translate, generate)
      — como camada opcional, nunca obrigatória; regras da skill continuam mandando
- [ ] Ditado por voz, atalhos customizáveis, temas (claro/escuro/custom)
- [ ] Export `.fdx`, RTL/i18n, modo série (temporadas/episódios)
- [ ] Sincronização/colaboração (avaliar; decisão futura)

## 10. Como iniciar o desenvolvimento localmente (VS Code)

Pré-requisitos: **Node ≥ 20**, **Rust (rustup)**, toolchain do sistema
(`build-essential`/Xcode/MSVC). No VS Code: extensões `rust-analyzer`, `CodeLLDB`.

```bash
# 1. gerar o scaffold oficial Tauri 2 dentro de app/ + src-tauri/
npm create tauri-app@latest
#    (frontend: TypeScript; identificador: studio.cenamestre.app)

# 2. rodar em modo dev
npm run tauri dev

# 3. empacotar
npm run tauri build   # gera .msi/.dmg/.AppImage
```

A skill já está em `skills/skill-master-scene/` — **não mexer nos nomes de arquivo**.
Ordem de implementação sugerida: loader da skill → engine/format → editor → lint → export.

## 11. Licença e limites legais (leia antes de copiar qualquer coisa)

- A skill `skill-master-scene` é **GPL-3.0** → o software que a executa deve ser
  **GPL-3.0 compatível**. Recomendação: publicar o app como GPL-3.0.
- **Starc / KIT Scenarist são GPL-3.0**: estudar comportamento e matemática é livre;
  **copiar trechos de código cria contaminação de licença e quebra a premissa do
  projeto** — implementação sempre do zero.
- Medidas de página, nomes de elementos e regras de gramática são fatos/métodos de
  indústria — uso livre. Nomes de produtos de terceiros só como citação factual.
- Marca "Master Scenes" / autores da skill: creditar `REFERENCES.md` de origem.

## 12. Convenções para IAs que trabalhem neste repositório

1. Ler `BaseSoftware.MD` + `skills/skill-master-scene/SKILL.md` antes de codar.
2. **Não renomear, mover ou editar** arquivos dentro de `skills/skill-master-scene/`
   (é dado versionado externamente; updates via `git -C skills/skill-master-scene pull`).
3. Direção da verdade: **skill manda, código obedece**. Se o código e o YAML
   divergirem, o YAML está certo.
4. Carregar só os arquivos que a tarefa exige (o SKILL.md de origem é explícito:
   não ler todas as referências a cada requisição).
5. Números de perfis não verificados nunca são apresentados como canônicos.
6. Docs em **português**; identificadores de código em **inglês**.
7. Uma mudança de comportamento do formatador/linter = atualizar a skill OU o
   engine, nunca os dois com regras duplicadas.

## 13. Glossário

- **Master Scene / spec format**: layout padrão de roteiro; nome vem da *master
  scene heading* (slugline) que abre cada cena.
- **Slugline / Scene Heading**: `INT. LOCAL - MOMENTO`, tudo em caixa alta.
- **Wryly**: parenthetical que diz emoção/entrega ao ator — anti-pattern de estilo.
- **Fountain**: sintaxe leve de markup para roteiros; nosso intercâmbio canônico.
- **Severity matrix**: tabela `error|warning|style` das regras de lint.
- **Perfil**: YAML que calibra página/mídia/mercado por cima da gramática única.
- **Skill**: o pacote de conhecimento em `skills/skill-master-scene/` — o cérebro.

---

*Documento vivo: atualize seções 9 (roadmap) e 12 (convenções) conforme o projeto
crescer. As seções 2, 4 e 11 são princípios — mudar só com decisão consciente.*
