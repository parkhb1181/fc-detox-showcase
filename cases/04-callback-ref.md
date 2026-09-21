# 4. 토글 한 번에 255ms가 걸렸다

## 상황

상자를 열면 안에서 나온 보상이 줄줄이 펼쳐지는 화면이 있습니다. 보상을 50개쯤 열어두면 화면이 눈에 띄게 굼떴습니다.

실측(열린 보상 53행): **단순 토글 한 번 255ms, 선수팩 1개 열기 510ms 동안 화면이 멈춤.**

## 원인

보상 한 줄(`ChainReward`)은 `memo`로 감싸 두었는데도 매번 다시 그려지고 있었습니다.

- 보상을 여는 함수(`openInline`)는 `useChainOpen` 훅이 만들어 각 줄에 prop으로 내려줍니다.
- 이 훅을 쓰는 화면 5곳이 전부 콜백을 **인라인 화살표**로 넘기고 있었습니다.
  `useChainOpen({ onGain: (t) => setGainTril((g) => g + t) })`
- 인라인 함수는 렌더할 때마다 새로 만들어지니, 이걸 의존성에 넣은 `openInline`도 매번 새 함수가 됩니다.
- prop이 바뀌었으니 `memo`가 무력화되고, 화면이 한 번 갱신될 때마다 **이미 열린 보상 줄 전부**가 다시 그려졌습니다.

## 판단

호출하는 화면 5곳에서 각자 `useCallback`으로 고치는 방법도 있었지만, 그러면 **다음에 만드는 화면에서 또 터집니다.** 훅 안에서 규칙으로 막기로 했습니다.

- 콜백은 ref에 담고, 렌더마다 최신 값으로 갈아끼운다
- `openInline`의 의존성은 비워서 **함수 정체성을 고정**한다
- 실행할 때는 ref에서 꺼내 쓰니 **동작은 항상 최신 렌더의 콜백**을 따른다

## 코드

`lib/useChainOpen.js`

```js
export function useChainOpen({ onGain, onReveal } = {}) {
  // 콜백은 ref로 받고 openInline의 함수 정체성은 고정한다 (2026-09-10).
  //   호출부는 전부 인라인 화살표를 넘긴다 → deps에 그대로 두면 보드가 재렌더될 때마다
  //   openInline이 새 함수가 되고, 그게 onOpen prop으로 내려가 memo(ChainReward)를 통째로 무력화한다.
  //   실측(열린 보상 53행): 단순 토글 한 번 255ms · 선수팩 1개 열기 510ms 블로킹.
  //   호출부가 콜백을 useCallback으로 고정해 주길 기대하지 않는다. 보드 5곳이 전부 인라인이라
  //   "각자 고치기"는 다음 보드에서 또 터진다. 훅 안에서 규칙으로 막는다.
  const cbRef = useRef(null);
  cbRef.current = { onGain, onReveal };

  // productId 1개를 count개 개봉 → obtained[] 반환. 실패 시 throw (호출부가 재시도 UI 표시).
  const openInline = useCallback(
    async (productId, count = 1, opts = {}) => {
      const res = await fetch('/api/open', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ productId, count, ...(opts.noVal0 ? { noVal0: true } : {}) }),
      });
      if (!res.ok) throw new Error(`open failed: ${res.status}`);
      const data = await res.json();
      const obtained = data.obtained ?? [];
      // 하위 상자는 열기 전까지 가치를 합산하지 않는다 (이중 계상 방지)
      const gain = obtained.reduce((s, r) => s + rewardTril(r), 0);
      // 최신 콜백을 ref에서 읽는다: 정체성은 고정, 동작은 항상 이번 렌더의 것
      const { onGain: gainCb, onReveal: revealCb } = cbRef.current;
      if (gain > 0) gainCb?.(gain);
      revealCb?.(obtained);
      return obtained;
    },
    []
  );

  // ...
}
```

같은 훅에는 상자 안의 상자를 끝까지 여는 로직도 있습니다. 처음엔 하나씩 순서대로 열어서 느렸는데, 각 가지가 독립 추첨이라 순서가 결과에 영향을 주지 않는다는 걸 확인하고 **같은 깊이는 병렬로** 열게 바꿨습니다. 실패한 가지는 원래 상태로 두고 나머지는 계속 엽니다.

## 설명할 수 있는 것

- **memo가 왜 안 먹었나**: 참조가 바뀌는 prop 하나 때문에 얕은 비교가 매번 실패
- **왜 useCallback을 호출부에 강제하지 않았나**: 5곳을 고쳐도 6번째에서 재발. 규칙은 한 곳에 둬야 지켜짐
- **ref 패턴의 주의점**: 렌더 중에 ref를 갱신하므로, 콜백을 렌더 중이 아니라 이벤트·비동기 시점에만 호출해야 함
- **측정**: 수정 전 수치는 남아 있고, 수정 후 수치는 이 발췌 시점에 다시 재지 않았음
