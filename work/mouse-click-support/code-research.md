# Mouse Click Support for Session List - Code Research

**Feature Path:** `work/mouse-click-support` (proposed)  
**Research Date:** 2026-03-19  
**Context:** Add mouse click-to-select functionality to the session list in the aoe TUI application

---

## 1. Entry Points

### Main TUI Loop
**File:** `src/tui/app.rs`

- **`App::run()`** (line 129): Main event loop that polls for events
- **Event handling** (lines 169-186): Currently handles `Event::Key` and `Event::Mouse`
- **`handle_mouse()`** (line 320): Delegates mouse events to `HomeView::handle_mouse()`

```rust
// Current mouse handling in app.rs:179
Event::Mouse(mouse) => {
    self.handle_mouse(mouse, terminal).await?;
    terminal.draw(|f| self.render(f))?;
    continue;
}
```

### Home View
**File:** `src/tui/home/mod.rs`

- **`HomeView` struct** (line 94): Main state for session list view
- **`cursor: usize`** (line 107): Current selected index in `flat_items`
- **`flat_items: Vec<Item>`** (line 103): Flattened list of sessions and groups

### Rendering
**File:** `src/tui/home/render.rs`

- **`render_list()`** (line 126): Renders the session list with ratatui `List` widget
- **`render_item()`** (line 208): Renders individual list items (sessions/groups)
- **List area calculation** (lines 140-142): `inner = block.inner(area)` defines clickable region

```rust
// Current list rendering (render.rs:157-171)
let list_items: Vec<ListItem> = self
    .flat_items
    .iter()
    .enumerate()
    .map(|(idx, item)| {
        let is_selected = idx == self.cursor;
        // ... render logic
    })
    .collect();

let list = List::new(list_items).highlight_style(Style::default().bg(theme.session_selection));
frame.render_widget(list, inner);
```

---

## 2. Data Layer

### Session Items
**File:** `src/session/mod.rs` (not shown in search, referenced throughout)

- **`Item` enum**: Represents either a `Group` or `Session` in the flat list
- **`Instance` struct**: Session data with fields:
  - `id: String` - unique identifier
  - `title: String` - display name
  - `status: Status` - Running, Stopped, Error, etc.
  - `project_path: String` - for preview
  - `group_path: String` - hierarchical grouping

### Flat Items Structure
**File:** `src/tui/home/mod.rs` (line 673-691)

```rust
pub(super) fn build_flat_items(&self) -> Vec<Item> {
    // Builds hierarchical list with groups and sessions
    // Returns Vec<Item> where each Item has depth information
}
```

---

## 3. Similar Features

### Diff View Mouse Handling
**File:** `src/tui/diff/input.rs` (lines 172-188)

Existing mouse handling pattern for scroll events:

```rust
pub fn handle_mouse(&mut self, mouse: MouseEvent) -> DiffAction {
    if self.show_help || self.branch_select.is_some() {
        return DiffAction::Continue;
    }
    
    match mouse.kind {
        MouseEventKind::ScrollUp => {
            self.scroll_up(3);
            DiffAction::Continue
        }
        MouseEventKind::ScrollDown => {
            self.scroll_down(3);
            DiffAction::Continue
        }
        _ => DiffAction::Continue,
    }
}
```

**Key patterns to reuse:**
- Check for overlay/dialogs first (return `Continue` if active)
- Match on `mouse.kind` for different event types
- Return appropriate action enum

### Search Navigation
**File:** `src/tui/home/input.rs` (lines 481-509)

Keyboard-based navigation that mouse click should replicate:
- `j/k` or arrow keys change `self.cursor`
- `self.update_selected()` syncs `selected_session`/`selected_group`

---

## 4. Integration Points

### Mouse Event Imports
**Files requiring updates:**
- `src/tui/home/input.rs` - already imports `MouseEvent` (line 3)
- `src/tui/app.rs` - already imports `MouseEvent` (line 4)

### Current Mouse Enablement
**File:** `src/tui/mod.rs` (lines 93, 111)

Mouse capture is **already enabled** at the terminal level:
```rust
execute!(stdout, EnterAlternateScreen, EnableMouseCapture)?;
// ...
execute!(terminal.backend_mut(), LeaveAlternateScreen, DisableMouseCapture)?;
```

### Action Enum
**File:** `src/tui/app.rs` (line 508)

```rust
pub enum Action {
    Quit,
    AttachSession(String),
    AttachTerminal(String, TerminalMode),
    EditFile(PathBuf),
    StopSession(String),
    SetTheme(String),
}
```

Mouse clicks should **not** return an Action - they should update internal state directly (cursor position).

---

## 5. Existing Tests

### Test Framework
**Pattern observed in:** `src/tui/diff/input.rs` (lines 192-267)

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crossterm::event::KeyModifiers;
    
    fn key(code: KeyCode) -> KeyEvent {
        KeyEvent::new(code, KeyModifiers::NONE)
    }
    
    #[test]
    fn test_normal_keys_work_without_warning() {
        let mut view = make_diff_view_no_warning();
        let action = view.handle_key(key(KeyCode::Char('q')));
        assert!(matches!(action, DiffAction::Close));
    }
}
```

**Test gaps:**
- No existing mouse event tests in `home/input.rs`
- No tests for `handle_mouse()` in `HomeView`

---

## 6. Shared Utilities

### Layout Calculation
**File:** `src/tui/home/render.rs`

- **`Layout::default()`** (lines 56-65): Splits area into list/preview panes
- **`block.inner(area)`** (line 140): Gets inner rectangle excluding borders

### Item Rendering
**File:** `src/tui/home/render.rs`

- **`get_indent(depth: usize)`** (line 78): Returns indentation string for tree hierarchy
- **Status icons** (lines 83-92): Constants for running, idle, error states

### Selection Logic
**File:** `src/tui/home/input.rs`

- **`update_selected()`** (not shown, called throughout): Syncs cursor with `selected_session`/`selected_group`
- **`select_session_by_id()`** (referenced in app.rs:437): Select session by ID

---

## 7. Potential Problems

### 1. **No ListState Usage**
**Issue:** Current implementation uses static `List` widget without `ListState`

```rust
// Current (render.rs:167)
let list = List::new(list_items).highlight_style(...);
frame.render_widget(list, inner);
```

**Problem:** Ratatui's `ListState` provides built-in selection tracking, but the current code manually tracks `cursor` index.

**Impact:** Mouse click must manually calculate which item was clicked based on y-coordinate.

### 2. **Variable Row Heights**
**Issue:** Session rows may have different heights due to:
- Workspace info suffixes (line 328-337 in render.rs)
- Sandbox mode indicators
- Search match highlighting

**Impact:** Cannot simply divide `mouse.row` by constant row height. Must track rendered line positions.

### 3. **Scroll Offset**
**Issue:** List widget may scroll when items exceed visible area

**Impact:** Mouse click must account for scroll offset when calculating selected item. Current code doesn't expose scroll state.

### 4. **Dialog Overlays**
**Issue:** Multiple dialogs can overlay the list (new session, confirm, etc.)

**Impact:** Mouse clicks should be ignored when dialogs are active. Current `handle_mouse()` only checks `diff_view`.

### 5. **Group Collapse/Expand**
**Issue:** Groups can be collapsed/expanded (lines 725-745 in input.rs)

**Impact:** Clicking on group should toggle collapse, not select. Need to distinguish group vs session clicks.

### 6. **Terminal Resize**
**Issue:** List area changes on terminal resize

**Impact:** Must recalculate clickable bounds on each render, not cache coordinates.

---

## 8. Constraints & Infrastructure

### Dependencies
**File:** `Cargo.toml`

```toml
ratatui = { version = "0.29", features = ["crossterm"] }
crossterm = "0.28"
```

### Mouse Event API (crossterm 0.28)
```rust
pub struct MouseEvent {
    pub row: u16,        // 0-indexed row from top
    pub column: u16,     // 0-indexed column from left
    pub kind: MouseEventKind,
    pub modifiers: KeyModifiers,
    pub button: MouseButton,
}

pub enum MouseEventKind {
    Down(MouseButton),
    Up(MouseButton),
    Drag(MouseButton),
    Moved,
    ScrollUp,
    ScrollDown,
    ScrollLeft,
    ScrollRight,
}

pub enum MouseButton {
    Left,
    Right,
    Middle,
}
```

### Build System
- **Rust edition:** 2021
- **MSRV:** 1.74
- **Profile:** Release uses LTO, strip

### Pre-commit Hooks
**File:** `Cargo.toml` (dev-dependencies)
```toml
cargo-husky = { version = "1", default-features = false, features = ["precommit-hook", "run-cargo-fmt", "run-cargo-clippy"] }
```

---

## 9. External Libraries

### Ratatui 0.29
**Purpose:** TUI rendering framework

**Key APIs for this feature:**
- `ratatui::prelude::Rect` - rectangle coordinates (x, y, width, height)
- `ratatui::widgets::List` - list widget (currently used without state)
- `Frame::render_widget()` - renders widgets

**Documentation:** https://docs.rs/ratatui/0.29

### Crossterm 0.28
**Purpose:** Terminal manipulation and event handling

**Key APIs for this feature:**
- `crossterm::event::MouseEvent` - mouse event struct
- `crossterm::event::MouseEventKind` - type of mouse event
- `EnableMouseCapture` / `DisableMouseCapture` - terminal commands

**Documentation:** https://docs.rs/crossterm/0.28

---

## Implementation Plan

### Files to Modify

#### 1. **`src/tui/home/input.rs`** (Primary)
**Lines to add:** ~50-80 lines

Add mouse handling logic to `handle_mouse()` method (currently line 977):

```rust
pub fn handle_mouse(&mut self, mouse: MouseEvent) -> Option<Action> {
    // Pass mouse events to diff view if active
    if let Some(ref mut diff_view) = self.diff_view {
        match diff_view.handle_mouse(mouse) {
            DiffAction::Continue => return None,
            DiffAction::Close => {
                self.diff_view = None;
                return None;
            }
            DiffAction::EditFile(path) => {
                return Some(Action::EditFile(path));
            }
        }
    }
    
    // Ignore mouse if any dialog is active
    if self.has_dialog() {
        return None;
    }
    
    // Handle left click for selection
    if let MouseEventKind::Down(MouseButton::Left) = mouse.kind {
        // Check if click is within list bounds (need to track list area from render)
        // Calculate which item was clicked based on mouse.row
        // Update self.cursor and call self.update_selected()
    }
    
    None
}
```

**Challenge:** Need access to list area bounds from render phase.

#### 2. **`src/tui/home/render.rs`** (Supporting)
**Lines to modify:** ~20-30 lines

Store list area bounds for mouse handling:

```rust
// Add field to HomeView or pass bounds to handle_mouse
// Option 1: Store last rendered bounds
pub struct HomeView {
    // ... existing fields
    last_list_area: Option<Rect>,
}

// In render_list():
self.last_list_area = Some(inner);
```

#### 3. **`src/tui/home/mod.rs`** (Supporting)
**Lines to add:** ~5-10 lines

Add field to store list bounds:
```rust
pub struct HomeView {
    // ... existing fields (around line 94)
    last_list_area: Option<Rect>,
}
```

### Recommended Approach

#### **Option A: Simple Row-Based Selection** (Recommended for MVP)
1. Store `last_list_area: Option<Rect>` in `HomeView`
2. In `render_list()`, save `inner` bounds
3. In `handle_mouse()`:
   - Check if `mouse.row` is within `last_list_area.y` to `last_list_area.y + height`
   - Calculate row index: `(mouse.row - list_area.y) as usize`
   - Account for search bar if active (takes 1 row at bottom)
   - Clamp to `flat_items.len()`
   - Update `self.cursor` and call `self.update_selected()`

**Pros:** Simple, minimal changes  
**Cons:** Doesn't handle variable row heights perfectly

#### **Option B: Precise Item Hit-Testing**
1. During render, build a map of y-coordinate ranges to item indices
2. Store this map in `HomeView`
3. In `handle_mouse()`, look up which item contains `mouse.row`

**Pros:** Accurate for variable heights  
**Cons:** More complex, requires more state

#### **Option C: Use ListState**
1. Refactor to use ratatui's `ListState`
2. Use `StatefulWidget` pattern
3. `ListState` handles selection automatically

**Pros:** More idiomatic ratatui  
**Cons:** Significant refactoring, may break existing behavior

### Specific Implementation Steps (Option A)

1. **Add field to `HomeView`** (`src/tui/home/mod.rs:94`):
   ```rust
   last_list_area: Option<Rect>,
   ```

2. **Initialize in `HomeView::new()`** (`src/tui/home/mod.rs:187`):
   ```rust
   last_list_area: None,
   ```

3. **Store bounds in `render_list()`** (`src/tui/home/render.rs:140`):
   ```rust
   self.last_list_area = Some(inner);
   ```

4. **Implement click handling** (`src/tui/home/input.rs:977`):
   ```rust
   use crossterm::event::{MouseButton, MouseEventKind};
   
   pub fn handle_mouse(&mut self, mouse: MouseEvent) -> Option<Action> {
       // Existing diff view handling...
       
       // Ignore if dialog active
       if self.has_dialog() {
           return None;
       }
       
       // Only handle left click
       if let MouseEventKind::Down(MouseButton::Left) = mouse.kind {
           if let Some(list_area) = self.last_list_area {
               // Check if click is within list vertical bounds
               if mouse.row >= list_area.y && mouse.row < list_area.y + list_area.height {
                   // Calculate which row was clicked
                   let relative_row = (mouse.row - list_area.y) as usize;
                   
                   // Account for search bar taking bottom row
                   let effective_height = if self.search_active {
                       list_area.height.saturating_sub(1)
                   } else {
                       list_area.height
                   };
                   
                   if relative_row < effective_height as usize && relative_row < self.flat_items.len() {
                       self.cursor = relative_row;
                       self.update_selected();
                   }
               }
           }
       }
       
       None
   }
   ```

5. **Add tests** (`src/tui/home/input.rs`, new `#[cfg(test)]` module):
   - Test click within list bounds
   - Test click outside list bounds
   - Test click with dialog active
   - Test click with search active

### Additional Features (Future)

- **Double-click to attach:** Track click timing, double-click calls `Action::AttachSession`
- **Right-click context menu:** Show delete/stop/rename options
- **Scroll wheel support:** Already partially handled by ratatui, but could enhance
- **Click on group to collapse/expand:** Check if item is `Item::Group`, toggle collapsed state

---

## Summary

**Current State:**
- Mouse capture is enabled at terminal level
- `handle_mouse()` exists but returns `None` for all events
- List rendering uses static `List` widget without `ListState`
- Keyboard navigation (`j/k`) works via `cursor` index

**Required Changes:**
1. Store list area bounds from render phase
2. Implement click-to-select logic in `handle_mouse()`
3. Handle edge cases: dialogs, search, groups vs sessions
4. Add tests for mouse interaction

**Estimated Complexity:** Medium
- Core logic: ~50 lines
- Testing: ~30 lines
- Risk: Low (isolated to home view, existing mouse infrastructure)

**Files to Change:**
1. `src/tui/home/mod.rs` - add `last_list_area` field
2. `src/tui/home/render.rs` - store bounds during render
3. `src/tui/home/input.rs` - implement click handling logic
