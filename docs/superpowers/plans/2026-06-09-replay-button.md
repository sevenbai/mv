# Replay 按鈕 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 影片播放結束後，於右下角顯示一顆純 icon、無文字的 replay 按鈕，點擊後從頭重播並維持全螢幕。

**Architecture:** 在既有單檔 `index.html` 中新增一顆 `position: fixed` 的圓形按鈕（內含 inline SVG 環形箭頭），沿用現有 `#overlay .icon` 的半透明白底 + 白邊視覺語言；以 `video` 的 `ended` 事件淡入按鈕，按鈕 `click` 時 `currentTime = 0` 並 `play()`，留在全螢幕。不更動現有點擊播放／全螢幕流程。

**Tech Stack:** 純 HTML + CSS + 原生 JavaScript（無 build / 無框架 / 無 test runner）。驗證採瀏覽器手動測試。

---

## File Structure

僅修改一個檔案：

- `index.html` — 全部變更集中於此：新增按鈕 DOM、對應 CSS、`ended` 與 `click` 兩個事件監聽。

無新增檔案。專案無 API router，不涉及 swagger。

---

### Task 1: 新增 replay 按鈕的 markup 與樣式

**Files:**
- Modify: `index.html`（`<style>` 區塊內新增樣式；`#overlay` div 之後新增按鈕 DOM）

- [ ] **Step 1: 在 `<style>` 區塊末端（`#overlay .hint { ... }` 規則之後、`</style>` 之前）新增按鈕樣式**

```css
    /* End-of-playback replay button (icon only, bottom-right) */
    #replayBtn {
      position: fixed;
      right: 24px;
      bottom: 24px;
      width: 56px;
      height: 56px;
      padding: 0;
      border-radius: 50%;
      background: rgba(255, 255, 255, 0.12);
      border: 2px solid rgba(255, 255, 255, 0.85);
      display: flex;
      align-items: center;
      justify-content: center;
      cursor: pointer;
      z-index: 20;
      transition: opacity 0.4s ease, background 0.2s ease;
    }

    #replayBtn:hover {
      background: rgba(255, 255, 255, 0.25);
    }

    #replayBtn.hidden {
      opacity: 0;
      pointer-events: none;
    }
```

- [ ] **Step 2: 在 `<div id="overlay">...</div>` 結束之後、`<script>` 之前，新增按鈕 DOM**

按鈕預設帶 `hidden` class（初始不顯示）；以 `aria-label` 提供無障礙標籤，畫面上不顯示任何文字；icon 為 Material「replay」環形箭頭 path，白色填滿，與現有白色播放三角形同一風格。

```html
  <button id="replayBtn" class="hidden" type="button" aria-label="重新播放">
    <svg viewBox="0 0 24 24" width="28" height="28" aria-hidden="true">
      <path fill="#fff" d="M12 5V1L7 6l5 5V7c3.31 0 6 2.69 6 6s-2.69 6-6 6-6-2.69-6-6H4c0 4.42 3.58 8 8 8s8-3.58 8-8-3.58-8-8-8z"/>
    </svg>
  </button>
```

- [ ] **Step 3: 在瀏覽器手動驗證按鈕外觀**

1. 用瀏覽器開啟 `index.html`。
2. 開啟 DevTools Console，執行：`document.getElementById('replayBtn').classList.remove('hidden')`
3. 預期：右下角出現一顆約 56px 的圓形按鈕，半透明白底 + 白色細邊，圓內為白色環形箭頭（replay icon），**沒有任何文字**。
4. 滑鼠移到按鈕上：預期背景變亮（透明度提高）、游標為手指。
5. 執行：`document.getElementById('replayBtn').classList.add('hidden')`
6. 預期：按鈕在約 0.4s 內淡出消失。

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m @'
Add replay button markup and styles

Bottom-right, icon-only circular button matching the existing white play
button style; hidden by default.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
'@
```

---

### Task 2: 串接 replay 行為（ended 顯示、click 重播）

**Files:**
- Modify: `index.html`（`<script>` 區塊，於 `overlay.addEventListener("click", start, { once: true });` 之後新增）

- [ ] **Step 1: 在 `<script>` 區塊內新增按鈕參照與事件監聽**

於既有 `overlay.addEventListener("click", start, { once: true });` 這行之後新增以下程式碼：

```javascript
    const replayBtn = document.getElementById("replayBtn");

    // Show the replay button every time playback ends (repeatable, no `once`).
    player.addEventListener("ended", () => {
      replayBtn.classList.remove("hidden");
    });

    // Restart from the beginning, stay in fullscreen, hide the button.
    replayBtn.addEventListener("click", () => {
      replayBtn.classList.add("hidden");
      player.currentTime = 0;
      const p = player.play();
      if (p && typeof p.catch === "function") {
        // Playback failed; show the button again so the user can retry.
        p.catch(() => { replayBtn.classList.remove("hidden"); });
      }
    });
```

- [ ] **Step 2: 手動驗證完整重播流程**

1. 用瀏覽器開啟 `index.html`。
2. 點擊中央 overlay → 影片進入全螢幕並帶聲音播放。
3. 在 Console 執行以下指令，快轉到接近結尾以觸發 `ended`：
   `const v = document.getElementById('player'); v.currentTime = v.duration - 0.5;`
4. 預期：影片播完後，右下角的 replay 按鈕淡入出現。
5. 點擊 replay 按鈕。
6. 預期：影片從頭（0 秒）重新播放、帶聲音、維持全螢幕；按鈕淡出消失。

- [ ] **Step 3: 手動驗證可重複 replay**

1. 接續上一步，再次在 Console 執行：
   `const v = document.getElementById('player'); v.currentTime = v.duration - 0.5;`
2. 預期：影片再次播完後，replay 按鈕**再次**淡入出現（驗證 `ended` 監聽未用 `once`，可重複）。

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m @'
Wire up replay button behavior

Show the button on video `ended`; clicking restarts playback from the
start while staying in fullscreen, and re-shows the button if play() is
rejected. Listener is repeatable across multiple playbacks.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
'@
```

---

## Self-Review

**Spec coverage（對照 `docs/superpowers/specs/2026-06-09-replay-button-design.md`）：**

- 右下角、距邊 24px、56px 圓鈕 → Task 1 Step 1 CSS ✓
- 半透明白底 + 白邊、沿用 `#overlay .icon` 風格 → Task 1 Step 1 ✓
- inline SVG 環形箭頭、無文字 → Task 1 Step 2 ✓
- 預設隱藏、0.4s opacity 淡入淡出 → Task 1 Step 1（`.hidden` + transition）✓
- hover 亮度回饋、`cursor: pointer` → Task 1 Step 1 ✓
- `ended` 顯示、不使用 `once`、可重複 → Task 2 Step 1 + Step 3 ✓
- click → `currentTime = 0` + `play()`、維持全螢幕、按鈕隱藏 → Task 2 Step 1 ✓
- `play()` 以 `.catch` 包覆 → Task 2 Step 1（並再次顯示按鈕，較 spec 更穩健）✓
- 不顯示原生 controls → 既有 `<video>` 未設 `controls`，本計畫不新增，維持現狀 ✓
- 僅改 `index.html` → File Structure ✓

無缺漏。

**Placeholder scan:** 無 TBD/TODO；每個改碼步驟皆含完整程式碼。

**Type/識別字一致性:** `replayBtn`、`player`、`hidden` class、`#replayBtn` 選擇器於各 Task 間命名一致。
