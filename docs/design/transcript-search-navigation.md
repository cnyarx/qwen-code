# Transcript search and structure navigation design

Status: implemented for the Ink virtual viewport.
Related: PR #17024, issues #16995 and #16996.

## 1. Background and goals

With `ui.useTerminalBuffer` on (the default Virtualized History path), the
conversation lives inside an in-app viewport. The host terminal's scrollback and
search cannot reach it, and the only movement was by distance: lines, pages,
start, and end. A user cannot directly find an earlier command's output or jump
back one turn.

This feature has two goals:

- Let the user search the **entire current session** and anchor the viewport to
  a result, including text inside structured tool results.
- Let the user move one **turn** or **tool-output block** at a time, without
  changing the meaning of existing page-scroll keys.

### 1.1 Scope

Supported: the Ink renderer, the virtual viewport returned by
`MainContent` when `ui.useTerminalBuffer` is true.

Not supported here: OpenTUI (which has a distinct scrollbox), legacy Static
history, per-occurrence highlighting inside rendered markdown, and automatic
expansion of a matched collapsed tool block. Search does not cross sessions.

### 1.2 Terminology

| Term                    | Meaning                                                                                                                                |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Item                    | One row of history rendered by the virtual viewport, not one line of text.                                                             |
| Turn                    | A user item with `type === 'user'` and `sentToModel !== false`, up to the next such item. A steer injection does not start a new turn. |
| Tool output             | One `tool_group` item, which can contain several tool calls and results.                                                               |
| Reading-position anchor | `getScrollIndex()`: the top item while scrolled up, or the last item while following output.                                           |
| Navigation cursor       | A separate record of the requested navigation target. It is not a scroll offset.                                                       |

## 2. Shared item model

`MainContent` builds its virtual list with `buildVirtualItems`. Search and
structure navigation use the **same function and the same item references**.
This is the index contract:

| Index    | Content                                                                                                                   |
| -------- | ------------------------------------------------------------------------------------------------------------------------- |
| `0`      | The `vp-banner` sentinel, never a navigation anchor or search match.                                                      |
| `1..N`   | Committed history, in display order.                                                                                      |
| `N+1..M` | Pending entries with negative IDs, separated from committed items so they cannot later collide when history is committed. |

If a live pending tool group has the same `batchId` as its committed copy, the
builder drops the committed one. Unrelated batches are not de-duplicated by
`callId`, because core history compaction and provider adapters may re-mint or
reuse those IDs. Consequently an index returned by search still addresses the
actual item shown by the viewport after that collapse.

This is a deliberate shared builder rather than independently searching
`uiState.history`, because independently filtering and collapsing the same
history could produce off-by-one jumps and would be very difficult to diagnose
in a long session.

## 3. Search processing and rendering

### 3.1 Matching

`searchTranscriptItems` converts every supported item into searchable text:

- Ordinary text-bearing history: `item.text`, including prompts, assistant
  responses, thoughts, notices, and errors.
- Tool groups: tool name, description, raw arguments, result display (including
  structured results serialized as JSON), full Ctrl+O detail, and notices.
- Tool-use summaries: their summary text.

Matching is case-insensitive, performed on the original text, at most one hit
per item. Other variants without a user-facing text field (for example `about`,
`help`, or `stats`) do not contribute searchable text.

Search result structure:

```ts
interface TranscriptSearchMatch {
  index: number; // index in buildVirtualItems output
  itemType: string; // e.g. 'user' or 'tool_group'
  snippet: string; // single-line terminal-safe context
}

interface TranscriptSearchResult {
  matches: readonly TranscriptSearchMatch[];
  truncated: boolean; // only true when an older match was really omitted
}
```

`getSearchableText` is cached in a `WeakMap` keyed by item identity, so normal
re-renders do not redo text projection for stable committed items. Pending items
are normally recreated and therefore miss this cache instead of retaining stale
results.

### 3.2 Cap and order

The matcher scans from newest to oldest and reverses the result before display.
At most 500 matching items are returned. With 501 matches, the 501st (newest)
remains present, in ascending viewport order.

`truncated` is computed **before adding the next match**. Therefore a result
that exactly fills the cap does not incorrectly render `500+`; one extra omitted
match is required.

### 3.3 Snippets and terminal safety

The snippet expands ±48 characters around the **raw** match position, then
applies `sanitizeTerminalText` and collapses whitespace. It does not search for
the needle again after flattening. This matters for these two concrete cases:

```text
"beta  gamma"
  => match "beta  gamma" in raw text
  => snippet contains "beta gamma", at the matched location, not item index 0

"before\x1b]0;ATTACK\x07 after"
  => snippet has no raw ESC or BEL bytes
```

Searching happens on the raw text so control-byte content can still be queried;
rendering alone is sanitized. There is no new security promise that every
plausible external tool producer has sanitized its data: matcher cleanup is now
also required and covered at that boundary.

## 4. Search mode arbitration

`InputPrompt` already reuses one composer buffer for Ctrl+R reverse search.
Transcript search uses the same established mode pattern rather than another
input component or new modal state.

| Input                             | While find is open                                                                                   | On close                                              |
| --------------------------------- | ---------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| `Alt+S`                           | Enter find and save draft, cursor, and reading-position anchor. Buffer is cleared for the new query. | N/A                                                   |
| Printable query text              | Replace the query and recompute matches. Never consume prompt-suggestion suggestions or dictation.   | Typed text goes to the prompt only after find closes. |
| Up / Down                         | Previous / next match, with wraparound.                                                              | N/A                                                   |
| `Ctrl+R`                          | Ignored: two modes cannot share the draft slot.                                                      | One `Esc` returns the original draft.                 |
| `!`, `?`, `Tab`, `Right`, `Enter` | No shell-mode toggle, shortcut panel, formatting completion, or prompt-suggestion acceptance.        | Enter keeps the selected match.                       |
| `Esc`                             | Close, restore the original draft, and restore the saved viewport state.                             | Follow mode is restored as below.                     |

The mode guards also cover plain typing, paste, and `/export` completion inputs,
not only named shortcuts. Otherwise `/export` or a printable `!` could bypass an
otherwise guarded shortcut and insert formatting or a suggestion into the
query.

## 5. Viewport restoration

When find opens, it stores the index returned by `getScrollIndex`. While the
viewport follows output, this is the last rendered item index. The item builder
is intentionally available both while find is open and while it is closed;
using an empty "inactive" list at activation would lose that state. This is how
follow is correctly recognized when starting find from the live tail.

```text
Enter: keep the viewport at the accepted match, restore draft.

Esc with saved follow state:
  scrollToEnd() -> re-enable follow.

Esc while reading somewhere else:
  scrollToIndex(saved index) -> keep that reading position, clamped if needed.

No saved index:
  scrollToEnd() -> safe follow fallback.
```

This distinction matters for a final assistant message taller than the
terminal: `scrollToIndex(last)` would put the beginning of that message at the
viewport top but disable later auto-follow. That is not the same as restoring
the bottom.

## 6. Structure navigation

### 6.1 Key dispatch

| Key             | Action                                                       |
| --------------- | ------------------------------------------------------------ |
| `Alt+PgUp`      | Previous turn anchor.                                        |
| `Alt+PgDn`      | Next turn anchor, or follow again when past the newest turn. |
| `Ctrl+Alt+Up`   | Previous tool-output group.                                  |
| `Ctrl+Alt+Down` | Next tool-output group.                                      |

`ScrollableList` owns all scroll-adjacent key handling and receives
`onNavigateStructure` from the item owner. It knows nothing about transcript
semantics. Structure matching runs **before** normal page scrolling because the
plain `pageup` binding deliberately ignores modifiers.

The modifier bits are load-bearing: `Meta+PgUp` is not plain PgUp. Negative
binding tests ensure plain page-scroll keys cannot unexpectedly become turn
jumps. While `onNavigateStructure` is absent, Alt+PageUp remains historical
page-up behavior.

### 6.2 Navigation cursor and resets

`findAdjacentAnchor(items, from, direction, kind)` is exclusive. So:

- from an item in the middle of turn N, previous lands on that turn's own
  prompt first, not an earlier turn;
- from an item already at turn N's prompt, previous lands on turn N - 1;
- no match gives -1, never wraps to the other end of the session.

The viewport clamps a requested scroll to its maximum possible offset. Once
the target is already inside the final page, later calls may leave its anchor
unchanged. Therefore structure jumps cannot use the new anchor as their next
origin. `createTranscriptNavCursor` instead records the requested target, so
consecutive keypresses each advance one boundary even though the visual offset
is clamped.

The cursor is reset when the reading position changes for an independent
reason:

1. Line/page scrolling on the keyboard.
2. Mouse wheel or scrollbar dragging.
3. External item positioning, such as a search jump (which uses
   `ScrollActions`).
4. An append that **actually moves** the reading-position anchor (auto-follow).
   An append that leaves it unchanged does not clear the cursor, so continuous
   streaming cannot repeatedly replay a clamped jump.

A navigation cursor is not follower-tracking: auto-follow state is only used by
the viewport itself, and can be restored with `scrollToEnd` when passing the
newest anchor.

### 6.3 Coalesced input

Wheel and drag events may be batched in a single frame. Structure jumps cancel
the queued scroll flush before dispatching, so a queued wheel delta cannot
apply after the jump and displace the requested item.

### 6.4 External interfaces and failures

This is a local UI feature, so there is no server API or error-code surface.
Its failure states are deliberately bounded:

| Situation                    | Behavior                                                                                                                                     |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| No older/newer anchor        | `At first turn` / `At latest turn`; do not wrap.                                                                                             |
| Legacy Static mode           | `scrollToIndex` returns false and search reports that Virtualized History is required. Mode missing implementation is never claimed to work. |
| Target already on final page | Scroll clamping may keep the image still, but the cursor still advances. This is clamping, not a missing jump.                               |
| Empty query                  | No matching work.                                                                                                                            |
| Empty search result          | Match position -1. New matches arriving later clamp to zero, and subsequent arrow navigation has no -1 dead-key or off-by-one state.         |
| New output while searching   | Re-clamp the active position; no teleport to the newest hit.                                                                                 |
| New output while navigating  | Reset only if the actual reading-position anchor moved.                                                                                      |

## 7. Validation evidence

The following kinds of tests are permanent, not planned:

- Matching includes structured tool output and Ctrl+O detail text.
- Index computation with pending and de-duplicated items stays aligned.
- Initial selection, wraparound, negative positions, list growth, and direction
  symmetry are covered by hook tests.
- Match-cap order, exact full cap, whitespace-heavy snippets, and control
  sequences are covered at function level.
- Keys include negative assertions for missing modifiers.
- Search-mode tests cover drafts, query acceptance, typing cleanup, voice
  suppression, and restoring the saved viewport/follow state.
- Navigation-cursor tests cover clamped consecutive jumps, unchanged-anchor
  append, real auto-follow, and manual-scroll reset.
- Coalesced wheel input does not displace a jump.

The validation plan also includes real tmux interaction. Test-only "gate
passed" output is not substituted for real interactive behavior.

## 8. Compatibility and limits

Existing line/page scroll keys retain behavior. There are no new settings,
migrations, server routes, external protocol changes, or security configuration
changes. Source search is scoped to the current session.

OpenTUI is not modified. Existing `<Static>` history cannot be addressed by item
and is explicitly unsupported. Findings outside this implementation must not be
represented as tested merely because they share this file's terminology.

---

# 中文说明

## 1. 背景和边界

默认虚拟视口会把会话渲染在应用内部，终端自己的滚动历史无法搜索。此前只能按行翻页、按页翻页，想找更早命令输出只能反复滚屏。

本次只支持 Ink 渲染器和 `ui.useTerminalBuffer=true` 的虚拟视口。OpenTUI、旧 Static 模式、Markdown 行内高亮和自动展开命中块不在范围内。不会跨会话检索，也没有新增配置项或外部协议。

“一轮”以用户实际发给模型的 `user` 条目开始，中途插入的 steer 消息不另起一轮。“一个工具输出块”对应一个 `tool_group`，可以包含两次以上的工具调用，不是按某几行文本计算。

## 2. 索引与共享数据

`MainContent` 和搜索、结构导航共用同一个 `buildVirtualItems`：第 0 项是横幅；之后是已落盘历史；再之后是负 ID 的实时项。还在流式输出的 `tool_group` 若与已提交副本有同一 `batchId`，只留实时副本，避免行重复。不会用 `callId` 瞎做去重，因为 core 压缩历史或适配器会换号。

这意味着匹配返回第 N 项就一定就是视口的第 N 项，不会因为一边去重、另一边不去重而跳错地方。

## 3. 搜索处理

普通条目搜索 `text`；工具组搜索工具名、说明、原始参数、结果（含结构化结果序列化)、`Ctrl+O` 完整输出与提示信息。大小写不敏感，一个条目最多有一条命中。

匹配从最新向最早扫描，再反转显示，保住最新 500 条且保持升序。恰好 500 条不会误报 `500+`，只有确实丢弃了更早命中才算截断。同身份历史项用 `WeakMap` 缓存投影后的文本，实时项重建时不复用旧缓存。

摘要直接从原始命中位置向前后扩展，再调用 `sanitizeTerminalText`，最后收敛多余空白。不会先折叠空白后再找查询词，因而带连续空格的 `beta  gamma` 仍能从真正命中位置取摘要。原始 ESC/BEL 控制码可以出现在历史数据中，但不会直接进入摘要显示。

## 4. 搜索模式

搜索沿用 `Ctrl+R` 的成熟做法：复用当前输入框而不是新增窗口。进入时保存草稿、光标和读取位置，输入框被清空用于查询；退出恢复草稿。

- `↑`、`↓` 循环切换命中。
- `Enter` 保留当前命中并交还原草稿。
- `Esc` 退出搜索，并恢复打开时的读取位置或跟随状态。
- `Ctrl+R` 禁止叠加，不能覆盖草稿槽。
- `!`、`?`、普通文本、粘贴、`/export`、`Tab`、`Right`、`Enter` 都不能触发 shell 切换、快捷键面板、格式化补全或提示建议接收。

精确条件放在按键管线的各个分支处，而不只是在快捷键登记处：否则 `/export` 或一个 `!` 就能绕过主搜索模式，把别的文本塞进查询。

## 5. 退出后的跟随

实时尾部的锚点本来就是最后一个条目，不是空值。保存尾态时，要用完整条目序列来判断，不能用一个“搜索未打开就为空”的列表，否则尾态判断必然失败。

可能在最后一条很高回复打开的搜索，如果退出时执行 `scrollToIndex(last)`，会把长回复顶端放到屏幕上方，而不是恢复底部；新输出就不会继续自动滚入视野。正确处理是尾部进入的退出用 `scrollToEnd`，其他位置用保存的索引（受最大滚动范围限制）。

## 6. 结构导航

- `Alt+PgUp` / `Alt+PgDn`：上/下一轮；尾部首次回退先落在这一轮的提问处，下一次才到上一轮。
- `Ctrl+Alt+↑` / `Ctrl+Alt+↓`：上/下一个工具输出块。
- 四个按键都必须携带修饰键，普通 `PgUp/PgDn` 仍只翻页。

虚拟视口在最后一屏会裁剪目标滚动位置，所以上一次跳转后，新的读取锚点可能没变。不能再用它算下一次，否则总是跳同一个位置。独立的导航游标记录用户真正请求的目标，保证连续按键每次移动一个边界。手动滚动、滚轮、拖滚动条、搜索跳转、真正跟随新输出时重置游标；仅有内容追加但读取位置没动时，不重置。

## 7. 边界和测试

- 没有更早/更晚锚点只提示，不循环跳去另一端。
- 空查询没有结果；后来内容到位后，前进/后退没有哨兵 -1 造成的死键或错位。
- 新历史只能钳制搜索位置，不能把用户正在看的命中挤走。
- 没有 Virtualized History 时明确报告不支持，不能假装完成。
- 新输出只有在真正移动读取位置时才重置导航游标。
- 同帧排队的滚轮冲刷要先取消，再执行结构跳转。

固定回归覆盖：结构化工具输出、索引一致性、空结果增长、修饰键负例、恰好上限、连续空白、终端控制码、草稿与尾态恢复、钳住时连续跳转和同帧滚轮竞态。还使用真实 tmux 做交互验证，不把单纯“门禁通过”当作实测结论。

## 8. 限制

OpenTUI 未改，旧 Static 不支持条目定位。命中当前不提供行内高亮，也不会自动展开折叠的工具块。搜索摘要中的控制序列已转义；不假设外部工具已净化历史数据，因此搜索渲染层自身也执行脱敏。
