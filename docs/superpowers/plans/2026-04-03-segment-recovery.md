# Segment Recovery Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 파형에서 구간을 선택한 뒤 "구간 복구" 버튼을 눌러 삭제 표시된 영역의 일부 또는 전부를 복구한다.

**Architecture:** 기존 `deletedRegions` 배열과 `undoStack`을 그대로 활용한다. `undoStack`의 `added` 필드를 단일 객체에서 배열로 통일하여 복구 시 0~2개의 잔여 구간을 일관되게 처리한다. 복구 로직은 선택 범위와 겹치는 삭제 구간을 잘라내고 남은 부분을 새 오버레이로 재생성한다.

**Tech Stack:** Vanilla JS, WaveSurfer.js v6, 단일 HTML 파일(`index.html`)

---

### Task 1: "구간 복구" 버튼 HTML·CSS 추가

**Files:**
- Modify: `index.html` — `#actions` div에 버튼 추가, CSS에 `.btn-recover` 추가

- [ ] **Step 1: CSS에 `.btn-recover` 스타일 추가**

`index.html`의 `.btn-undo` 스타일 바로 다음 줄에 추가:

```css
.btn-recover { background: #10b981; color: #fff; }
```

- [ ] **Step 2: HTML에 버튼 추가**

`#actions` div 안, `↩ 되돌리기` 버튼 바로 다음에 추가:

```html
<button class="btn-recover" id="btn-recover" disabled>구간 복구</button>
```

- [ ] **Step 3: JS 상단에 `btnRecover` 상수 추가**

`btnUndo` 상수 선언 바로 다음에:

```javascript
const btnRecover = document.getElementById('btn-recover');
```

- [ ] **Step 4: `setButtonsEnabled`에 `btnRecover` 포함**

```javascript
function setButtonsEnabled(enabled) {
  btnPlayRegion.disabled = !enabled;
  btnExtract.disabled    = !enabled;
  btnDelete.disabled     = !enabled;
  btnRecover.disabled    = !enabled;
}
```

---

### Task 2: `undoStack.added`를 배열로 통일

**Files:**
- Modify: `index.html` — `btnDelete` 핸들러와 `btnUndo` 핸들러

현재 `btnDelete`는 `undoStack.push({ added, removed })` 형태로 `added`가 단일 객체다.
복구 시 `added`가 0~2개이므로 배열로 통일한다.

- [ ] **Step 1: `btnDelete` 핸들러에서 `added`를 배열로 변경**

기존:
```javascript
undoStack.push({ added, removed: overlapping });
```

변경:
```javascript
undoStack.push({ added: [added], removed: overlapping });
```

- [ ] **Step 2: `btnUndo` 핸들러를 배열 기반으로 수정**

기존 undo 핸들러 전체를 아래로 교체:

```javascript
btnUndo.addEventListener('click', () => {
  if (undoStack.length === 0) return;
  const { added, removed } = undoStack.pop();

  // added 배열의 오버레이 모두 제거
  for (const a of added) {
    const r = wavesurfer.regions.list[a.id];
    if (r) r.remove();
    deletedRegions = deletedRegions.filter(d => d.id !== a.id);
  }

  // removed 구간 복원 + undoStack 내 구 ID 갱신
  for (const seg of removed) {
    const restored = addDeletedRegionOverlay(seg.start, seg.end);
    deletedRegions.push({ start: seg.start, end: seg.end, id: restored.id });

    for (const entry of undoStack) {
      for (const a of entry.added) {
        if (a.id === seg.id) a.id = restored.id;
      }
      for (const rm of entry.removed) {
        if (rm.id === seg.id) rm.id = restored.id;
      }
    }
  }

  btnUndo.disabled     = undoStack.length === 0;
  btnDownload.disabled = deletedRegions.length === 0;
});
```

---

### Task 3: `applyRecovery` 함수 구현 및 버튼 핸들러 연결

**Files:**
- Modify: `index.html` — `applyAllDeletions` 함수 아래에 `applyRecovery` 추가, 버튼 핸들러 추가

- [ ] **Step 1: `applyRecovery` 함수 추가**

`applyAllDeletions` 함수 바로 다음에:

```javascript
function applyRecovery(rs, re) {
  const affected = deletedRegions.filter(d => d.start < re && d.end > rs);
  if (affected.length === 0) return false;

  const affectedIds = new Set(affected.map(d => d.id));
  for (const d of affected) {
    const r = wavesurfer.regions.list[d.id];
    if (r) r.remove();
  }
  deletedRegions = deletedRegions.filter(d => !affectedIds.has(d.id));

  const newRegions = [];
  for (const d of affected) {
    if (d.start < rs) {
      const r = addDeletedRegionOverlay(d.start, rs);
      const entry = { start: d.start, end: rs, id: r.id };
      newRegions.push(entry);
      deletedRegions.push(entry);
    }
    if (d.end > re) {
      const r = addDeletedRegionOverlay(re, d.end);
      const entry = { start: re, end: d.end, id: r.id };
      newRegions.push(entry);
      deletedRegions.push(entry);
    }
  }

  undoStack.push({ added: newRegions, removed: affected });
  return true;
}
```

- [ ] **Step 2: `btnRecover` 이벤트 핸들러 추가**

`btnUndo` 핸들러 바로 앞에:

```javascript
btnRecover.addEventListener('click', () => {
  if (!audioBuffer || !currentRegion) return;

  const recovered = applyRecovery(currentRegion.start, currentRegion.end);
  if (!recovered) {
    showError('선택한 구간에 삭제된 영역이 없습니다.');
    return;
  }

  // 현재 선택 구간 해제
  currentRegion.remove();
  currentRegion = null;
  startTimeEl.textContent = '-';
  endTimeEl.textContent   = '-';
  inputStart.value = '';
  inputEnd.value   = '';
  setButtonsEnabled(false);
  btnUndo.disabled     = false;
  btnDownload.disabled = deletedRegions.length === 0;
});
```

- [ ] **Step 3: 브라우저에서 동작 확인**

다음 시나리오를 수동으로 검증:

1. MP3 파일 로드
2. [10~20]초 구간 삭제 → 빨간 오버레이 확인
3. [13~17]초 선택 후 "구간 복구" → [10~13], [17~20] 오버레이 2개로 분리되는지 확인
4. "↩ 되돌리기" → [10~20] 단일 오버레이로 복원되는지 확인
5. [5~25]초 선택 후 "구간 복구" → 오버레이 전부 사라지는지 확인
6. 삭제된 영역 없는 구간 선택 후 "구간 복구" → 에러 메시지 표시 확인
7. 다운로드 → 복구된 구간이 오디오에 포함되는지 확인
