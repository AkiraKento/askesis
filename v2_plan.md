# 🌙 Áskesis v2 — Overnight Build Plan

> **Como usar:** quando o build v1 estiver mergeado, abra uma sessão nova do
> Antigravity, anexe `claude.md`, `askesis_app_mockup.html`, `Prompt_BLAST.md`
> **e este arquivo**, e cole o bloco final ("PROMPT PARA COLAR"). Vá dormir.
>
> Branch alvo: `claude/audit-android-app-files-Y06Kn` (ou criar
> `claude/askesis-v2`).

---

## 0 · Resumo executivo do escopo v2

| # | Melhoria | Camada principal | Risco |
|---|---|---|---|
| 1 | Aba Focus rolável + presets customizáveis | UI | 🟢 baixo |
| 2 | Review com formato h/m/s e enriquecimento | UI + format | 🟢 baixo |
| 3 | Dashboard com 4 sub-views (Today / 7d / 30d / Lifetime) | UI + analytics | 🟡 médio |
| 4 | Sync em nuvem entre Androids do usuário | Auth + DB + reconciliação | 🔴 alto |
| 5 | Leitura de tempo de tela por app (UsageStats) + categorização | Android nativo | 🔴 alto |
| 6 | Notificação persistente com controles do timer | Foreground service | 🟡 médio |
| 7 | Overlay flutuante (pop-up) do timer | SYSTEM_ALERT_WINDOW | 🔴 alto |
| 8 | Widget de home screen (últimas 3 tags + Start) | Android Widget | 🟡 médio |

**Ordem de execução:** fácil → difícil, com escape hatches. Se um item de
risco alto falhar, registrar em `findings.md`, criar issue de follow-up, e
seguir adiante — o app v2 ainda entrega valor.

---

## 1 · Deltas obrigatórios em `claude.md` (aplicar ANTES do código)

### 1.1 Schemas novos / alterados

#### `Tag` — campo novo
```json
{
  "...campos existentes...": "...",
  "is_app_category": "bool (default false) — true se a tag está associada a apps"
}
```

#### `FocusSession` — campos novos
```json
{
  "...campos existentes...": "...",
  "source": "Enum: manual | app_usage (default manual)",
  "source_app_package": "String? (com.ankidroid.anki, etc — só se source=app_usage)",
  "synced_at": "DateTime? (null = não sincronizado)",
  "deleted_at": "DateTime? (soft delete para sync)"
}
```

#### `Task` — campos novos
```json
{
  "...campos existentes...": "...",
  "synced_at": "DateTime?",
  "deleted_at": "DateTime? (soft delete para sync)"
}
```

#### NOVO: `TimerPreset`
```json
{
  "id": "String (UUID)",
  "label": "String (ex: '45 min')",
  "duration_seconds": "int",
  "is_default": "bool",
  "sort_order": "int"
}
```
Seed: 15, 25, 50, 90 min (`is_default=true`).

#### NOVO: `AppPackage`
```json
{
  "package_name": "String (PK — ex: com.ankidroid.anki)",
  "display_name": "String (ex: AnkiDroid)",
  "icon_bytes": "Uint8List? (cache do ícone do app)",
  "tag_id": "String? (UUID — null = não categorizado)",
  "ignored": "bool (true = excluir das contagens)",
  "first_seen_at": "DateTime",
  "last_seen_at": "DateTime"
}
```

#### NOVO: `AppUsageSession`
```json
{
  "id": "String (UUID)",
  "package_name": "String",
  "started_at": "DateTime",
  "ended_at": "DateTime",
  "duration_seconds": "int",
  "tag_id": "String? (resolvido na captura ou depois)",
  "promoted_to_focus_session_id": "String? (UUID)",
  "synced_at": "DateTime?"
}
```
Capturadas via `UsageStatsManager`. **Threshold: duration_seconds >= 180**
(3 min) para serem persistidas.

#### NOVO: `UserAccount`
```json
{
  "uid": "String (Firebase UID)",
  "email": "String?",
  "display_name": "String?",
  "device_id": "String (Android ID hash)",
  "last_sync_at": "DateTime?",
  "sync_enabled": "bool"
}
```

#### NOVO: `SyncCursor` (uma linha por coleção)
```json
{
  "collection": "String (tasks | focus_sessions | tags | timer_presets | app_packages | app_usage_sessions)",
  "last_pulled_at": "DateTime",
  "last_pushed_at": "DateTime"
}
```

### 1.2 Novas invariantes

10. **App usage threshold:** `AppUsageSession` com `duration_seconds < 180`
    é descartada antes da persistência.
11. **Soft delete obrigatório para entidades sincronizáveis:** Task,
    FocusSession, Tag, TimerPreset, AppPackage. Nunca delete duro local até
    confirmação do push.
12. **Conflito de sync = last-write-wins por campo** usando
    `updated_at` (server timestamp). Soft delete prevalece sobre update.
13. **Tag pode ser de duas naturezas:** manual (atividades feitas
    conscientemente) ou categoria de app (`is_app_category=true`). O usuário
    pode promover uma tag manual a categoria de app a qualquer momento.
14. **Permissões opt-in:** `PACKAGE_USAGE_STATS` e `SYSTEM_ALERT_WINDOW`
    são features avançadas. O app deve funcionar 100% sem elas.
15. **Sync é opt-in:** sem login → 100% local (comportamento v1 preservado).

### 1.3 Tech stack v2 (adições)

| Camada | Tecnologia | Justificativa |
|---|---|---|
| Auth | **firebase_auth** + **google_sign_in** | Login com Google trivial em Android |
| Cloud DB | **cloud_firestore** | Sync offline-first nativo, regras por UID, free tier suficiente |
| Android nativo | **MethodChannel** custom + Kotlin | UsageStatsManager, overlay, widget |
| Widget | **home_widget** (Flutter) + RemoteViews (Kotlin) | Comunicação Flutter ↔ widget |
| Background job | **workmanager** | Pull de UsageStats periódico (15 min) |
| Picture-in-picture | API nativa Android (`enterPictureInPictureMode`) | Pop-up do timer |

### 1.4 Novas permissões em `AndroidManifest.xml`

```xml
<!-- v2 -->
<uses-permission android:name="android.permission.PACKAGE_USAGE_STATS" tools:ignore="ProtectedPermissions"/>
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>
<uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE"/>
<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.QUERY_ALL_PACKAGES" tools:ignore="QueryAllPackagesPermission"/>
```

Cada permissão tem fluxo dedicado de pedido com explicação prévia (rationale
screen). O usuário pode negar e usar o app sem aquela feature.

---

## 2 · Deltas no mockup (`askesis_app_mockup.html`)

- Aba Focus: container externo com `overflow-y: auto`, altura fixa do
  conteúdo restante (descontando topbar e navbar).
- Aba Focus: chip "+ Adicionar" ao final dos presets de duração → abre
  bottom sheet com seletor minutes/seconds.
- Aba Review: itens com duração > 60 min mostram "1h 15min 30s" (formato
  curto h/min/s, omitindo unidades zeradas, ex.: "45 min", "1h 5min",
  "2h 0min 30s").
- Aba Review: novo bloco no topo "Resumo do período" com totais por tag.
- Aba Dashboard: 4 sub-views renomeadas — Today | 7 days | 30 days |
  Lifetime. Cada uma com gráficos próprios (ver §5).
- Topbar: novo ícone `ti ti-cloud` ao lado do `ti ti-hourglass` → conta /
  status de sync.

---

## 3 · Pre-flight checklist (FASE 0)

```
[ ] git checkout claude/askesis-v2  (criar a partir de main)
[ ] flutter pub upgrade --major-versions
[ ] flutter pub add firebase_core firebase_auth cloud_firestore \
        google_sign_in workmanager home_widget url_launcher \
        flutter_app_usage device_info_plus
[ ] flutter pub add --dev mockito build_runner
[ ] Adicionar plugins Android nativos:
      android/app/src/main/kotlin/com/askesis/askesis/
        - UsageStatsBridge.kt
        - OverlayService.kt
        - AskesisWidgetProvider.kt
[ ] Atualizar AndroidManifest.xml com permissões da §1.4
[ ] Criar projeto no Firebase Console, baixar google-services.json,
    colocar em android/app/
[ ] Configurar build.gradle (kotlin DSL): apply plugin com.google.gms...
[ ] Atualizar claude.md com schemas e invariantes da §1
[ ] Commit: "chore(v2): bootstrap firebase + native plugins"
```

> **Se Firebase não puder ser configurado autonomamente** (precisa de login
> Google do usuário no console), registrar em `findings.md` com instruções
> manuais e seguir com tudo que não depende de Firebase. Outras fases
> permanecem funcionais.

---

## 4 · PHASE A · Focus UX (🟢 baixo risco — começar por aqui)

### A.1 Scroll na aba Focus
- Envolver o conteúdo da aba em `SingleChildScrollView` com
  `physics: BouncingScrollPhysics()`.
- Garantir que o seletor de tags use `Wrap` (não `Row` rígido) — quebra
  para múltiplas linhas quando há muitas tags.
- Testar com 30+ tags no seed de desenvolvimento.

### A.2 Presets customizáveis
- Nova entidade `TimerPreset` (ver §1.1).
- Repositório `TimerPresetRepository`.
- UI: chips horizontais com presets ordenados por `sort_order`. Chip final
  "+ Adicionar" abre bottom sheet.
- Bottom sheet: 2 sliders (horas 0–4, minutos 0–59) + preview do tempo +
  campo "Apelido (opcional)". Salvar adiciona ao final.
- Long-press em chip de preset não-default → menu: Editar / Remover.
- Migração: no primeiro launch v2, popular `TimerPreset` com 15/25/50/90
  se a tabela estiver vazia.

### A.3 Testes
- Widget test: aba Focus rolável com 50 tags renderiza todos os controles.
- Unit test: `TimerPresetRepository` (CRUD + ordenação).

**Commit:** `feat(focus): scrollable layout + custom timer presets`

---

## 5 · PHASE B · Review v2 (🟢 baixo)

### B.1 Formatação de duração
Criar `lib/shared/format/duration_format.dart`:
```dart
String formatHms(int seconds) {
  // < 60 → "Xs"
  // < 3600 → "Xmin" ou "Xmin Ys"
  // >= 3600 → "Xh Ymin" ou "Xh Ymin Zs"
  // Omitir unidades zeradas exceto a maior.
}
```
Substituir todas as renderizações de duração no app (Review, Dashboard,
Focus history) por essa função. Testes unitários: 13 casos cobrindo
limiares (59s, 60s, 3599s, 3600s, 3661s, 5400s, etc.).

### B.2 Enriquecimento visual do Review
- Cabeçalho com "Resumo do período" (default: últimos 30 dias): total de
  foco, total de sessões, tag mais praticada.
- Filtro de intervalo de data via `showDateRangePicker`.
- Filtro de tag com chips multi-seleção.
- Filtro extra: `Source` (Manual / App Usage / Tudo) — relevante após
  Phase E.
- Paginação infinita (50 por vez).
- Cards com mais detalhe: ícone + título + tag colorida + duração
  formatada + horário "Hoje · 14:25" / "Ontem" / "12 mai".

**Commit:** `feat(review): h/m/s formatting + period summary + filters`

---

## 6 · PHASE C · Dashboard rebuild (🟡 médio)

### C.1 Estrutura das 4 sub-views

| View | Período | Gráficos principais | Cards |
|---|---|---|---|
| Today | dia atual (00:00–23:59) | Bar chart por hora (24 barras) + donut por tag | Foco hoje, Sessões, Tarefas feitas, Maior streak ininterrupto |
| 7 days | últimos 7 dias | Bar chart diário + line chart cumulativo | Total, Média/dia, Dia mais produtivo, Sessões/dia média |
| 30 days | últimos 30 dias | Heatmap dia×hora (30×24) + stacked bar diário por tag | Total, Média, Tendência (vs 30d anteriores), Top 3 tags |
| Lifetime | desde o início | Heatmap GitHub anual (~52×7) + line chart cumulativo mensal | Foco total, Dias ativos, Consistência %, Maior streak |

### C.2 Implementação
- `lib/presentation/dashboard/views/today_view.dart`, `seven_days_view.dart`,
  `thirty_days_view.dart`, `lifetime_view.dart`.
- `lib/presentation/dashboard/widgets/`: `hour_bar_chart.dart`,
  `daily_bar_chart.dart`, `cumulative_line_chart.dart`, `tag_donut.dart`,
  `heatmap_year.dart`, `heatmap_day_hour.dart`, `streak_card.dart`.
- Queries Isar centralizadas em `lib/data/repositories/analytics_repository.dart`
  (read-only). Sem cache persistido (invariante #4).
- Comparativos de tendência: dois períodos espelhados, mostrar delta % com
  seta ↑/↓.

### C.3 Testes
- Cada query do `AnalyticsRepository` com seed determinístico.
- Smoke widget test por sub-view.

**Commit:** `feat(dashboard): today/7d/30d/lifetime views with charts`

---

## 7 · PHASE D · Cloud sync (🔴 alto)

### D.1 Modelo de sync
- **Estratégia:** offline-first com Firestore (que já tem cache local). O
  Isar continua sendo source-of-truth local. Sync é unidirecional
  bidirecional periódico:
  1. **Pull:** Firestore → Isar, comparando `updated_at`.
  2. **Push:** linhas com `synced_at < updated_at` → Firestore.
  3. **Resolução de conflito:** last-write-wins por campo via
     server timestamp.
  4. **Soft delete:** `deleted_at != null` propaga e mascara em queries
     locais.

### D.2 Esquema Firestore
```
users/{uid}/
  meta              (document) — display_name, devices[]
  tags/{tagId}
  tasks/{taskId}
  focus_sessions/{sessionId}
  timer_presets/{presetId}
  app_packages/{packageName}
  app_usage_sessions/{sessionId}
```

Regras de segurança (`firestore.rules`):
```
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {
    match /users/{uid}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
  }
}
```

### D.3 Auth flow
- Tela de "Conta" acessível pelo ícone `ti ti-cloud` no topbar.
- Estado: deslogado → botão "Entrar com Google" (google_sign_in).
- Logado → email, dispositivos vinculados, "Sincronizar agora", "Sair".
- Primeira sincronização após login: merge — registros locais sobem com
  `device_id` próprio; conflitos resolvidos por timestamp.

### D.4 SyncService
- `lib/services/sync_service.dart`.
- Roda em background a cada 15 min via `workmanager` E quando o app volta
  ao foreground.
- Streams Firestore com cache offline (Firestore SDK já cuida).
- Estado exposto via Riverpod `syncStatusProvider`:
  `idle | syncing | error | offline`.

### D.5 Tela "Conta & Sync"
- Lista de dispositivos com último sync.
- Botão "Forçar sync".
- Toggle "Sync automático".
- Exportar/Importar JSON (mantém backup local manual mesmo com sync ativo).

### D.6 Testes
- Mock Firestore via `fake_cloud_firestore`.
- Casos de conflito: mesma task editada em 2 dispositivos com timestamps
  diferentes.
- Soft delete propaga corretamente.
- Login/logout não destrói dados locais.

**Commits:**
- `feat(sync): firebase auth + google sign-in`
- `feat(sync): firestore schema + push/pull engine`
- `feat(sync): account screen + manual force sync`

**Escape hatch:** se Firebase não puder ser provisionado autonomamente,
deixar o código pronto, gating na ausência de `google-services.json`, e
documentar em `findings.md` os passos manuais.

---

## 8 · PHASE E · App Usage tracking (🔴 alto — Android nativo)

### E.1 Bridge Kotlin
Criar `android/app/src/main/kotlin/com/askesis/askesis/UsageStatsBridge.kt`:
- `MethodChannel("askesis/usage_stats")`.
- Métodos:
  - `hasPermission()` → bool.
  - `requestPermission()` → abre `Settings.ACTION_USAGE_ACCESS_SETTINGS`.
  - `queryUsage(startMs: Long, endMs: Long)` → `List<Map>` com
    `package_name`, `start_ms`, `end_ms`, `total_ms`.
  - `getAppInfo(pkg)` → `display_name` + `icon_bytes` (Base64).
  - `getInstalledApps()` → lista de apps instalados (filtrar
    system apps via `ApplicationInfo.FLAG_SYSTEM`).

Use `UsageStatsManager.queryEvents` para reconstruir sessões individuais
(MOVE_TO_FOREGROUND / MOVE_TO_BACKGROUND), **não** `queryUsageStats` que
agrega por dia.

### E.2 Captura periódica
- `workmanager` task `app_usage_pull` rodando a cada 15 min:
  1. Lê último cursor (`SharedPreferences key: last_usage_pull_ms`).
  2. Chama `queryUsage(last_cursor, now)`.
  3. Filtra sessões com `total_ms >= 180_000` (3 min).
  4. Resolve `tag_id` via `AppPackage.tag_id` (se mapeado).
  5. Persiste `AppUsageSession` em Isar.
  6. Cursor avança para `now`.

### E.3 UI
- **Aba Focus / Dashboard:** sessões `source=app_usage` aparecem
  misturadas com manuais.
- **Nova tela "Apps" (acessível via Dashboard → Lifetime → "Apps"):**
  Lista de `AppPackage` ordenada por tempo total (30 dias).
  Cada item: ícone, nome, tempo total formatado, badge da tag
  (ou "Não categorizado" amarelo).
  Tap → bottom sheet de atribuição: lista de tags + criar nova + opção
  "Ignorar este app" (`ignored=true`).
- **Banner de descoberta:** quando há ≥1 app não categorizado com tempo
  significativo (>30 min na semana), mostrar banner no topo da Dashboard:
  "Você usou *AnkiDroid* por 1h 15min — categorizar?".

### E.4 Promoção a FocusSession
- Quando uma `AppUsageSession` tem `tag_id` resolvido, criar
  automaticamente uma `FocusSession` espelho com `source=app_usage`,
  `source_app_package=...`, `completed=true`, vinculada via
  `promoted_to_focus_session_id`.
- Categorização retroativa: ao atribuir tag a um app, fazer backfill das
  últimas 30 dias de `AppUsageSession` daquele app.

### E.5 Testes
- Mock do bridge: simular eventos foreground/background.
- Filtragem ≥3min funciona.
- Backfill retroativo cria sessões corretas.

**Commits:**
- `feat(android): usage stats bridge (kotlin)`
- `feat(usage): periodic capture + app catalog screen`
- `feat(usage): app→tag mapping + retroactive backfill`

**Escape hatch:** se o usuário negar `PACKAGE_USAGE_STATS`, app continua
funcionando sem a feature; mostrar card explicativo em Settings → "Ativar
tempo de tela".

---

## 9 · PHASE F · Always-on controls (🟡–🔴)

### F.1 Notificação persistente com controles (🟡 médio)
- Extensão de `flutter_foreground_task` com ações customizadas:
  - **Pausar / Retomar** (alterna).
  - **Parar** (abre app para confirmar cancelamento).
  - **+ 5 min** (estende countdown — só no modo timer).
- Layout: progress bar, tempo restante grande, tag, 3 botões.
- Atualiza a cada segundo enquanto rodando.
- Tap no corpo da notificação abre a aba Focus.

### F.2 Overlay flutuante / pop-up do timer (🔴 alto)
- Permissão `SYSTEM_ALERT_WINDOW` (intent
  `Settings.ACTION_MANAGE_OVERLAY_PERMISSION`).
- `OverlayService.kt` (Android Service) que cria `WindowManager` view com
  layout circular minúsculo (~80dp): timer + cor da tag.
- Drag para reposicionar; tap expande para mini-controles
  (pause/cancel/expand).
- Toggle no Focus: "📱 Mostrar enquanto fora do app".
- **Alternativa fallback se overlay falhar:** Picture-in-Picture nativo do
  Android (`enterPictureInPictureMode`) na atividade do Focus. Menos
  flexível mas não requer permissão especial.

### F.3 Widget de home screen (🟡 médio)
- `AskesisWidgetProvider.kt` (RemoteViews).
- Layout: 4×2 (médio):
  - Linha 1: 3 chips das últimas 3 tags usadas (tap inicia
    cronômetro com aquela tag).
  - Linha 2: 2 botões grandes — "▶ Timer 25min" (último preset usado),
    "⏱ Cronômetro".
- Atualizado via `home_widget` package toda vez que uma sessão termina.
- Tap em chip/botão dispara deep link `askesis://start?type=timer&tag=...`
  → `go_router` redireciona e auto-inicia a sessão.
- Widget de tamanho 2×2 (pequeno): apenas botão "▶ Start" com a última
  configuração.

### F.4 Testes
- Smoke: notificação com 3 ações renderiza.
- Widget: deep links abrem o app na rota correta.
- Overlay: não bloqueia gestos críticos do sistema.

**Commits:**
- `feat(notif): persistent controls (pause/stop/+5min)`
- `feat(android): floating overlay service`
- `feat(android): home screen widget + deep links`

**Escape hatch overlay:** se a permissão for negada, oferecer PiP
automaticamente. Se PiP também falhar, manter apenas a notificação.

---

## 10 · PHASE G · QA + Build (todas)

```
[ ] flutter analyze (zero warnings)
[ ] flutter test (verde, cobertura ≥80% nas novas camadas)
[ ] flutter test integration_test/ (ver casos abaixo)
[ ] flutter build apk --release --split-per-abi
[ ] Validar tamanho dos APKs (<30MB cada)
[ ] Atualizar progress.md com:
    - O que ficou pronto vs o que precisou de escape hatch
    - Tamanhos finais dos APKs
    - Cobertura
    - Lista de débitos técnicos
[ ] Atualizar findings.md com TODAS as decisões autônomas
[ ] Commit final: "build: askesis v2.0.0"
[ ] git push -u origin claude/askesis-v2
```

### Casos de integration test mínimos
1. Login Google → criar task → logout → login em "outro device" (state
   resetado) → task aparece.
2. Iniciar timer 25min → minimizar → notificação mostra contagem →
   pausar pela notificação → retomar → completar → aparece em Review.
3. Sem permissão de UsageStats → tela "Apps" mostra CTA para conceder.
4. Com UsageStats: simular sessão >3min de pacote conhecido → categorizar
   → vira FocusSession.

---

## 11 · Regras de comportamento durante a execução

- **Não pergunte.** Em qualquer ambiguidade, escolha a opção mais alinhada
  a `claude.md` e registre em `findings.md` com timestamp e justificativa.
- **Self-annealing.** Test/build falha → analise stack trace → patch →
  retest → atualize SOP relevante em `architecture/`.
- **Escape hatches são aceitáveis** para itens 🔴: registre o que ficou
  pendente em `findings.md`, crie issue stub, e siga.
- **Visual:** divergências mockup × claude.md → mockup vence para visual,
  claude.md vence para regra de negócio.
- **Permissões:** sempre exibir rationale screen antes do prompt do
  sistema. Usuário pode negar — o app NÃO pode crashar nem ficar inerte.
- **Sync:** dados locais nunca são apagados em logout. Apenas o cursor
  de sync é resetado.
- **Tags `is_app_category`:** podem coexistir com tags manuais — não são
  exclusivas.
- **Nunca pule testes.** Nunca deixe TODOs no código. Fora-de-escopo →
  `findings.md`.

---

## 12 · Entrega esperada ao acordar

- Branch `claude/askesis-v2` commitada fase a fase e pushed.
- APKs em `build/app/outputs/flutter-apk/app-*-release.apk`.
- `progress.md` com checklist completo, status de cada feature
  (✅ pronto / ⚠️ escape hatch / ❌ bloqueado), tamanhos, cobertura.
- `findings.md` com decisões autônomas.
- `claude.md` atualizado com schemas v2 e invariantes 10–15.
- `architecture/sop_*.md` atualizados (incluir
  `sop_sync.md`, `sop_app_usage.md`, `sop_widget.md`).

---

## 13 · PROMPT PARA COLAR NO ANTIGRAVITY

```
Você é o System Pilot do projeto Áskesis e vai executar a evolução v2 sem
fazer perguntas — eu estarei dormindo. Trabalhe autonomamente até o fim.

ANEXOS (precedência nessa ordem):
1. claude.md            → CONSTITUIÇÃO (lei). Aplique os deltas da §1 do
                          v2_plan.md ANTES de qualquer código.
2. v2_plan.md           → ROADMAP v2 (este plano). É a sua fonte de
                          ordem de execução, escape hatches e critérios
                          de aceitação.
3. askesis_app_mockup.html → contrato visual base. Aplique os deltas
                          da §2 do v2_plan.md.
4. Prompt_BLAST.md      → metodologia (já adaptada para Flutter).

ESCOPO v2 (8 melhorias):
  A. Aba Focus rolável + presets customizáveis de duração
  B. Review com formato h/min/s + filtros enriquecidos
  C. Dashboard com 4 sub-views: Today / 7 days / 30 days / Lifetime
  D. Sync em nuvem entre Androids (Firebase Auth + Firestore)
  E. Leitura de tempo de tela por app (UsageStats) + categorização
  F. Notificação persistente com controles + overlay flutuante + widget
  G. QA + build do APK release

REGRAS GLOBAIS:
- Pré-flight (FASE 0 do v2_plan): bootstrap firebase, plugins nativos,
  atualizar manifest, atualizar claude.md com schemas v2.
- Execute as fases na ordem A → B → C → D → E → F → G.
- Cada fase termina com: testes verdes + commit semântico + atualização
  de progress.md.
- Itens 🔴 alto risco (D, E, F.2) podem usar escape hatches descritos no
  v2_plan. Registre em findings.md.
- Após cada commit, faça push para `claude/askesis-v2`.
- Em ambiguidade: claude.md vence regra de negócio, mockup vence visual,
  v2_plan.md vence ordem/escopo. Última instância: decida e registre em
  findings.md.
- Cobertura mínima: FocusSessionService 100%, SyncService 90%, demais
  serviços/repos ≥80%, smoke widget tests para cada nova tela.
- Permissões opt-in: app deve funcionar 100% mesmo sem
  PACKAGE_USAGE_STATS ou SYSTEM_ALERT_WINDOW.
- Sync é opt-in: deslogado = comportamento v1 preservado.

CHECKLIST FINAL ESPERADO AO ACORDAR:
  [ ] Branch claude/askesis-v2 commitada e pushed
  [ ] APKs split-per-abi gerados
  [ ] progress.md com status de cada feature (✅/⚠️/❌)
  [ ] findings.md com todas as decisões autônomas
  [ ] claude.md com schemas v2 e invariantes 10–15
  [ ] architecture/sop_sync.md, sop_app_usage.md, sop_widget.md
  [ ] flutter analyze = 0 warnings
  [ ] flutter test = verde

Comece pelo Pré-flight (FASE 0 do v2_plan.md). Boa noite.
```

---

## 14 · Anexo: arquivos novos esperados ao fim do v2

```
askesis/
├── v2_plan.md                                    ← este arquivo
├── claude.md                                     ← atualizado (schemas v2)
├── firestore.rules                               ← novo
├── android/
│   └── app/
│       ├── google-services.json                  ← novo (do Firebase Console)
│       └── src/main/
│           ├── AndroidManifest.xml               ← permissões v2
│           ├── kotlin/com/askesis/askesis/
│           │   ├── MainActivity.kt
│           │   ├── UsageStatsBridge.kt           ← novo
│           │   ├── OverlayService.kt             ← novo
│           │   └── AskesisWidgetProvider.kt      ← novo
│           └── res/
│               ├── layout/widget_askesis_4x2.xml ← novo
│               ├── layout/widget_askesis_2x2.xml ← novo
│               └── xml/widget_info.xml           ← novo
├── lib/
│   ├── data/
│   │   ├── isar/
│   │   │   ├── timer_preset.dart                 ← novo
│   │   │   ├── app_package.dart                  ← novo
│   │   │   ├── app_usage_session.dart            ← novo
│   │   │   ├── user_account.dart                 ← novo
│   │   │   └── sync_cursor.dart                  ← novo
│   │   └── repositories/
│   │       ├── timer_preset_repository.dart      ← novo
│   │       ├── app_package_repository.dart       ← novo
│   │       ├── app_usage_repository.dart         ← novo
│   │       └── analytics_repository.dart         ← novo
│   ├── services/
│   │   ├── sync_service.dart                     ← novo
│   │   ├── usage_stats_service.dart              ← novo
│   │   ├── overlay_service.dart                  ← novo
│   │   └── widget_bridge_service.dart            ← novo
│   ├── presentation/
│   │   ├── account/                              ← nova feature
│   │   ├── apps/                                 ← nova feature
│   │   ├── focus/
│   │   │   └── widgets/preset_picker_sheet.dart  ← novo
│   │   └── dashboard/
│   │       └── views/                            ← novo (4 sub-views)
│   └── shared/
│       └── format/duration_format.dart           ← novo
└── architecture/
    ├── sop_sync.md                               ← novo
    ├── sop_app_usage.md                          ← novo
    ├── sop_overlay.md                            ← novo
    └── sop_widget.md                             ← novo
```
