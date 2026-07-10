---
name: 3d-knowledge-graph-html
description: >-
  이 리포의 3D 지식그래프 HTML(index.html · 에이전트_시스템_3D지도.html)을 만들거나 수정할 때 사용한다.
  3d-force-graph 기반 시각화의 노드/엣지 데이터 편집, 점선 엣지, 부모 주위 공전(궤도) 모션, 노드 드래그,
  카메라 자동 회전, 초기 필터 상태, Playwright 헤드리스 검증까지 이 리포 특유의 구조와 함정을 담고 있다.
  Use when building or editing the 3D knowledge-graph HTML in this repo (3d-force-graph based):
  nodes/edges, dashed links, orbital motion around parent nodes, node drag, camera auto-rotate,
  and headless browser verification.
---

# 3D 지식그래프 HTML 빌딩 스킬

에이전트 시스템 3D 지도(3d-force-graph 기반)를 만들고 고칠 때의 구조·기법·함정 모음.
2026-07-10 세션에서 정립됨.

## 대상 파일 & 동기화 규칙 (중요)
- `index.html` 과 `에이전트_시스템_3D지도.html` 은 **항상 완전히 동일**해야 한다.
- 한쪽을 편집하면 반드시 `cp index.html 에이전트_시스템_3D지도.html` 로 동기화하고 `diff -q` 로 확인.
- `claude log/`, `working log/` 는 `.gitignore` 대상(내부 기록, 공개 리포에 안 올림).

## 아키텍처 핵심
- three.js + three-spritetext + 3d-force-graph 가 **하나의 IIFE 로 인라인 번들**되어 있음.
  전역에는 `window.ForceGraph3D`, `window.SpriteText` 만 노출된다.
- **THREE 는 전역 미노출** (번들 내부 지역변수 `et`). 그래서 `THREE.LineDashedMaterial` 등을 직접 못 만든다.
  필요한 생성자는 **라이브 씬 객체에서 harvest** 해야 한다 (아래 점선 엣지 참고).
- 데이터: `HANGYEOL`(에이전트), `SKILLS`, `STUDENTS`, `CAT` 등 상수 → `buildData()` 가 필터 체크박스
  (`fSkill`/`fAxis`/`fWorkOnly`)를 읽어 `{nodes, links}` 생성. 링크는 `source=부모, target=자식` 트리 구조.
- 초기 필터 상태는 HTML `<input type="checkbox" ... [checked]>` 로 지정.

## tickFrame 동작 (모든 애니메이션의 근거)
`tickFrame` 안에서 **노드/링크 위치 갱신과 `onEngineTick` 콜백은 `engineRunning`일 때만** 실행된다.
엔진은 `cooldownTime`(기본 15s) 후 멈춘다. 따라서 매 프레임 훅이 필요하면:
```js
Graph.cooldownTime(Infinity);   // 엔진 상시 가동 → onEngineTick 매 프레임 호출
```
공전 모션·점선 갱신·자동 회전 모두 이 상시 가동 + `onEngineTick` 에 의존한다.

## 점선 엣지
기본 링크는 `linkWidth>0`이면 실린더 **Mesh**로, `0`이면 **Line**으로 생성된다.
Line 을 확보해야 THREE 생성자를 harvest 할 수 있으므로:
```js
Graph.linkWidth(0);                 // 링크를 Line 으로
// 씬에서 생성자 harvest: Line, BufferGeometry, BufferAttribute(pos), LineBasicMaterial,
//   Object3D(= Object.getPrototypeOf(Line.prototype).constructor)
// linkThreeObjectExtend(false) + linkThreeObject(그룹=Object3D + 다중 짧은 Line 대시)
// linkPositionUpdate(대시 재배치, true 반환 시 기본 배치 생략)
Graph.refresh();                    // 기존 실선 객체를 점선 그룹으로 재생성
```
- 대시는 지연 할당(`while(dashes.length<count) makeDash()`) + `MAXDASH` 상한, `frustumCulled=false`.
- harvest 는 기본 실선이 씬에 생긴 뒤에야 성공하므로 `requestAnimationFrame` 로 재시도.
- 색은 `material.color.set(css)` (rgba 알파는 무시되니 `opacity` 로 별도 반영).

## 공전(궤도) 모션 — 부모 주위를 도는 노드
- `SETTLE_MS`(~3200ms) 동안 힘-기반 배치 후 `startOrbit()` 로 전환.
- `startOrbit()`: 링크로 `parentOf` 맵 구성 → 각 노드의 부모 대비 offset 으로
  반지름 `r`, 궤도면 직교기저 `u,v`(법선은 노드 id 해시로 결정 → 다양한 기울기), 각속도 `omega` 저장.
  각속도는 **케플러 느낌**: `omega ≈ base / sqrt(r)` + 노드별 편차·방향, 상·하한 clamp.
- `onEngineTick` → `orbitTick()`: **BFS 순서(부모→자식)** 로 절대좌표 계산,
  `pos = 부모절대pos + r*(cosθ·u + sinθ·v)`, θ=`phase + omega*t`. 계산값을 `fx/fy/fz`(+`x/y/z`)로 핀.
  루트는 중심 고정. 작은 노드(스킬·축)도 자기 부모 주위를 돎(중첩).
- 필터 변경 시: `Graph.graphData(buildData()); markResettle(); Graph.d3ReheatSimulation();`
  → 재배치 후 `SETTLE_MS` 뒤 재공전.
- ⚠️ **함정:** init 시점에 `d3ReheatSimulation()` 을 조기 호출하면 `e.layout` 생성 전 `tickFrame` 이
  돌아 `Cannot read properties of undefined (reading 'tick')` 발생. init 에선 호출하지 말 것
  (`graphData()` 가 엔진을 시작함). 필터 변경 핸들러에선 layout 이 이미 있어 안전.

## 노드 드래그와 공전 공존
- `Graph.enableNodeDrag(true)`.
- `onNodeDrag` → `dragging.add(id)`, `onNodeDragEnd` → `dragging.delete(id)` 후 `recomputeOrbit(node)`.
- `orbitTick` 은 `dragging.has(id)` 인 노드를 건너뛰되 그 현재 좌표를 `absPos` 에 넣어
  **자식이 드래그 위치 기준으로 계속 공전**하게 한다.
- `recomputeOrbit`: 드롭 위치로 `r,u,v` 재계산 + `phase = -omega*tNow` → 튐 없이 이어서 공전.

## 공전 켜기/끄기 버튼
- `orbitPaused` 시 `orbitTick()` 생략 → 좌표 고정(제자리 정지).
- 재개 시 `orbitStart += now - pauseStart` 로 시간축 보정 → 튐 없음.

## 카메라 자동 회전 — ⚠️ TrackballControls 함정
- 이 그래프의 컨트롤은 **TrackballControls**(`Graph.controls().constructor` = `T_`)이며
  **`autoRotate` 를 지원하지 않는다.** `controls.autoRotate = true` 는 무동작(원래부터 버그).
- 해결: `onEngineTick` 에서 매 프레임 카메라를 `controls.target` 기준으로 직접 회전:
```js
const dx = cam.position.x - tg.x, dz = cam.position.z - tg.z;
const ang = 0.06 * dt;  // ~100초/바퀴
cam.position.x = tg.x + dx*Math.cos(ang) - dz*Math.sin(ang);
cam.position.z = tg.z + dx*Math.sin(ang) + dz*Math.cos(ang);
cam.lookAt(tg.x, tg.y, tg.z);
```
- `spinning` 플래그로 버튼·`pointerdown`(드래그 시작)과 연동. onEngineTick 은 controls.update() 이전에
  실행되어 TrackballControls 가 새 position 을 그대로 유지함.
- 용어: **자동 회전 = 카메라 이동**, **공전 = 노드 이동**. 서로 독립.

## 검증 (필수) — Playwright 헤드리스
사전 설치된 Chromium + 전역 Playwright 사용:
```js
import { chromium } from '/opt/node22/lib/node_modules/playwright/index.mjs';
// file:// 로 로드, waitForTimeout 로 settle/orbit 대기
// 확인: pageerror/console.error 0건, 점선 그룹 수(o.__dashes), 노드 이동량(공전),
//        실제 mouse down/move/up 드래그 이동, 공전·자동회전 토글 시 이동량 변화
```
- 최상위 `const` (Graph, controls, spinning, dragging, orbitMode…)는 classic script 전역이라
  `page.evaluate` 에서 그대로 접근 가능.
- 노드 화면좌표는 `Graph.graph2ScreenCoords(x,y,z)`.

## 작업 워크플로우
1. 지정 브랜치에서 작업, `index.html` 편집 → 다른 파일로 `cp` 동기화(`diff -q` 확인).
2. Playwright 로 헤드리스 검증(에러 0건 + 기능 수치 확인).
3. 명확한 한국어 커밋 → 푸시.
4. 사용자가 요청하면 PR 생성 → main 병합. PR 이 이미 병합됐으면 브랜치를 최신 main 에서 재시작.
