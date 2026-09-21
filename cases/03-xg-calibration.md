# 3. 넥슨은 xG를 주지 않는다

## 상황

전적 분석에 기대득점(xG)을 보여주고 싶었는데, 넥슨 Open API는 xG를 주지 않습니다. 대신 슛 좌표·종류·결과는 줍니다.

공개된 축구 xG 모델을 그대로 가져다 쓸 수도 있었지만, **FC 온라인은 전체 득점률이 40% 안팎**이라 실제 축구(약 10%)와 자릿수가 다릅니다. 그대로 쓰면 모든 슛이 과소평가됩니다.

## 판단

실제 경기 슛을 모아서 **거리 구간별 득점률**과 **슛 종류별 배율**을 직접 뽑기로 했습니다.

- **표본 모으기**: 경기 상세에는 양 팀 슛이 다 들어 있고 상대 닉네임도 나옵니다. 상대를 다음 조회 대상으로 넣어 넓혀가면(BFS) 한 경기 요청으로 양쪽 슛을 얻습니다.
- **호출 예산**: 개발 단계 키는 초당 5건, 하루 1,000건 제한이라 예산(BUDGET)으로 호출 수를 묶었습니다. 모은 원본은 저장해두고, 산식만 바꿀 때는 넥슨을 다시 부르지 않고 재계산합니다.
- **표본 노이즈 처리**: 거리가 멀어지는데 득점률이 오르는 구간이 생겼습니다(실측에서 두 구간이 뒤집힘). 값을 손으로 고치지 않고, 인접 구간을 표본 수로 가중 평균해 **단조 감소를 강제**했습니다(PAVA). 그래야 다음 재보정에도 같은 규칙이 적용됩니다.
- **슛 종류 배율**: 표본이 적은 종류는 배율이 튀기 쉬워서, 표본 수에 비례해 1 쪽으로 당겼습니다.
- 화면에는 **"자체 산출 추정치"**라고 밝힙니다.

측정 당시 모델: 유저 36명 · 경기 198 · 슛 2,142 / 골 849 (39.6%) · 검산 오차 1.66%

## 코드

`scripts/calibrate-xg.js` 단조 감소 강제

```js
/**
 * 단조 감소 강제 (PAVA): curve = [[거리, 득점률, 표본], ...]를 제자리에서 고친다.
 * 뒤 구간이 앞 구간보다 득점률이 높으면 표본 수로 가중 평균해 둘을 같은 값으로 만든다.
 * 거리가 멀수록 득점률이 낮다는 건 물리적으로 당연한데, 표본이 얇으면 뒤집힌다.
 */
function monotonize(curve) {
  for (let i = 1; i < curve.length; i++) {
    while (i > 0 && curve[i][1] > curve[i - 1][1]) {
      const n = curve[i - 1][2] + curve[i][2];
      const v = Number(((curve[i - 1][1] * curve[i - 1][2] + curve[i][1] * curve[i][2]) / n).toFixed(4));
      curve[i - 1][1] = v;
      curve[i][1] = v;
      i--;
    }
  }
  return curve;
}
```

`scripts/calibrate-xg.js` 구간 집계와 종류 배율

```js
function finish(shots, info) {
  if (shots.length < 300) throw new Error(`표본이 너무 적다(${shots.length}). BUDGET을 늘려 다시 돌릴 것`);

  // 거리 구간별 득점률
  const EDGES = [0, 0.05, 0.08, 0.11, 0.14, 0.17, 0.21, 0.26, 0.32, 1];
  const curve = [];
  for (let i = 0; i < EDGES.length - 1; i++) {
    const g = shots.filter((s) => {
      const d = dist(s);
      return d >= EDGES[i] && d < EDGES[i + 1];
    });
    if (g.length < 15) continue; // 표본이 얇은 구간은 버린다(옆 구간에서 보간된다)
    const goals = g.filter((s) => s.result === 3).length;
    curve.push([Number(((EDGES[i] + EDGES[i + 1]) / 2).toFixed(3)), Number((goals / g.length).toFixed(3)), g.length]);
  }

  // 거리가 멀어지는데 득점률이 올라가는 구간이 생긴다(표본 노이즈).
  // 값을 손으로 고치지 말 것. 여기서 규칙으로 눌러야 다음 재보정에도 같은 결과가 나온다.
  monotonize(curve);

  const goalsAll = shots.filter((s) => s.result === 3).length;
  const overall = goalsAll / shots.length;

  // 슛 종류 배율 (표본으로 축소)
  const K = 40;
  const typeMult = {};
  const byType = {};
  for (const s of shots) (byType[s.type] ??= []).push(s);
  for (const [t, g] of Object.entries(byType)) {
    if (Number(t) === 9) continue; // PK는 따로
    const rate = g.filter((s) => s.result === 3).length / g.length;
    const raw = rate / overall;
    typeMult[t] = Number((1 + (raw - 1) * (g.length / (g.length + K))).toFixed(3));
  }
  // ... 모델 저장 (생략)
}
```

`lib/recordStats.js` 슛 하나의 xG

```js
// 넥슨은 xG를 주지 않는다. 이건 우리가 만든 추정치다. 화면에도 "자체 산출"이라고 밝힌다.
// 계수는 코드에 박지 않는다 → data/xg-model.json (스크립트 산출물).
// 계수를 여기 베껴 두면 다음 재보정과 조용히 어긋난다.

export function shotXg(shot) {
  if (shot.type === 9) return XG_PENALTY;          // 페널티킥은 거리와 무관
  const d = Math.hypot(1 - shot.x, shot.y - 0.5);  // 골문 중앙까지 거리

  const last = XG_CURVE[XG_CURVE.length - 1];
  let base;
  if (d <= XG_CURVE[0][0]) base = XG_CURVE[0][1];
  else if (d >= last[0]) base = last[1];
  else {
    const i = XG_CURVE.findIndex(([dd]) => dd >= d);
    const [d0, v0] = XG_CURVE[i - 1];
    const [d1, v1] = XG_CURVE[i];
    base = v0 + ((v1 - v0) * (d - d0)) / (d1 - d0);  // 구간 사이는 선형 보간
  }

  const mult = XG_TYPE_MULT[shot.type] ?? 1;
  return Math.max(0.02, Math.min(0.95, base * mult));
}
```

## 설명할 수 있는 것

- **왜 공개 모델을 안 썼나**: 게임과 실축구의 득점률 자릿수가 다름
- **PAVA를 쓴 이유**: 표본이 얇은 구간의 역전을 사람이 손으로 고치면 다음 재보정 때 재현이 안 됨
- **종류 배율 축소(K=40)**: 표본 수가 적을수록 전체 평균 쪽으로 당겨 튀는 값을 막음
- **한계**: 표본이 2,142슛으로 작고 특정 유저 주변에서 넓혀간 표본이라 편향 가능성이 있음. 그래서 화면에 추정치임을 밝힘

원본 수집 스크립트의 API 호출부와 표본 시작점은 외부 계정 정보가 들어 있어 넣지 않았습니다.
