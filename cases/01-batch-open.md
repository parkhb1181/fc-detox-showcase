# 1. 함수 호출이 무료 한도를 넘었다

## 상황

상자·선수팩을 여러 개 한꺼번에 여는 기능이 있습니다. 런칭 직후 Vercel Function 호출이 Hobby 요금제 한도를 넘었습니다.

원인은 두 군데였습니다.

- 개봉 API(`/api/open`)가 한 번에 **최대 10개**만 받게 되어 있어서, 999개를 열면 클라이언트가 10개씩 쪼개 **100번** 호출했습니다.
- 보관함 화면은 API가 여러 개를 받을 수 있는데도 **1개씩** 부르고 있었습니다. 상자 999개면 요청 999번입니다.

## 판단

추첨은 서버가 시드 기반으로 N회 독립 실행합니다. 그래서 **한 번에 999개를 보내도 1개씩 999번 보낸 것과 결과·확률이 같습니다.** 쪼갤 이유가 없었습니다.

1. API 상한을 10에서 999로 올린다
2. 보관함은 같은 상품끼리 묶어 한 요청으로 보낸다. 요청 수가 개봉 개수에서 **상품 종류 수**(보통 2~5)로 떨어진다
3. 화면에서는 먼저 목록에서 빼고(낙관적 업데이트), 실패하면 되돌려서 다시 누를 수 있게 한다

## 코드

`app/api/open/route.js`

```js
// body: { productId: string, count: number(1~999) }
//   상한 999 (2026-07-18): 종전 10 → 999. 클라가 999개를 10개씩 쪼개 100번 호출하던 걸
//   1번으로 합쳐 Vercel 함수 호출을 최대 1/100로 줄인다(런칭 트래픽 대응). 추첨은 999회라도 1초 미만.
export async function POST(request) {
  let body;
  try {
    body = await request.json();
  } catch {
    return NextResponse.json({ error: 'invalid body' }, { status: 400 });
  }

  const { productId, count, noVal0 } = body || {};
  const n = Number(count);

  if (!productId || !Number.isInteger(n) || n < 1 || n > 999) {
    return NextResponse.json({ error: 'productId 필수, count는 1~999 정수' }, { status: 400 });
  }

  // ... 상품 종류에 따라 시드 기반 추첨 (생략)
}
```

`components/OpenScreen.jsx` 보관함 개봉

```jsx
/**
 * 보관함 개봉: 같은 상품은 한 요청으로 묶는다 (2026-07-18 런칭 긴급패치).
 *
 * 왜: /api/open은 count 1~999를 받는데 여기서 1개씩 부르고 있었다.
 *   상자 999개를 열면 999 요청 → Vercel Function 호출이 폭발했다
 *   (실측: 하루 /api/open 1.4M회, Hobby 한도 1M의 165%).
 *   productId로 묶어 count=N을 한 번에 보내면 요청 수 = 상품 종류 수로 떨어진다(보통 2~5).
 * 추첨 자체는 서버가 N회 독립 실행하므로 결과·확률은 1개씩 열 때와 동일하다.
 */
const openPoolItems = useCallback(
  async (items) => {
    const byProduct = new Map();
    for (const it of items) {
      if (openingPoolRef.current.has(it.__uid)) continue; // ref 잠금: 이중 추첨 방지
      openingPoolRef.current.add(it.__uid);
      const arr = byProduct.get(it.productId);
      if (arr) arr.push(it);
      else byProduct.set(it.productId, [it]);
    }
    if (byProduct.size === 0) return;

    // 낙관적 제거: 결과는 히어로·획득 내역이 보여준다
    const taken = new Set([...byProduct.values()].flat().map((x) => x.__uid));
    setPool((p) => p.filter((x) => !taken.has(x.__uid)));

    await Promise.all(
      [...byProduct.entries()].map(async ([productId, group]) => {
        try {
          // 서버 상한이 999다. 넘겨 보내면 400이 떨어지고 그 그룹이 통째로 되돌아온다
          // (실측: ×2997을 한 번에 보내 실패 → 목록에 그대로 남았다). 999씩 쪼갠다.
          const got = [];
          for (let left = group.length; left > 0; left -= API_MAX) {
            const n = Math.min(API_MAX, left);
            got.push(...(await openInline(productId, n))); // N개를 1요청으로
          }
          const kids = toPoolUnits(got);
          if (kids.length) setPool((p) => [...p, ...kids]);
        } catch {
          setPool((p) => [...p, ...group]); // 실패: 되돌려 재시도 가능하게
          for (const g of group) openingPoolRef.current.delete(g.__uid);
        }
      })
    );
  },
  [openInline, setPool, toPoolUnits]
);
```

## 설명할 수 있는 것

- **왜 묶어도 되는가**: 서버 추첨이 개수만큼 독립 실행되고, 시드가 상품·요청 단위로 갈리기 때문
- **왜 999를 넘기면 다시 쪼개는가**: 상한을 올린 뒤 2,997개를 한 번에 보냈다가 400으로 그룹 전체가 되돌아온 걸 보고 추가
- **실패 처리**: 목록에서 먼저 빼고, 실패한 그룹만 되돌리면서 잠금을 풀어 사용자가 다시 누를 수 있게 함. 자동 재시도는 넣지 않았음
- **이중 추첨 방지**: 같은 항목이 두 번 요청되지 않게 ref로 잠금
