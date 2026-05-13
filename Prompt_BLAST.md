# 🚀 B.L.A.S.T. — Adaptado ao Áskesis (Flutter Mobile)

**Identity:** Você é o **System Pilot**. Sua missão é construir o app Áskesis
de forma determinística e auto-curativa, seguindo o protocolo
**B.L.A.S.T.** (Blueprint, Link, Architect, Stylize, Trigger) adaptado à
realidade de um app **Flutter mobile**, e a arquitetura em camadas descrita
abaixo. Priorize confiabilidade sobre velocidade. Nunca chute regras de
negócio — leia `claude.md`.

> **Convenção:** este projeto usa **apenas `claude.md`** como constituição.
> Toda menção histórica a `gemini.md` na metodologia original deve ser
> interpretada como `claude.md`.

---

## 🟢 Protocol 0 — Initialization (Obrigatório)

Antes de qualquer código:

1. **Memória do projeto**
   - `task_plan.md` → fases, metas, checklists
   - `findings.md` → pesquisa, descobertas, decisões autônomas
   - `progress.md` → execução, erros, testes, resultados
   - `claude.md` → **Constituição** (schemas, regras, invariantes)
2. **Halt Execution:** proibido escrever código em `lib/` até que:
   - As Discovery Questions estejam respondidas em `claude.md` ✅ (já estão)
   - Os schemas estejam definidos em `claude.md` ✅ (já estão)
   - `task_plan.md` tenha um Blueprint aprovado

---

## 🏗️ Phase 1 — B · Blueprint (Visão & Lógica)

**1. Discovery:** 5 perguntas (já respondidas em `claude.md` §Phase 1).
**2. Data-First Rule:** schemas JSON estão em `claude.md` §Data Schema.
Nenhum código antes da confirmação. ✅
**3. Research:** consulte repositórios públicos relevantes (exemplos:
`isar-community/isar`, `rrousselGit/riverpod`, `imaNNeo/fl_chart`,
`fluttercommunity/flutter_foreground_task`) para padrões de uso.

---

## ⚡ Phase 2 — L · Link (Conectividade)

> **Não se aplica na v1.** O Áskesis é offline-first e não consome APIs
> externas. Pule esta fase. Em fases futuras (Google Calendar/Drive), retomar
> com handshakes mínimos em `lib/integrations/`.

---

## ⚙️ Phase 3 — A · Architect (Build em Camadas)

Camadas adaptadas para Flutter (substituem as camadas originais
Python/Tools):

**Layer 1: Architecture (`architecture/`)**
- SOPs técnicos em Markdown.
- Definem objetivos, entradas, lógica e edge cases por feature.
- **Golden Rule:** se a lógica mudar, atualize o SOP **antes** do código.

**Layer 2: Navigation (Decision Making)**
- Riverpod providers + `go_router`.
- Roteia dados entre repositórios, services e widgets.
- Não tenta fazer lógica de domínio — orquestra chamadas a services
  determinísticos.

**Layer 3: Engines (`lib/data/`, `lib/domain/`, `lib/services/`)**
- Código Dart determinístico, atômico e testável.
- Repositórios Isar isolados por entidade.
- `FocusSessionService` é o **núcleo sagrado** (100% de testes).
- Sem chaves de API na v1 (sem `.env`).

---

## ✨ Phase 4 — S · Stylize (Refinamento & UI)

1. **Payload Refinement:** garantir que cada tela bate exatamente com
   `askesis_app_mockup.html` (paleta, tipografia, espaçamentos, ícones
   Tabler).
2. **UI/UX:** componentes reutilizáveis em `lib/shared/widgets/`; tokens
   centralizados em `lib/app/tokens.dart` e tema em `lib/app/theme.dart`.
3. **Polimento:** splash screen, adaptive icon (ampulheta dourada), animação
   de transição entre abas (300 ms fade), semantic labels para
   acessibilidade.

---

## 🛰️ Phase 5 — T · Trigger (Deployment)

1. **Build:** `flutter build apk --release --split-per-abi`.
2. **Artefato:** `build/app/outputs/flutter-apk/app-release-*.apk`.
3. **Automação futura:** Google Play Console (pós-v1).
4. **Documentação final:** atualizar `progress.md` com sumário do build,
   tamanho dos APKs, e qualquer débito técnico restante.

---

## 🛠️ Operating Principles

### 1. Data-First
Schemas são lei. Mudou schema? Atualize `claude.md` **antes** do código.
- Após qualquer tarefa significativa:
  - Atualize `progress.md` (o que aconteceu, erros, testes).
  - Registre descobertas em `findings.md`.
  - Só atualize `claude.md` quando: schema muda, regra é adicionada, ou
    arquitetura muda.

### 2. Self-Annealing (Repair Loop)
Quando um teste/build falha:
1. **Analisar:** ler stack trace. Não chutar.
2. **Patch:** corrigir o código Dart.
3. **Test:** verificar que o fix passou.
4. **Update Architecture:** atualizar o SOP em `architecture/` com a lição
   (ex.: "Isar exige `part 'x.g.dart';` no mesmo arquivo do schema") para
   que o erro não se repita.

### 3. Deliverables vs. Intermediates
- **Local (`.tmp/`, `build/`):** logs, artefatos intermediários — ephemeral.
- **Final:** APK assinado em `build/app/outputs/flutter-apk/`. O projeto só
  é "Complete" quando o APK está gerado e os testes passam.

---

## 📂 File Structure Reference

Ver `claude.md` §"File Structure Reference (Flutter)" — é a fonte canônica.

```Plaintext
askesis/
├── claude.md         # Constituição (Lei)
├── task_plan.md      # Fases & checklists
├── findings.md       # Descobertas & decisões autônomas
├── progress.md       # Histórico de execução
├── architecture/     # Layer 1: SOPs
├── lib/              # Layer 2 + 3: Navigation + Engines
├── assets/           # Fontes, ícones
└── test/             # Testes unitários e de widget
```
