# ⏳ Áskesis — Project Constitution (`claude.md`)
> Este arquivo é **lei**. Nenhuma decisão arquitetural, schema de dados ou regra
> de comportamento pode ser alterada sem atualizar este documento primeiro.
>
> **Convenção:** este projeto usa **apenas `claude.md`** como constituição.
> Qualquer referência a `gemini.md` em documentos auxiliares deve ser lida como
> `claude.md`.

---

## 🟢 Protocol 0 — Initialization Log

| Artifact | Status |
|---|---|
| `claude.md` (Constituição) | ✅ Criado |
| `task_plan.md` (Fases & Checklist) | ✅ Criado |
| `findings.md` (Descobertas & Restrições) | ✅ Criado |
| `progress.md` (Histórico de execução) | 🔲 A iniciar |

**Halt Status:** `RELEASED` — Schema confirmado. Fase 2 (Link) é dispensada
(offline-first, sem APIs externas na v1). Fase 3 (Architect) liberada.

---

## 🏗️ Phase 1 — B · Blueprint: Discovery Answers

### 1. ⭐ North Star
Entregar um **aplicativo Android de produtividade pessoal** — **Áskesis** (do
grego: prática disciplinada, exercício da vontade) — que unifica em uma
interface minimalista de 4 abas:
- Gestão de tarefas com múltiplas visualizações
- Foco cronometrado com sistema de tags
- Revisão do histórico de atividade
- Análise de desempenho pessoal

O app deve ser **offline-first**, rápido e sem fricção. O usuário deve sentir
que o app trabalha *para* ele, não *contra* ele.

### 2. 🔗 Integrations
**v1.0 — Standalone (sem integrações externas):**
- Sem APIs de terceiros
- Sem autenticação de servidor
- Sem sincronização em nuvem
- Sem chaves de API necessárias

**Roadmap futuro (pós-v1):**
- Google Calendar (leitura de eventos → conversão em tarefas)
- Google Drive (backup do banco de dados local)
- Notificações locais via sistema Android (sem servidor push)

### 3. 🗄️ Source of Truth
**Banco de dados local no dispositivo** via **Isar DB** (Flutter).
- Local-first: 100% funcional sem internet
- Dados nunca saem do dispositivo na v1
- Backup manual opcional via exportação JSON
- Sem servidor, sem nuvem, sem Firebase na v1

### 4. 📦 Delivery Payload
- **APK Android** gerado via `flutter build apk --release --split-per-abi`
- Target SDK: Android 8.0+ (API 26+), compileSdk 34
- Distribuição inicial: sideload direto (APK)
- Distribuição futura: Google Play Store
- UI em **português (pt-BR)**; código e comentários em **inglês**

### 5. 📐 Behavioral Rules
**DO:**
- Interface minimalista, 4 abas fixas na barra inferior
- Ampulheta como ícone/logo (símbolo de tempo e disciplina)
- Paleta escura definida em "Visual Contract" (abaixo)
- Transições suaves entre abas (fade 300ms)
- **Timer Focus protegido contra interrupção acidental: cancelar SEMPRE exige
  confirmação modal**
- Tags com cor personalizável para categorizar sessões e tarefas
- Review e Dashboard são **somente leitura** — gerados automaticamente dos
  dados registrados
- Recuperação automática de sessão Focus ativa após crash/kill (ver §Crash
  Recovery)

**DON'T:**
- Sem gamificação forçada (sem streaks chamativos, sem pontos)
- Sem sincronização automática sem consentimento do usuário
- Sem notificações sem permissão explícita
- Sem tela de onboarding complexa — o app deve ser autoexplicativo
- Sem publicidade ou monetização invasiva

---

## ⚙️ Tech Stack (Fixado)

| Camada | Tecnologia | Justificativa |
|---|---|---|
| UI Framework | **Flutter (Dart 3.x)** | Cross-platform, alta performance, widgets customizáveis |
| State Management | **Riverpod 2.x** | Reativo, testável, sem boilerplate excessivo |
| Banco de Dados | **Isar 3.x** | NoSQL local ultrarrápido, native Flutter |
| Navegação | **go_router** | Declarativa, ShellRoute para a bottom nav |
| Timer em background | **flutter_foreground_task** | Mantém timer ativo mesmo com app em background |
| Notificações Locais | **flutter_local_notifications** | Alerta ao fim do timer |
| Gráficos (Dashboard) | **fl_chart** | Bar charts e custom painters |
| Ícones | **tabler_icons** | Mockup utiliza família Tabler (`ti ti-*`) |
| Fontes | **DM Serif Display** + **DM Sans** | Embarcadas em `assets/fonts/` |

> **Política de versão:** fixar a versão estável mais recente compatível com
> Dart 3.x no momento do bootstrap. Em caso de incompatibilidade, fixar a
> última versão funcional anterior e registrar em `findings.md`.

---

## 🎨 Visual Contract (Design Tokens)

A interface deve reproduzir fielmente o mockup HTML
(`askesis_app_mockup.html`). Tokens centralizados em `lib/app/theme.dart`:

| Token | Valor | Uso |
|---|---|---|
| `bg` | `#0d0d0d` | Fundo da app |
| `surface` | `#161616` | Cards, calendário, mini-painéis |
| `surfaceAlt` | `#1a1a1a` | Toggles, navbar |
| `border` | `#1e1e1e` / `#222` | Divisórias |
| `text` | `#e8e4dc` | Texto primário |
| `textMuted` | `#555` | Texto secundário |
| `textFaint` | `#333` – `#444` | Labels apagados |
| `accent` | `#c9a84c` | Dourado/ampulheta — botões primários, destaques |
| `accentSoft` | `#1f1a0a` | Background de chip selecionado |
| `danger` | `#c94c4c` | Prioridade alta, ações destrutivas |
| `warn` | `#c9a84c` | Prioridade média |
| `success` | `#4a7c4c` | Prioridade baixa, sucesso |
| `tagStudy` | bg `#1a2633` / fg `#4a90d9` | Tag default "Estudo" |
| `tagHealth` | bg `#1a2a1e` / fg `#4caf72` | Tag default "Saúde" |
| `tagResearch` | bg `#2a1e33` / fg `#9b6fc7` | Tag default "Research" |
| `tagReview` | bg `#33261a` / fg `#c97a3a` | Tag default "Revisão" |

Tipografia:
- **DM Serif Display:** títulos da app, números grandes (timer, stats).
- **DM Sans:** corpo, labels, botões.

---

## 📊 Data Schema (Payload Shapes)

> **INVARIANTE:** Nenhum código pode ser escrito sem que estes schemas estejam
> confirmados.

### Entity: `Task`
```json
{
  "id": "String (UUID)",
  "title": "String",
  "description": "String? (nullable)",
  "status": "Enum: todo | in_progress | done",
  "priority": "Enum: low | medium | high",
  "tag_ids": ["String (UUID)"],
  "due_date": "DateTime? (nullable)",
  "created_at": "DateTime",
  "updated_at": "DateTime",
  "completed_at": "DateTime? (nullable)"
}
```

### Entity: `FocusSession`
```json
{
  "id": "String (UUID)",
  "type": "Enum: timer | stopwatch",
  "target_duration_seconds": "int? (null se stopwatch)",
  "actual_duration_seconds": "int",
  "tag_id": "String (UUID)",
  "task_id": "String? (UUID, nullable)",
  "started_at": "DateTime",
  "ended_at": "DateTime",
  "completed": "bool (false se cancelada antes do fim)"
}
```

### Entity: `Tag`
```json
{
  "id": "String (UUID)",
  "name": "String",
  "color_hex": "String (ex: #4A90E2)",
  "icon_code": "int? (codepoint Tabler ou Material)",
  "created_at": "DateTime",
  "is_default": "bool (true para tags seed do primeiro launch)"
}
```

### Entity: `ActiveFocusSnapshot` (recovery)
Persistido para recuperação após crash. Existe no máximo 1 registro a qualquer
momento.
```json
{
  "id": "String (UUID)",
  "type": "Enum: timer | stopwatch",
  "target_duration_seconds": "int?",
  "elapsed_seconds": "int",
  "tag_id": "String",
  "task_id": "String?",
  "started_at": "DateTime",
  "last_tick_at": "DateTime",
  "is_paused": "bool"
}
```

### Derived: `DailyStats` (computed, não persistido)
```json
{
  "date": "String (YYYY-MM-DD)",
  "tasks_completed": "int",
  "focus_time_seconds": "int",
  "sessions_count": "int",
  "sessions_completed": "int",
  "top_tag_id": "String? (UUID)"
}
```

---

## 🌱 First-Launch Seed

No primeiro launch, criar as 4 tags default abaixo (todas com
`is_default=true`):

| name | color_hex | icon (Tabler) |
|---|---|---|
| Estudo | `#4A90D9` | `book-2` |
| Research | `#9B6FC7` | `microscope` |
| Saúde | `#4CAF72` | `heartbeat` |
| Revisão | `#C97A3A` | `refresh` |

---

## 🗂️ App Structure — 4 Abas

### Aba 1 · To-do
- 3 sub-visualizações (toggle no topo): **Lista | Kanban | Calendário**
- **Lista:** tarefas agrupadas por data (Hoje / Amanhã / Próximos / Atrasadas).
- **Kanban:** 3 colunas — `To-do` / `Em curso` / `Feito` — com drag-to-move.
- **Calendário:** mini-calendar mensal com indicador nos dias que têm tarefas
  + lista das tarefas do dia selecionado.
- CRUD via bottom sheet (título, descrição, prioridade, tags, due_date).

### Aba 2 · Focus
- Seletor de modo: **Temporizador** (countdown) | **Cronômetro** (stopwatch).
- Seletor de **Tag** obrigatório antes de iniciar.
- Vínculo opcional com uma **Task** existente.
- Tela de execução: display grande do tempo, botões Pausar / Cancelar /
  Concluir.
- **Cancelar exige confirmação modal** (invariante).
- Presets de duração no modo Temporizador: 15 / 25 / 50 / 90 min.

### Aba 3 · Review
- Histórico de sessões Focus em ordem cronológica decrescente.
- Filtro por Tag (chips dourados quando selecionados) e por intervalo de data.
- Cada item exibe: tag (cor + ícone), duração, hora de início, status
  (completa / cancelada).
- Sessões canceladas aparecem com indicador visual (texto secundário "Cancelada"
  e duração apagada).

### Aba 4 · Dashboard
- **4 sub-views:** Overview | Day | Week | Year.
- **Overview:** métricas all-time (tempo total, sessões, tarefas, dias ativos)
  + proporção por tag.
- **Day:** bar chart de foco por hora (6h–23h) + 2 stat cards (foco hoje,
  sessões hoje).
- **Week:** bar chart dos últimos 7 dias + total semanal + média/dia.
- **Year:** heatmap estilo GitHub (**~365 células**, ~52 colunas × 7 linhas,
  ordenado por semana) com legenda Menos→Mais + stat cards (dias ativos, %
  consistência).

---

## 🏛️ Architectural Invariants

1. **Local-first:** Toda lógica de negócio funciona sem internet.
2. **Schema-first:** Qualquer mudança de schema deve ser refletida aqui antes
   de alterar código.
3. **Timer é sagrado:** A lógica de `FocusSessionService` deve ter **100% de
   cobertura de testes unitários** antes de qualquer UI do Focus. Cancelamento
   exige confirmação modal — não pode ser feito sem.
4. **Derived data não é persistida:** `DailyStats` é sempre computada em
   runtime via queries Isar.
5. **Tags são obrigatórias no Focus:** Uma `FocusSession` sem `tag_id` é
   inválida e deve ser rejeitada na camada de repositório.
6. **Sem estado global mutável:** Todo estado da UI passa pelo Riverpod.
   Nenhum `setState` fora de widgets folha.
7. **Tag deletada não destrói histórico:** ao deletar uma `Tag`, suas
   `FocusSession` órfãs preservam o `tag_id` como soft reference. UI exibe
   "Sem tag" (cor neutra `#555`) quando a tag não existe mais. Deleção pede
   confirmação se a tag está em uso.
8. **Crash recovery do Focus:** o `FocusSessionService` persiste um
   `ActiveFocusSnapshot` em Isar a cada 5 segundos enquanto ativo. Ao reabrir
   o app, se houver snapshot, mostrar modal "Retomar sessão de [tag] iniciada
   há X min?" com opções Retomar / Descartar.
9. **Cobertura mínima de testes:**
   - `FocusSessionService`: **100%**.
   - Repositórios (`Task`, `FocusSession`, `Tag`): **≥ 70%**.
   - Smoke test de widget para cada uma das 4 abas.
   - `flutter analyze` deve passar com **zero warnings**.

---

## 📂 File Structure Reference (Flutter)

```
askesis/
├── claude.md                       # ← Este arquivo (Lei)
├── task_plan.md
├── findings.md
├── progress.md
├── pubspec.yaml
├── analysis_options.yaml
├── assets/
│   └── fonts/
│       ├── DMSerifDisplay-Regular.ttf
│       ├── DMSans-Regular.ttf
│       ├── DMSans-Medium.ttf
│       └── DMSans-Light.ttf
├── architecture/
│   ├── sop_task.md
│   ├── sop_focus_session.md
│   ├── sop_review.md
│   └── sop_dashboard.md
├── android/                        # gerado por `flutter create`
├── lib/
│   ├── main.dart
│   ├── app/
│   │   ├── router.dart
│   │   ├── theme.dart
│   │   └── tokens.dart             # design tokens (paleta, espaços)
│   ├── domain/
│   │   ├── models/                 # Task, FocusSession, Tag
│   │   └── repositories/           # Interfaces (abstratas)
│   ├── data/
│   │   ├── isar/                   # Schemas Isar gerados
│   │   ├── repositories/           # Implementações concretas
│   │   └── seed.dart               # Seed das 4 tags default
│   ├── services/
│   │   ├── focus_session_service.dart
│   │   └── notification_service.dart
│   ├── presentation/
│   │   ├── shell/                  # ShellRoute + bottom nav
│   │   ├── todo/
│   │   ├── focus/
│   │   ├── review/
│   │   └── dashboard/
│   └── shared/
│       ├── widgets/
│       └── providers/
└── test/
    ├── domain/
    ├── data/
    ├── services/
    └── widget/
```

---

## 🛟 Crash Recovery — Sequência

1. Ao iniciar uma sessão Focus, gravar `ActiveFocusSnapshot` em Isar.
2. A cada 5 s (e em `pause`/`resume`), atualizar o snapshot.
3. Ao concluir ou cancelar a sessão, **deletar** o snapshot e gravar a
   `FocusSession` final.
4. No `main.dart`, antes de exibir a UI, checar se existe snapshot:
   - Se sim, abrir modal "Retomar sessão?" sobre a aba Focus.
   - "Retomar" → reidratar service com `elapsed_seconds` + delta desde
     `last_tick_at`.
   - "Descartar" → persistir como `FocusSession` com `completed=false` e
     deletar snapshot.

---

## 🔄 v2 — Evolução (cloud sync + screen time + sempre-ativo)

> Os schemas e invariantes abaixo são **adições/alterações** da v2.
> Plano de execução completo em `v2_plan.md`.

### v2.1 Schemas alterados

#### `Tag` — campo novo
- `is_app_category: bool` (default `false`) — `true` se a tag categoriza
  apps (não é exclusivo com uso manual).

#### `FocusSession` — campos novos
- `source: Enum { manual, app_usage }` (default `manual`).
- `source_app_package: String?` — pacote Android quando
  `source == app_usage`.
- `synced_at: DateTime?` — null = ainda não sincronizado.
- `deleted_at: DateTime?` — soft delete (propaga via sync).

#### `Task` — campos novos
- `synced_at: DateTime?`
- `deleted_at: DateTime?`

### v2.2 Entidades novas

#### `TimerPreset`
```json
{
  "id": "String (UUID)",
  "label": "String",
  "duration_seconds": "int",
  "is_default": "bool",
  "sort_order": "int"
}
```
Seed: 15, 25, 50, 90 min (`is_default=true`).

#### `AppPackage`
```json
{
  "package_name": "String (PK)",
  "display_name": "String",
  "icon_bytes": "Uint8List?",
  "tag_id": "String?",
  "ignored": "bool",
  "first_seen_at": "DateTime",
  "last_seen_at": "DateTime"
}
```

#### `AppUsageSession`
```json
{
  "id": "String (UUID)",
  "package_name": "String",
  "started_at": "DateTime",
  "ended_at": "DateTime",
  "duration_seconds": "int",
  "tag_id": "String?",
  "promoted_to_focus_session_id": "String?",
  "synced_at": "DateTime?"
}
```
**Threshold:** `duration_seconds >= 180` (3 min) — sessões mais curtas são
descartadas na captura.

#### `UserAccount`
```json
{
  "uid": "String (Firebase UID)",
  "email": "String?",
  "display_name": "String?",
  "device_id": "String",
  "last_sync_at": "DateTime?",
  "sync_enabled": "bool"
}
```

#### `SyncCursor`
```json
{
  "collection": "String",
  "last_pulled_at": "DateTime",
  "last_pushed_at": "DateTime"
}
```

### v2.3 Invariantes adicionais

10. **App usage threshold:** sessões `AppUsageSession` com
    `duration_seconds < 180` são descartadas antes da persistência.
11. **Soft delete obrigatório** para Task, FocusSession, Tag, TimerPreset,
    AppPackage. Sem `DELETE` físico antes do push confirmado.
12. **Sync = last-write-wins por campo** via server timestamp; soft delete
    sempre vence update.
13. **Tags têm dupla natureza:** manuais (uso consciente) e/ou
    `is_app_category=true` (categoriza apps). Não são exclusivas.
14. **Permissões opt-in:** `PACKAGE_USAGE_STATS` e `SYSTEM_ALERT_WINDOW`
    são features avançadas — o app funciona 100% sem elas. Sempre exibir
    rationale antes do prompt do sistema.
15. **Sync é opt-in:** sem login → comportamento v1 (100% local). Logout
    NÃO apaga dados locais — apenas reseta `SyncCursor`.

### v2.4 Stack adicional

| Camada | Tecnologia |
|---|---|
| Auth | `firebase_auth` + `google_sign_in` |
| Cloud DB | `cloud_firestore` (cache offline nativo) |
| Background | `workmanager` (pull de UsageStats a cada 15 min) |
| Widget | `home_widget` (Flutter) + RemoteViews (Kotlin) |
| Native bridge | `MethodChannel` em Kotlin (UsageStats, Overlay) |

### v2.5 Permissões Android adicionais

`PACKAGE_USAGE_STATS`, `SYSTEM_ALERT_WINDOW`, `FOREGROUND_SERVICE`,
`FOREGROUND_SERVICE_SPECIAL_USE`, `POST_NOTIFICATIONS`, `INTERNET`,
`QUERY_ALL_PACKAGES`.

---

*Última atualização: Phase 1 — Blueprint concluído + lacunas de auditoria
fechadas + roadmap v2 incorporado (schemas, invariantes 10–15, stack
adicional). Schema confirmado. Phase 2 (Link) dispensada na v1;
reintroduzida na v2 (Firebase). Phase 3 (Architect) liberada.*
