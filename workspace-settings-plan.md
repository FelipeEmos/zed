# Plano: Workspace Profiles - Theme Override por Path

## 🎯 Objetivo

Permitir que cada workspace (janela) do Zed tenha configurações próprias baseadas no **path do projeto**, sem modificar o sistema de profiles existente.

**Caso de uso:**

```json
{
  "workspace_profiles": {
    "solve_world_hunger": {
      "path": "~/Development/solve-world-hunger",
      "theme": "Nordfox - blurred"
    },
    "personal_blog": {
      "path": "~/Projects/blog",
      "theme": "Solarized Light",
      "ui_font_size": 16
    }
  },
  "profiles": {
    "uirapuru": {
      "theme": "Tokyo Night Storm"
    }
  }
}
```

**Resultado:**

- Abrir `~/Development/solve-world-hunger` → tema "Nordfox - blurred"
- Abrir `~/Projects/blog` → tema "Solarized Light" + font size 16
- Abrir qualquer outro projeto → usa profile "uirapuru" (ou settings global)

---

## 🔍 Por que essa abordagem é superior?

### Comparação com alternativas:

| Abordagem                                | Pros                                           | Cons                                           |
| ---------------------------------------- | ---------------------------------------------- | ---------------------------------------------- |
| **Modificar profile-settings**           | Usa infraestrutura existente                   | Pisaria em discussões internas, mudança grande |
| **Workspace no DB**                      | Persiste automaticamente                       | Path não é estável, difícil de compartilhar    |
| **ProjectSettings (.zed/settings.json)** | Por projeto                                    | Comita no git (rejeitado pela comunidade)      |
| **✅ workspace_profiles**                | Não mexe em profiles, familiar, compartilhável | Precisa resolver path matching                 |

### Vantagens específicas:

1. **Não toca no sistema de profiles** - Zero risco de conflitos com decisões de design existentes
2. **Path-based matching** - Intuitivo: "esse path sempre usa essas configs"
3. **No settings.json** - Usuários já conhecem, fácil de editar/compartilhar
4. **Ortogonal** - Feature independente que complementa profiles
5. **Menos código** - Não precisa DB, UI de gerenciamento complexa, etc.

---

## 🏗️ Arquitetura Atual (Recap)

### Hierarchy de Settings hoje:

```
Local (.zed/settings.json no path específico)
  ↓ fallback
Global (user settings)
  ↓ merge
Global Profile (se ativo via ActiveSettingsProfileName)
  ↓ fallback
Default
```

### Por que Theme ignora essa hierarchy?

```rust
// theme/src/theme.rs:452
impl GlobalTheme {
    fn configured_theme(cx: &mut App) -> Arc<Theme> {
        let theme_settings = ThemeSettings::get_global(cx);  // ← SEMPRE global!
        // ...
    }
}
```

**Problema:** Mesmo que local settings tenha `"theme": "Dark"`, é ignorado porque:

1. `GlobalTheme` chama `get_global()` ao invés de `get(location)`
2. Tema é aplicado globalmente via `cx.refresh_windows()`
3. Não há conceito de "tema por janela"

---

## ✅ Solução: workspace_profiles

### Nova Hierarchy:

```
Local (.zed/settings.json no path específico)
  ↓ fallback
Workspace Profile (matched por path do workspace)  ← NOVO!
  ↓ fallback
Global (user settings)
  ↓ merge
Global Profile (se ativo)
  ↓ fallback
Default
```

### Estrutura no settings.json:

```rust
// settings/src/settings_content.rs

#[derive(Clone, Debug, Deserialize, JsonSchema)]
pub struct UserSettingsContent {
    // ... campos existentes ...

    #[serde(default)]
    pub profiles: HashMap<String, SettingsContent>,

    #[serde(default)]
    pub workspace_profiles: HashMap<String, WorkspaceProfileSettingsContent>,  // ← NOVO!
}

#[derive(Clone, Debug, Deserialize, JsonSchema)]
pub struct WorkspaceProfileSettingsContent {
    /// Path do workspace (suporta ~ e globs)
    ///
    /// Examples:
    /// - "~/Development/my-project"
    /// - "/home/user/work/*"
    /// - "~/Projects/client-*"
    pub path: String,

    /// Settings to apply when this workspace is active
    #[serde(flatten)]
    pub settings: SettingsContent,
}
```

---

## 📋 Implementação Detalhada

### Parte 1: Path Matching - Resolver qual workspace_profile usar

**Por quê?** Precisamos descobrir qual workspace_profile aplicar baseado nos paths abertos.

```rust
// workspace/src/workspace.rs

impl Workspace {
    /// Retorna o workspace_profile que matcha com os paths deste workspace
    fn matched_workspace_profile(&self, cx: &App) -> Option<String> {
        let user_settings = cx.global::<SettingsStore>().user_settings()?;
        let workspace_profiles = &user_settings.workspace_profiles;

        // Pega todos os root paths deste workspace
        let workspace_paths = self.root_paths(cx);

        // Procura workspace_profile que matcha
        for (profile_name, profile) in workspace_profiles {
            let pattern = shellexpand::tilde(&profile.path);

            // Tenta match exato primeiro
            for workspace_path in &workspace_paths {
                if workspace_path.as_path() == Path::new(pattern.as_ref()) {
                    return Some(profile_name.clone());
                }
            }

            // Depois tenta glob matching
            if let Ok(pattern) = glob::Pattern::new(&pattern) {
                for workspace_path in &workspace_paths {
                    if pattern.matches_path(workspace_path) {
                        return Some(profile_name.clone());
                    }
                }
            }
        }

        None
    }
}
```

**Edge cases:**

- Multiple paths: usa o primeiro que matchar
- Glob patterns: suporta `*` e `**`
- `~` expansion: resolve home dir
- Relative paths: resolve baseado no cwd

---

### Parte 2: SettingsStore - Resolver settings com workspace_profile

**Por quê?** Precisamos modificar a hierarchy de merge para incluir workspace_profile.

```rust
// settings/src/settings_store.rs

pub struct SettingsStore {
    // ... campos existentes ...

    // Cache: WorkspaceId -> nome do workspace_profile ativo
    active_workspace_profiles: HashMap<WorkspaceId, Option<String>>,
}

impl SettingsStore {
    /// Atualiza qual workspace_profile está ativo para um workspace
    pub fn set_active_workspace_profile(
        &mut self,
        workspace_id: WorkspaceId,
        profile_name: Option<String>,
        cx: &mut App,
    ) -> Result<()> {
        self.active_workspace_profiles.insert(workspace_id, profile_name);
        self.recompute_values(Some(workspace_id), cx)?;
        Ok(())
    }

    /// Pega settings considerando workspace_profile
    pub fn get_for_workspace<T: Settings>(
        &self,
        workspace_id: WorkspaceId,
    ) -> &T {
        // Usa SettingValue com workspace context
        self.setting_values
            .get(&TypeId::of::<T>())
            .unwrap()
            .value_for_workspace(Some(workspace_id))
            .downcast_ref::<T>()
            .unwrap()
    }
}
```

**Modificar `recompute_values`:**

```rust
fn recompute_values(
    &mut self,
    workspace_id: Option<WorkspaceId>,
    cx: &mut App,
) -> Result<(), InvalidSettingsError> {

    // 1. Recompute global (se nenhum workspace específico)
    if workspace_id.is_none() {
        let mut merged = self.default_settings.as_ref().clone();
        merged.merge_from_option(self.extension_settings.as_deref());
        merged.merge_from_option(self.global_settings.as_deref());

        if let Some(user_settings) = self.user_settings.as_ref() {
            merged.merge_from(&user_settings.content);
            merged.merge_from_option(user_settings.for_release_channel());
            merged.merge_from_option(user_settings.for_os());
            merged.merge_from_option(user_settings.for_profile(cx));  // Global profile
        }

        merged.merge_from_option(self.server_settings.as_deref());
        self.merged_settings = Rc::new(merged);

        for setting_value in self.setting_values.values_mut() {
            let value = setting_value.from_settings(&self.merged_settings);
            setting_value.set_global_value(value);
        }
    }

    // 2. Recompute workspace (NOVO!)
    if let Some(ws_id) = workspace_id {
        let mut merged = self.merged_settings.as_ref().clone();

        // Merge workspace_profile se ativo
        if let Some(profile_name) = self.active_workspace_profiles.get(&ws_id).and_then(|p| p.as_ref()) {
            if let Some(user_settings) = self.user_settings.as_ref() {
                if let Some(ws_profile) = user_settings.workspace_profiles.get(profile_name) {
                    merged.merge_from(&ws_profile.settings);
                }
            }
        }

        for setting_value in self.setting_values.values_mut() {
            let value = setting_value.from_settings(&merged);
            setting_value.set_workspace_value(ws_id, value);
        }
    }

    // 3. Recompute local (continua igual)
    for ((root_id, directory_path), local_settings) in &self.local_settings {
        // ... código existente ...
    }

    Ok(())
}
```

---

### Parte 3: SettingValue - Adicionar workspace layer

**Por quê?** Seguir o padrão de `local_values` mas para workspaces.

```rust
// settings/src/settings_store.rs

struct SettingValue<T> {
    global_value: Option<T>,
    workspace_values: HashMap<WorkspaceId, T>,  // ← NOVO!
    local_values: Vec<(WorktreeId, Arc<RelPath>, T)>,
}

impl<T: Clone + Send + Sync + 'static> SettingValue<T> {
    fn value_for_workspace(&self, workspace_id: Option<WorkspaceId>) -> &dyn Any {
        // 1. Workspace-specific (se existir)
        if let Some(ws_id) = workspace_id {
            if let Some(value) = self.workspace_values.get(&ws_id) {
                return value;
            }
        }

        // 2. Global (fallback)
        self.global_value.as_ref().unwrap()
    }

    fn value_for_location(
        &self,
        workspace_id: Option<WorkspaceId>,
        location: Option<SettingsLocation>,
    ) -> &dyn Any {
        // 1. Local (mais específico)
        if let Some(loc) = location {
            for (wt_id, path, value) in self.local_values.iter().rev() {
                if loc.worktree_id == *wt_id && loc.path.starts_with(path) {
                    return value;
                }
            }
        }

        // 2. Workspace (novo!)
        if let Some(ws_id) = workspace_id {
            if let Some(value) = self.workspace_values.get(&ws_id) {
                return value;
            }
        }

        // 3. Global (fallback)
        self.global_value.as_ref().unwrap()
    }

    fn set_workspace_value(&mut self, workspace_id: WorkspaceId, value: Box<dyn Any>) {
        self.workspace_values.insert(workspace_id, *value.downcast().unwrap());
    }
}
```

---

### Parte 4: GlobalTheme - Usar workspace_profile

**Por quê?** Tema precisa respeitar workspace_profile ao invés de sempre usar global.

```rust
// theme/src/theme.rs

pub struct GlobalTheme {
    default_theme: Arc<Theme>,
    workspace_themes: HashMap<WorkspaceId, Arc<Theme>>,  // ← NOVO!
    default_icon_theme: Arc<IconTheme>,
    workspace_icon_themes: HashMap<WorkspaceId, Arc<IconTheme>>,
}

impl GlobalTheme {
    /// Configura tema considerando workspace_profile
    fn configured_theme_for_workspace(
        workspace_id: Option<WorkspaceId>,
        cx: &mut App,
    ) -> Arc<Theme> {
        let themes = ThemeRegistry::default_global(cx);

        // ← MUDANÇA CRÍTICA: pegar settings do workspace
        let theme_settings = if let Some(ws_id) = workspace_id {
            cx.global::<SettingsStore>()
                .get_for_workspace::<ThemeSettings>(ws_id)
        } else {
            ThemeSettings::get_global(cx)
        };

        let system_appearance = SystemAppearance::global(cx);
        let theme_name = theme_settings.theme.name(*system_appearance);

        let theme = match themes.get(&theme_name.0) {
            Ok(theme) => theme,
            Err(err) => {
                if themes.extensions_loaded() {
                    log::error!("{err}");
                }
                themes
                    .get(default_theme(*system_appearance))
                    .unwrap_or_else(|_| themes.get(DEFAULT_DARK_THEME).unwrap())
            }
        };

        theme_settings.apply_theme_overrides(theme)
    }

    /// Recarrega tema de workspace específico
    pub fn reload_theme_for_workspace(
        workspace_id: Option<WorkspaceId>,
        cx: &mut App,
    ) {
        let theme = Self::configured_theme_for_workspace(workspace_id, cx);

        cx.update_global::<Self, _>(|this, _| {
            if let Some(ws_id) = workspace_id {
                this.workspace_themes.insert(ws_id, theme);
            } else {
                this.default_theme = theme;
            }
        });

        // ← IMPORTANTE: só refresh janelas do workspace
        if let Some(ws_id) = workspace_id {
            refresh_workspace_windows(ws_id, cx);
        } else {
            cx.refresh_windows();
        }
    }

    pub fn theme_for_workspace(
        workspace_id: Option<WorkspaceId>,
        cx: &App,
    ) -> &Arc<Theme> {
        let global = cx.global::<Self>();
        workspace_id
            .and_then(|id| global.workspace_themes.get(&id))
            .unwrap_or(&global.default_theme)
    }
}
```

---

### Parte 5: Workspace - Ativar workspace_profile ao abrir

**Por quê?** Quando workspace abre, precisa descobrir e ativar seu workspace_profile.

```rust
// workspace/src/workspace.rs

impl Workspace {
    pub fn new(
        workspace_id: Option<WorkspaceId>,
        project: Entity<Project>,
        app_state: Arc<AppState>,
        window: &mut Window,
        cx: &mut Context<Self>,
    ) -> Self {
        // ... código existente ...

        // ← NOVO: Ativar workspace_profile se houver match
        if let Some(ws_id) = workspace_id {
            let matched_profile = workspace.matched_workspace_profile(cx);

            SettingsStore::update_global(cx, |store, cx| {
                store
                    .set_active_workspace_profile(ws_id, matched_profile, cx)
                    .log_err();
            });

            // Reload tema deste workspace
            GlobalTheme::reload_theme_for_workspace(Some(ws_id), cx);
        }

        workspace
    }
}
```

**Observer para mudanças no settings.json:**

```rust
// Quando user settings muda, recomputar workspace_profiles ativos
cx.observe_global::<SettingsStore>(move |cx| {
    // Para cada workspace aberto, re-check se workspace_profile mudou
    for workspace in app_state.workspace_store.read(cx).workspaces() {
        workspace.update(cx, |workspace, cx| {
            let ws_id = workspace.database_id();
            let matched = workspace.matched_workspace_profile(cx);

            SettingsStore::update_global(cx, |store, cx| {
                let current = store.active_workspace_profiles.get(&ws_id);
                if current != Some(&matched) {
                    store.set_active_workspace_profile(ws_id, matched, cx).log_err();
                    GlobalTheme::reload_theme_for_workspace(Some(ws_id), cx);
                }
            });
        });
    }
}).detach();
```

---

### Parte 6: Window precisa conhecer WorkspaceId

**Por quê?** `window.theme()` precisa saber qual workspace para pegar tema correto.

**Opção: HashMap Global (menos invasivo)**

```rust
// workspace/src/workspace.rs

struct WindowWorkspaceMap(HashMap<WindowId, WorkspaceId>);
impl Global for WindowWorkspaceMap {}

impl Workspace {
    pub fn new(..., window: &mut Window, cx: &mut Context<Self>) -> Self {
        // Registrar workspace para essa janela
        if let Some(ws_id) = workspace_id {
            let window_id = cx.window_id();
            cx.update_global::<WindowWorkspaceMap, _>(|map, _| {
                map.0.insert(window_id, ws_id);
            });
        }

        // ... resto do código ...
    }
}

// Cleanup ao fechar
impl Drop for Workspace {
    fn drop(&mut self) {
        // Remove do map
        // (precisa access to cx, então talvez no workspace.closed event)
    }
}
```

**Usar no ActiveTheme:**

```rust
// theme/src/theme.rs

impl ActiveTheme for App {
    fn theme(&self) -> &Arc<Theme> {
        // App não tem window context, usa global
        GlobalTheme::theme_for_workspace(None, self)
    }
}

// Adicionar método em Window
impl Window {
    pub fn theme(&self, cx: &App) -> &Arc<Theme> {
        let workspace_id = cx
            .try_global::<WindowWorkspaceMap>()
            .and_then(|map| map.0.get(&self.window_id()).copied());

        GlobalTheme::theme_for_workspace(workspace_id, cx)
    }
}
```

---

### Parte 7: Refatorar chamadas de `cx.theme()`

**Por quê?** Código que renderiza precisa usar `window.theme(cx)` ao invés de `cx.theme()`.

**Pattern:**

```rust
// Antes:
fn render(&mut self, _: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
    let colors = cx.theme().colors();
    // ...
}

// Depois:
fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
    let colors = window.theme(cx).colors();
    // ...
}
```

**Escopo da mudança:**

- Todo `Render` trait implementation
- Todo código que usa `cx.theme()` dentro de `Window` context
- ~200-300 locais estimados

**Estratégia:**

1. Adicionar `Window::theme()` sem remover `cx.theme()`
2. Deprecar `App::theme()` com warning
3. Adicionar clippy lint sugerindo `window.theme(cx)`
4. Refatorar gradualmente
5. (Opcional) Remover `App::theme()` em release futura

---

## 🚧 Edge Cases e Considerações

### 1. Multiple root paths no workspace

**Cenário:** Workspace com `/project-a` e `/project-b`.

**Solução:**

- Usa o **primeiro path** que matchar
- Documentar: "Se múltiplos paths, define workspace_profile mais específico primeiro"

### 2. Glob matching ambiguidade

**Cenário:**

```json
{
  "workspace_profiles": {
    "all_work": { "path": "~/work/*", "theme": "Dark" },
    "client_x": { "path": "~/work/client-x", "theme": "Light" }
  }
}
```

Abrir `~/work/client-x` → qual usar?

**Solução:**

- **Exact match ganha** (client_x)
- Se ambos são glob, usa o **mais específico** (menos wildcards)
- Documentar ordem de precedência

### 3. Path não existe ainda

**Cenário:** workspace_profile para `~/future-project` que não existe.

**Solução:**

- Não é problema! Path matching falha gracefully
- Quando criar o projeto, automaticamente matcha

### 4. Symbolic links

**Cenário:** Workspace aberto via symlink.

**Solução:**

- Resolver symlinks antes de matching
- `fs::canonicalize()` antes de comparar

### 5. Workspace sem database_id

**Cenário:** Testes, workspaces temporários.

**Solução:**

- Fallback para global settings
- `workspace_id.is_none()` → usa tema global

---

## 📝 Checklist de Implementação

### Phase 1: Settings Structure

- [ ] Adicionar `WorkspaceProfileContent` em `settings_content.rs`
- [ ] Adicionar `workspace_profiles: HashMap<...>` em `UserSettingsContent`
- [ ] Schema JSON para autocomplete/validation
- [ ] Testes de parse do settings.json

### Phase 2: Path Matching

- [ ] Implementar `Workspace::matched_workspace_profile()`
- [ ] Suporte a `~` expansion
- [ ] Suporte a glob patterns (`*`, `**`)
- [ ] Exact match vs glob priority
- [ ] Testes de matching (edge cases)

### Phase 3: SettingsStore Integration

- [ ] Adicionar `active_workspace_profiles: HashMap<WorkspaceId, Option<String>>`
- [ ] Implementar `set_active_workspace_profile()`
- [ ] Implementar `get_for_workspace::<T>()`
- [ ] Modificar `recompute_values()` para workspace layer
- [ ] Adicionar `workspace_values: HashMap<WorkspaceId, T>` em `SettingValue<T>`
- [ ] Implementar `value_for_workspace()` e `set_workspace_value()`

### Phase 4: GlobalTheme

- [ ] Adicionar `workspace_themes: HashMap<WorkspaceId, Arc<Theme>>`
- [ ] Implementar `configured_theme_for_workspace()`
- [ ] Implementar `reload_theme_for_workspace()`
- [ ] Implementar `theme_for_workspace()`
- [ ] Função helper `refresh_workspace_windows()`

### Phase 5: Workspace Integration

- [ ] Chamar `set_active_workspace_profile()` em `Workspace::new()`
- [ ] Observer para re-check quando settings muda
- [ ] Cleanup ao fechar workspace

### Phase 6: Window → Workspace Mapping

- [ ] Criar `WindowWorkspaceMap` global
- [ ] Registrar em `Workspace::new()`
- [ ] Cleanup ao fechar
- [ ] Adicionar `Window::theme(cx: &App)`

### Phase 7: Refactoring (Gradual)

- [ ] Deprecar `App::theme()` (ou manter como fallback)
- [ ] Adicionar clippy lint para `cx.theme()` em Window context
- [ ] Refatorar `Render` implementations principais
- [ ] Documentação de migração

### Phase 8: Testing

- [ ] Unit tests: path matching
- [ ] Unit tests: settings hierarchy
- [ ] Integration test: múltiplos workspaces
- [ ] Integration test: hot reload settings
- [ ] Manual testing: real projects

### Phase 9: Documentation

- [ ] Schema documentation (JSON schema)
- [ ] User-facing docs (como usar workspace_profiles)
- [ ] Migration guide (se breaking changes)

---

## 🧪 Testing Strategy

### Unit Tests: Path Matching

```rust
#[test]
fn test_workspace_profile_path_matching() {
    let profiles = [
        ("exact", WorkspaceProfile { path: "/home/user/project", .. }),
        ("glob", WorkspaceProfile { path: "/home/user/work/*", .. }),
        ("tilde", WorkspaceProfile { path: "~/Development/app", .. }),
    ];

    // Exact match
    assert_eq!(
        match_workspace_profile(&profiles, "/home/user/project"),
        Some("exact")
    );

    // Glob match
    assert_eq!(
        match_workspace_profile(&profiles, "/home/user/work/client-x"),
        Some("glob")
    );

    // Tilde expansion
    assert_eq!(
        match_workspace_profile(&profiles, "/home/user/Development/app"),
        Some("tilde")
    );

    // No match
    assert_eq!(
        match_workspace_profile(&profiles, "/tmp/random"),
        None
    );
}

#[test]
fn test_workspace_profile_precedence() {
    let profiles = [
        ("specific", WorkspaceProfile { path: "/work/client-x/backend", .. }),
        ("glob", WorkspaceProfile { path: "/work/*", .. }),
    ];

    // Exact match tem precedência
    assert_eq!(
        match_workspace_profile(&profiles, "/work/client-x/backend"),
        Some("specific")
    );
}
```

### Integration Tests: Multiple Workspaces

```rust
#[gpui::test]
async fn test_workspace_profiles_different_themes(cx: &mut TestAppContext) {
    cx.update(|cx| {
        SettingsStore::update_global(cx, |store, _| {
            store.set_user_settings(r#"{
                "workspace_profiles": {
                    "project_a": {
                        "path": "/tmp/project-a",
                        "theme": "One Dark"
                    },
                    "project_b": {
                        "path": "/tmp/project-b",
                        "theme": "Solarized Light"
                    }
                }
            }"#).unwrap();
        });
    });

    let workspace_a = open_workspace("/tmp/project-a", cx).await;
    let workspace_b = open_workspace("/tmp/project-b", cx).await;

    // Verify themes
    workspace_a.update(cx, |_, window, cx| {
        assert_eq!(window.theme(cx).name, "One Dark");
    });

    workspace_b.update(cx, |_, window, cx| {
        assert_eq!(window.theme(cx).name, "Solarized Light");
    });
}
```

### Integration Tests: Hot Reload

```rust
#[gpui::test]
async fn test_workspace_profile_hot_reload(cx: &mut TestAppContext) {
    let workspace = open_workspace("/tmp/project", cx).await;

    workspace.update(cx, |_, window, cx| {
        assert_eq!(window.theme(cx).name, "Default Dark");
    });

    // Update settings
    cx.update(|cx| {
        SettingsStore::update_global(cx, |store, _| {
            store.set_user_settings(r#"{
                "workspace_profiles": {
                    "project": {
                        "path": "/tmp/project",
                        "theme": "One Dark"
                    }
                }
            }"#).unwrap();
        });
    });

    // Should auto-update
    workspace.update(cx, |_, window, cx| {
        assert_eq!(window.theme(cx).name, "One Dark");
    });
}
```

---

## 🎯 Success Criteria

### Funcionalidade

- [ ] Workspace A com path `/project-a` usa tema X
- [ ] Workspace B com path `/project-b` usa tema Y
- [ ] Ambos abertos simultaneamente, temas independentes
- [ ] Hot reload: editar `workspace_profiles` → tema muda imediatamente
- [ ] Glob patterns funcionam (`~/work/*`)
- [ ] Fallback graceful quando sem workspace_profile

### Performance

- [ ] Path matching < 1ms (não bloqueia abertura)
- [ ] Recompute apenas workspace afetado (não global)
- [ ] Sem memory leaks (cleanup ao fechar workspace)

### Developer Experience

- [ ] JSON schema funcionando (autocomplete no editor)
- [ ] Error messages claros (path inválido, tema não encontrado)
- [ ] Documentação clara com exemplos

### Code Quality

- [ ] Segue padrões da codebase
- [ ] Não quebra APIs existentes (ou deprecação gradual)
- [ ] Testes cobrindo edge cases
- [ ] Code review aprovado

---

## 🤝 Estratégia de PR

### PR 1: Settings Structure (Smallest)

**Goal:** Adicionar estrutura sem funcionalidade.

**Changes:**

- `WorkspaceProfileContent` struct
- Schema JSON
- Parsing tests
- **Nenhuma mudança de comportamento**

**Review focus:** Schema design, naming

---

### PR 2: Path Matching Logic

**Goal:** Lógica de matching isolada.

**Changes:**

- `matched_workspace_profile()` implementation
- Path resolution (tilde, glob)
- Unit tests
- **Ainda não integrado ao resto**

**Review focus:** Algoritmo de matching, edge cases

---

### PR 3: SettingsStore Integration

**Goal:** Hierarchy com workspace layer.

**Changes:**

- `workspace_values` em `SettingValue<T>`
- `set_active_workspace_profile()`
- `recompute_values()` modificado
- **Settings funcionando, tema ainda não**

**Review focus:** Não quebrar settings existentes

---

### PR 4: GlobalTheme + Window Integration

**Goal:** Tema por workspace funcionando.

**Changes:**

- `GlobalTheme` workspace-aware
- `WindowWorkspaceMap`
- `Window::theme()`
- **Feature completa!**

**Review focus:** Performance, window lifecycle

---

### PR 5: Refactoring (Opcional)

**Goal:** Limpar código antigo.

**Changes:**

- Refatorar `cx.theme()` → `window.theme(cx)`
- Deprecations
- Clippy lints

**Review focus:** Não quebrar extensões

---

## 📚 Documentação para Usuários

### Example: workspace_profiles.md

````markdown
# Workspace Profiles

Configure settings per project path.

## Basic Usage

In your `settings.json`:

```json
{
  "workspace_profiles": {
    "work_project": {
      "path": "~/work/client-project",
      "theme": "One Dark",
      "ui_font_size": 14
    },
    "personal": {
      "path": "~/personal/*",
      "theme": "Solarized Light",
      "buffer_font_size": 16
    }
  }
}
```
````

## Path Matching

- **Exact paths**: `"/home/user/project"`
- **Tilde expansion**: `"~/Development/app"`
- **Glob patterns**: `"~/work/*"` matches all in work dir
- **Recursive globs**: `"~/work/**"` matches nested too

## Precedence

1. Exact match wins over glob
2. More specific glob wins (`/work/client/*` > `/work/*`)
3. If no match, uses global settings

## Supported Settings

All settings work, including:

- `theme`
- `ui_font_size`, `buffer_font_size`
- `tab_size`, `soft_wrap`
- Any other user setting

## FAQ

**Q: Does this commit to git?**
A: No! `settings.json` is in `~/.config/zed/`, not project.

**Q: Can I share workspace_profiles?**
A: Yes, share your `settings.json` with team.

**Q: What about `.zed/settings.json`?**
A: Local project settings still work and have higher priority.

```

---

## 🔄 Alternatives Considered

### Alternative 1: Modify profile-settings directly

**Why not:**
- Profiles já têm discussões de design internas
- `ActiveSettingsProfileName` é global por design
- Arriscado mudar comportamento existente

### Alternative 2: Store in database (like window bounds)

**Why not:**
- Path não é estável (rename, move)
- Não compartilhável entre máquinas
- Mais complexo (migrations, cleanup)

### Alternative 3: `.zed/settings.json` (project settings)

**Why not:**
- Comita no git (rejeitado pela comunidade)
- Settings pessoais (tema) não devem ser compartilhados

### ✅ Alternative 4: workspace_profiles (chosen)

**Why yes:**
- Path-based matching é intuitivo
- Fica no user settings (não comita)
- Ortogonal a profiles
- Simples de implementar
- Fácil de testar e documentar

---

## 🎨 Future Enhancements (Out of Scope)

### UI for managing workspace_profiles

Atualmente: editar JSON manualmente.

**Futuro:**
- Settings UI com lista de workspace_profiles
- "Add current workspace" button
- Visual indicator quando workspace_profile ativo

### Auto-detect workspace_profile

**Ideia:** Sugerir criar workspace_profile se usuário muda tema.

```

┌─────────────────────────────────────┐
│ You changed theme to "One Dark" │
│ │
│ Apply to this workspace only? │
│ Path: ~/Development/my-project │
│ │
│ [Yes] [No] [Don't ask again] │
└─────────────────────────────────────┘

````

### Workspace groups

**Ideia:** Agrupar múltiplos paths.

```json
{
  "workspace_profiles": {
    "client_work": {
      "paths": ["~/work/client-*", "~/projects/client-*"],
      "theme": "One Dark"
    }
  }
}
````

### Inherit from profiles

**Ideia:** workspace_profile extends profile.

```json
{
  "workspace_profiles": {
    "project": {
      "path": "~/project",
      "extends": "uirapuru",
      "theme": "Different Theme" // Override só theme
    }
  },
  "profiles": {
    "uirapuru": {
      "theme": "Tokyo Night",
      "ui_font_size": 14,
      "buffer_font_size": 16
    }
  }
}
```

---

## 📊 Impacto Estimado

### Lines of Code

- Settings structure: ~50 LOC
- Path matching: ~100 LOC
- SettingsStore changes: ~150 LOC
- GlobalTheme changes: ~100 LOC
- Window integration: ~50 LOC
- Tests: ~200 LOC
- **Total: ~650 LOC**

### Files Modified

- `settings/src/settings_content.rs` (structure)
- `settings/src/settings_store.rs` (core logic)
- `theme/src/theme.rs` (GlobalTheme)
- `workspace/src/workspace.rs` (integration)
- New: `workspace/src/workspace_profile.rs` (matching logic)

### Risk Assessment

- **Low risk:** Aditivo, não remove features
- **Medium complexity:** Precisa entender settings hierarchy
- **High value:** Feature muito pedida pela comunidade

---

## ✅ Final Thoughts

Esta abordagem é **superior** porque:

1. **Não mexe em systems sensíveis** - Profiles fica intacto
2. **Path-based é intuitivo** - "Esse path = essas configs"
3. **Familiar para usuários** - Já editam `settings.json`
4. **Testável** - Path matching é puro, determinístico
5. **Escalável** - Qualquer setting pode ser workspace-specific
6. **Compartilhável** - Time pode usar mesmo workspace_profiles
7. **Ortogonal** - Não compete com profiles, complementa

**Next Steps:**

1. Discutir com maintainers (antes de codar)
2. Implementar PR 1 (structure) para feedback
3. Iterar baseado em code review
4. Documentar bem para adoção

---

_Last updated: 2024_
_Author: Based on codebase exploration and community feedback_
