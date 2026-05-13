# 🌙 Antigravity Overnight Build Prompt — Áskesis v1.0

> **Como usar:** cole o bloco abaixo no Antigravity em uma sessão nova,
> anexe os 3 arquivos (`claude.md`, `askesis_app_mockup.html`,
> `Prompt_BLAST.md`), envie, e vá dormir.

---

```
Você é o System Pilot do projeto Áskesis. Sua missão: entregar um APK Android
funcional ao fim desta sessão, sem fazer perguntas. Trabalhe autonomamente até
o fim — eu estarei dormindo.

═══════════════════════════════════════════════════════════════
ANEXOS (use como fonte da verdade, nessa ordem de precedência):
1. claude.md              → CONSTITUIÇÃO (lei). Schemas e invariantes são
                            imutáveis. Inclui tokens visuais, seed de tags,
                            crash recovery e cobertura mínima de testes.
2. askesis_app_mockup.html → CONTRATO VISUAL (paleta, layout, fluxo, modal
                            de cancelamento, "Sem tag" para órfãs).
3. Prompt_BLAST.md         → metodologia já adaptada para Flutter mobile
                            (Layer 1 = SOPs / Layer 2 = Riverpod+go_router /
                            Layer 3 = Dart engines). Use como guia de
                            processo (Blueprint → Architect → Stylize →
                            Trigger). Pule a fase Link (offline-first).
═══════════════════════════════════════════════════════════════

REGRAS DE NOMENCLATURA
- A única constituição é `claude.md`. Qualquer referência cruzada a
  `gemini.md` deve ser lida como `claude.md`.
- `tools/` (do BLAST original Python) é substituído por `lib/data/`,
  `lib/domain/`, `lib/services/` (Dart).

DECISÕES JÁ TOMADAS (não pergunte — apenas execute)
- Stack: Flutter (Dart 3.x) + Isar 3.x + Riverpod 2.x + go_router + fl_chart +
  flutter_foreground_task + flutter_local_notifications + tabler_icons.
- Idioma da UI: pt-BR. Código e comentários: inglês.
- Tema: dark only na v1. Tokens visuais EXATOS estão em claude.md
  §"Visual Contract".
- Fontes: DM Serif Display + DM Sans (DMSans-Light/Regular/Medium).
  Embarcar .ttf em `assets/fonts/` e declarar em pubspec.yaml.
- Min SDK: Android 8.0 (API 26). compileSdk 34. NDK conforme exigido pelo
  Isar.
- Seed no primeiro launch: 4 tags default — Estudo (#4A90D9, ti-book-2),
  Research (#9B6FC7, ti-microscope), Saúde (#4CAF72, ti-heartbeat),
  Revisão (#C97A3A, ti-refresh). Marcar `is_default=true`.
- Tag deletada: FocusSessions órfãs mantêm `tag_id` (soft reference). UI
  renderiza como "Sem tag" com cor #555. Deletar tag em uso pede
  confirmação.
- Crash recovery do Focus: persistir `ActiveFocusSnapshot` em Isar a cada
  5s; ao reabrir o app, modal "Retomar sessão de [tag] iniciada há X min?"
  com Retomar/Descartar. (Detalhes: claude.md §"Crash Recovery".)
- Cancelar timer: SEMPRE AlertDialog antes de cancelar (invariante #3).
  Texto: "Cancelar sessão? A sessão atual será descartada. O tempo
  decorrido não será registrado como completo." Botões: "Voltar" e
  "Cancelar" (vermelho).
- Cobertura mínima:
    FocusSessionService → 100%
    Repositórios        → ≥ 70%
    Smoke widget tests  → 1 por aba
    `flutter analyze`    → zero warnings

PLANO DE EXECUÇÃO (siga em ordem, marque progresso em progress.md)

FASE 0 — Bootstrap
  [ ] Verificar Flutter: `flutter --version` e `flutter doctor -v`. Se algo
      faltar, instalar/configurar antes de prosseguir.
  [ ] `flutter create . --org com.askesis --project-name askesis
        --platforms=android`
  [ ] Criar estrutura de pastas conforme claude.md §"File Structure".
  [ ] Adicionar dependências no pubspec.yaml fixando a versão estável mais
      recente compatível com Dart 3.x. Se houver incompatibilidade, fixar a
      última versão funcional anterior e registrar em findings.md.
  [ ] Baixar fontes DM Serif Display + DM Sans (Light/Regular/Medium) do
      Google Fonts, salvar em `assets/fonts/`, declarar em pubspec.yaml.
  [ ] Configurar tema dark em `lib/app/theme.dart` consumindo
      `lib/app/tokens.dart` com a paleta de claude.md §"Visual Contract".
  [ ] Configurar go_router com ShellRoute + 4 abas (To-do, Focus, Review,
      Dashboard) e bottom nav idêntica ao mockup.
  [ ] Commit: "chore: bootstrap flutter project"

FASE 1 — Domain & Data
  [ ] Modelos Isar: Task, FocusSession, Tag, ActiveFocusSnapshot — schemas
      EXATOS de claude.md §"Data Schema". Rodar build_runner para gerar
      `.g.dart`.
  [ ] Repositórios: TaskRepository, FocusSessionRepository, TagRepository
      — interface em `domain/`, implementação Isar em `data/`.
  [ ] `lib/data/seed.dart`: criar as 4 tags default no primeiro launch (só
      se o box estiver vazio).
  [ ] Testes unitários (≥70%) de cada repositório.
  [ ] SOPs em `architecture/sop_task.md`, `sop_focus_session.md`,
      `sop_review.md`, `sop_dashboard.md`.
  [ ] Commit: "feat(data): isar schemas + repositories + seed"

FASE 2 — Focus (núcleo sagrado, ANTES da UI das outras abas)
  [ ] `FocusSessionService` (lib/services/): máquina de estados
      idle → running → paused → finished | cancelled. Persiste
      `ActiveFocusSnapshot` a cada 5s, em pause/resume, e ao mudar de
      estado. Integra com flutter_foreground_task.
  [ ] Testes 100% do service: countdown completa, stopwatch, pause/resume,
      cancel descarta com completed=false, retomar snapshot recompõe
      `elapsed_seconds` somando delta de `last_tick_at`.
  [ ] UI Focus EXATAMENTE como o mockup: toggle Temporizador/Cronômetro,
      seletor de tag obrigatório (default Estudo), ring SVG com progresso
      dourado, presets 15/25/50/90 min, AlertDialog de confirmação no
      cancel.
  [ ] Notificação local ao completar a sessão (canal "focus_complete").
  [ ] No `main.dart`, checar ActiveFocusSnapshot ANTES do primeiro frame
      e abrir modal Retomar/Descartar se existir.
  [ ] Commit: "feat(focus): timer/stopwatch with foreground service + crash recovery"

FASE 3 — To-do
  [ ] 3 sub-views (Lista, Kanban, Calendário) com toggle no topo idêntico
      ao mockup.
  [ ] CRUD de tarefas via bottom sheet (título, descrição, prioridade,
      tags múltiplas, due_date opcional).
  [ ] Lista: agrupada por data (Hoje / Amanhã / Próximos / Atrasadas);
      check circular dourado quando done; tags pintadas; bolinha de
      prioridade nas cores danger/warn/success.
  [ ] Kanban: 3 colunas (To-do / Em curso / Feito) com drag-to-move
      atualizando `status`.
  [ ] Calendário: mini-calendar mensal com indicador nos dias com tarefas
      (cor `text` em vez de `muted`); seleção de dia mostra lista abaixo.
  [ ] Commit: "feat(todo): list/kanban/calendar views"

FASE 4 — Review
  [ ] Histórico cronológico desc das FocusSessions (paginated, 50 por vez).
  [ ] Filtros: chips de tag (acrescentar chip "Sem tag" para órfãs) +
      intervalo de data via date range picker.
  [ ] Item exibe: ícone na cor da tag (ou neutro se órfã), título (task
      vinculada se houver, senão nome da tag), meta (data/hora/tag),
      duração, "Cancelada" em vermelho mudo quando aplicável.
  [ ] Commit: "feat(review): session history with filters"

FASE 5 — Dashboard
  [ ] 4 sub-views (Overview, Day, Week, Year) — toggle idêntico ao mockup.
  [ ] Overview: 4 stat cards (Foco total, Sessões, Tarefas feitas, Dias
      ativos) + barras de proporção por tag (top 4 tags do período).
  [ ] Day: bar chart 6h→23h de foco por hora + 2 stat cards
      (Foco hoje, Sessões hoje).
  [ ] Week: fl_chart bar chart dos últimos 7 dias + Total semana +
      Média/dia. Dia atual destacado em dourado.
  [ ] Year: heatmap COMPLETO estilo GitHub — ~52 colunas × 7 linhas
      (~365 células). Use `SingleChildScrollView` horizontal. Buckets de
      intensidade: 0, 1–15min, 16–60min, 61–120min, >120min mapeando para
      `surfaceAlt`, `hm-1`, `hm-2`, `hm-3`, `hm-4` (ver tokens). Legenda
      Menos→Mais + stat cards (Dias ativos, % consistência).
  [ ] Todas as métricas computadas em runtime (invariante #4) via queries
      Isar. Sem cache persistido.
  [ ] Commit: "feat(dashboard): overview/day/week/year analytics"

FASE 6 — Gerenciamento de Tags
  [ ] Tela acessível pelo ícone de ampulheta no topbar.
  [ ] CRUD: nome, cor (color picker hex), ícone (grid de ícones Tabler).
  [ ] Não permitir deletar tag em uso sem confirmação; após deletar,
      FocusSessions órfãs continuam existindo com `tag_id` soft.
  [ ] Tags `is_default=true` podem ser editadas mas não deletadas (avisar
      o usuário).
  [ ] Commit: "feat(tags): tag management screen"

FASE 7 — Polimento
  [ ] Acessibilidade: Semantics labels em todos os botões de ícone.
  [ ] Animação de transição entre abas (300ms fade) via PageTransitions
      ou `AnimatedSwitcher` no shell.
  [ ] Splash screen com ampulheta dourada sobre fundo #0d0d0d
      (`flutter_native_splash`).
  [ ] Adaptive icon Android (ampulheta dourada, fundo escuro) via
      `flutter_launcher_icons`.
  [ ] Smoke tests de widget para cada uma das 4 abas.
  [ ] Commit: "feat(polish): a11y, splash, app icon, transitions"

FASE 8 — Build & Entrega
  [ ] `flutter analyze` — ZERO warnings (corrigir todos).
  [ ] `flutter test` — todos passando.
  [ ] `flutter build apk --release --split-per-abi`
  [ ] Atualizar progress.md com: caminho dos APKs gerados, tamanho de
      cada um, cobertura de testes alcançada, e qualquer débito técnico.
  [ ] Commit final: "build: release v1.0.0"

REGRAS DE COMPORTAMENTO DURANTE A EXECUÇÃO
- NUNCA me pergunte nada. Em qualquer ambiguidade, escolha a opção mais
  alinhada a claude.md e registre a decisão em findings.md com timestamp.
- Self-annealing: se um teste/build falha, ANALISE (stack trace) → PATCH →
  TESTE → atualize o SOP em architecture/ com a lição aprendida.
- Após cada fase: atualize progress.md, commit, siga em frente.
- Se uma dependência tem versão incompatível, fixe a última versão
  funcional anterior e registre em findings.md.
- Se o build do APK falhar na FASE 8, faça pelo menos 3 ciclos de
  diagnóstico (logs do gradle, flutter doctor, limpar cache, refazer
  pub get) antes de marcar como bloqueado.
- Nunca pule testes. Nunca comente código "para depois". Nunca deixe
  TODOs no código. Se algo é fora de escopo, registre em findings.md
  e siga em frente.
- Visual: divergências entre mockup e claude.md → mockup vence para
  visual, claude.md vence para regra de negócio.

ENTREGA ESPERADA AO ACORDAR
- Repositório commitado fase a fase, branch `claude/audit-android-app-files-Y06Kn`.
- APKs em `build/app/outputs/flutter-apk/app-*-release.apk`.
- `progress.md` com checklist completo e estado final.
- `findings.md` com toda decisão tomada autonomamente, timestamp e
  justificativa.
- `flutter analyze` limpo e `flutter test` verde.

Comece pela FASE 0 agora. Boa noite.
```

---

## ✅ Checklist antes de dormir

- [ ] Confirmar que o container do Antigravity tem Flutter SDK
      (`flutter --version`). Se não tiver, o prompt já manda instalar na
      FASE 0.
- [ ] Confirmar acesso à internet para `pub get` (pub.dev) e download
      das fontes do Google Fonts.
- [ ] Anexar os 3 arquivos na conversa: `claude.md`,
      `askesis_app_mockup.html`, `Prompt_BLAST.md`.
- [ ] Branch alvo: `claude/audit-android-app-files-Y06Kn`.
- [ ] Bons sonhos.
