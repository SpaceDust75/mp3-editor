# MP3 합치기 기능 설계 문서

## 개요

기존 MP3 편집기에 두 MP3 파일을 합치는 기능을 추가한다. 편집 기능과 완전히 분리된 탭으로 제공한다.

---

## UI 구조

### 탭 바
- `<h1>` 바로 아래에 탭 바 추가
- 탭 항목: `편집` / `합치기`
- 기본 활성 탭: `편집`
- 탭 전환 시 `#editor-section` / `#merge-section`을 `display: block/none`으로 토글
- 탭 간 이동해도 각 섹션의 상태(로드된 파일, 파형 등)는 유지

### 합치기 섹션 (`#merge-section`)
1. **파일 업로드 존 2개** — 각각 독립적으로 드래그앤드롭 및 클릭 업로드 지원. 파일 로드 후 파일명과 재생 길이(초) 표시.
2. **순서 목록** — 두 파일이 모두 로드된 경우 순서 표시 (1번, 2번). HTML5 Drag and Drop API로 순서 변경 가능. 순서 변경 시 목록 즉시 갱신.
3. **"합치기 & 다운로드" 버튼** — 두 파일 모두 로드된 경우에만 활성화. 클릭 시 스피너 표시 후 MP3 다운로드.
4. **에러 메시지 영역** — 파일 형식 오류 등 표시.

---

## 합치기 로직

### AudioBuffer 병합
```
mergeBuffers(bufferA, bufferB) → AudioBuffer
```
- 채널 수: `Math.max(bufferA.numberOfChannels, bufferB.numberOfChannels)`
- 샘플레이트: `bufferA.sampleRate` 기준 (두 번째 파일이 다를 경우 OfflineAudioContext로 리샘플링하지 않고, 첫 번째 파일 샘플레이트 그대로 사용 — 대부분 44100Hz로 동일)
- 총 길이: `bufferA.length + bufferB.length`
- 채널 수가 다를 경우(모노/스테레오 혼용):
  - 모노 파일은 동일 채널 데이터를 양쪽 채널에 복사해 스테레오로 업믹스

### 인코딩 및 다운로드
- 기존 `audioBufferToMp3(buffer)` 함수 재사용
- 파일명: `{파일1명}_{파일2명}_합치기.mp3`

---

## 상태 관리

| 변수 | 설명 |
|---|---|
| `mergeSlots[0]` | 첫 번째 슬롯 `{ file, audioBuffer, name, duration }` |
| `mergeSlots[1]` | 두 번째 슬롯 `{ file, audioBuffer, name, duration }` |
| `mergeOrder` | 현재 순서 `[0, 1]` 또는 `[1, 0]` |

---

## 에러 처리

- MP3가 아닌 파일 업로드 시: 해당 슬롯에 에러 메시지 표시
- 파일 디코딩 실패 시: 에러 메시지 표시, 슬롯 초기화
- 합치기 버튼은 두 슬롯 모두 정상 로드된 경우에만 활성화

---

## 범위 외 (YAGNI)

- 3개 이상 파일 합치기 — 미지원
- 합치기 전 구간 편집 — 미지원 (편집 탭에서 별도 수행)
- 샘플레이트 변환(리샘플링) — 미지원 (동일 샘플레이트 가정)
- 파형 미리보기 — 미지원
