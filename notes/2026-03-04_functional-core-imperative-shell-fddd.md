# Functional CoreとImperative Shellと関数型DDD

- **Date:** 2026-03-04
- **Keywords:** Functional Core, Imperative Shell, 関数型DDD, Needパターン, Read Model, state machine

## 議論の目的と背景
Functional Core / Imperative Shell / 関数型DDDを組み合わせるとき、
「ロジック途中で追加データ取得が必要になるケース」をどう扱うかを整理した。
特に、可読性を保ちながらShellにビジネス判断を漏らさない構造を目指した。

## 決定事項・重要なインサイト
- 業務ルールの適用順序はCoreに置く（Shellに置かない）。
- Shellの責務は「Needの実行（I/O）と結果の受け渡し」だけに限定する。
- `stock > 0` のような業務判定もCoreの `next/onResult` 側に閉じ込める。
- 基本は `BaseReadModel` を最初に作り、分岐で不足したデータだけ `Need*` で追加取得する。
- 分岐が増える場合は、巨大な1関数にせず「小さいstep関数」に分割したstate machine構成にする。

## 次のアクション・未解決の課題
- 実プロダクトに適用する際は、`Need` の粒度（何を1リクエストにするか）を先に決める。
- `BaseReadModel` に含める項目を定期的に見直し、Need発生率の高い項目を先読み側に寄せる。

## 具体コード例（更新版: 判定が複数続くケース）
以下は「判定が複数あり、条件に応じて必要データが変わる」例。
ポイントは、**次に何をするかの判断はすべてCoreが返す**こと。

```ts
// ===== Domain Types =====
type Cmd = {
  customerId: string;
  addressId: string;
  items: Array<{ sku: string; qty: number; unitPrice: number }>;
};

type BaseReadModel = {
  customerTier: "normal" | "gold";
  couponEligible: boolean;
  hasRestrictedItem: boolean;
  giftThreshold: number;         // 例: 30000
  freeShippingThreshold: number; // 例: 10000
};

type Draft = {
  subtotal: number;
  discount: number;
  shippingFee: number;
  giftAdded: boolean;
};

type Decision =
  | { kind: "Approved"; total: number; draft: Draft }
  | { kind: "Rejected"; reason: string };

// ===== Step ADT =====
type Step =
  | { type: "Done"; decision: Decision }
  | {
      type: "NeedFraudScore";
      input: { customerId: string };
      next: (r: { score: number }) => Step;
    }
  | {
      type: "NeedCreditLimit";
      input: { customerId: string };
      next: (r: { limit: number; used: number }) => Step;
    }
  | {
      type: "NeedKycStatus";
      input: { customerId: string };
      next: (r: { verified: boolean }) => Step;
    }
  | {
      type: "NeedGiftStock";
      input: { sku: string };
      next: (r: { available: number }) => Step; // stock > 0 判定はCore側
    }
  | {
      type: "NeedRemoteAreaFee";
      input: { addressId: string };
      next: (r: { remote: boolean }) => Step;
    };

type Ctx = { cmd: Cmd; base: BaseReadModel; draft: Draft };

// ===== Core (pure): small step functions =====
const sumSubtotal = (cmd: Cmd) => cmd.items.reduce((a, i) => a + i.qty * i.unitPrice, 0);

export function start(cmd: Cmd, base: BaseReadModel): Step {
  // 1) クーポン割引（先読み情報で確定できる部分）
  const subtotal = sumSubtotal(cmd);
  const discount = base.couponEligible
    ? Math.floor(subtotal * (base.customerTier === "gold" ? 0.12 : 0.05))
    : 0;

  const ctx: Ctx = {
    cmd,
    base,
    draft: { subtotal, discount, shippingFee: 0, giftAdded: false },
  };

  return stepRisk(ctx);
}

function stepRisk(ctx: Ctx): Step {
  const discounted = ctx.draft.subtotal - ctx.draft.discount;

  // 高額時だけ不正スコアを追加取得
  if (discounted >= 100000) {
    return {
      type: "NeedFraudScore",
      input: { customerId: ctx.cmd.customerId },
      next: (r) => afterFraud(ctx, r.score),
    };
  }

  return stepCompliance(ctx);
}

function afterFraud(ctx: Ctx, score: number): Step {
  if (score >= 90) {
    return { type: "Done", decision: { kind: "Rejected", reason: "high_fraud_risk" } };
  }

  const discounted = ctx.draft.subtotal - ctx.draft.discount;

  // より高額なら与信情報を追加取得
  if (discounted >= 200000) {
    return {
      type: "NeedCreditLimit",
      input: { customerId: ctx.cmd.customerId },
      next: (r) => afterCredit(ctx, r.limit, r.used),
    };
  }

  return stepCompliance(ctx);
}

function afterCredit(ctx: Ctx, limit: number, used: number): Step {
  const discounted = ctx.draft.subtotal - ctx.draft.discount;
  if (used + discounted > limit) {
    return { type: "Done", decision: { kind: "Rejected", reason: "credit_limit_exceeded" } };
  }
  return stepCompliance(ctx);
}

function stepCompliance(ctx: Ctx): Step {
  // 規制対象商品がある場合だけKYC確認
  if (!ctx.base.hasRestrictedItem) return stepGift(ctx);

  return {
    type: "NeedKycStatus",
    input: { customerId: ctx.cmd.customerId },
    next: (r) => (r.verified ? stepGift(ctx) : { type: "Done", decision: { kind: "Rejected", reason: "kyc_required" } }),
  };
}

function stepGift(ctx: Ctx): Step {
  const discounted = ctx.draft.subtotal - ctx.draft.discount;

  // 閾値到達時だけギフト在庫を確認
  if (discounted < ctx.base.giftThreshold) return stepShipping(ctx);

  return {
    type: "NeedGiftStock",
    input: { sku: "GIFT-001" },
    next: (r) => {
      const nextCtx: Ctx = {
        ...ctx,
        draft: { ...ctx.draft, giftAdded: r.available > 0 }, // ビジネス判定はCore側
      };
      return stepShipping(nextCtx);
    },
  };
}

function stepShipping(ctx: Ctx): Step {
  const discounted = ctx.draft.subtotal - ctx.draft.discount;

  if (discounted >= ctx.base.freeShippingThreshold) {
    return { type: "Done", decision: { kind: "Approved", total: discounted, draft: ctx.draft } };
  }

  // 地域情報で送料を決める必要がある場合のみ追加取得
  return {
    type: "NeedRemoteAreaFee",
    input: { addressId: ctx.cmd.addressId },
    next: (r) => {
      const shippingFee = r.remote ? 1200 : 600;
      const draft = { ...ctx.draft, shippingFee };
      const total = discounted + shippingFee;
      return { type: "Done", decision: { kind: "Approved", total, draft } };
    },
  };
}

// ===== Shell (imperative): no business decisions =====
type Ports = {
  loadBaseReadModel(cmd: Cmd): Promise<BaseReadModel>;
  fetchFraudScore(customerId: string): Promise<{ score: number }>;
  fetchCreditLimit(customerId: string): Promise<{ limit: number; used: number }>;
  fetchKycStatus(customerId: string): Promise<{ verified: boolean }>;
  fetchGiftStock(sku: string): Promise<{ available: number }>;
  fetchRemoteAreaFee(addressId: string): Promise<{ remote: boolean }>;
};

export async function approveOrder(cmd: Cmd, ports: Ports): Promise<Decision> {
  const base = await ports.loadBaseReadModel(cmd); // 先読み
  let step: Step = start(cmd, base);

  while (step.type !== "Done") {
    switch (step.type) {
      case "NeedFraudScore":
        step = step.next(await ports.fetchFraudScore(step.input.customerId));
        break;
      case "NeedCreditLimit":
        step = step.next(await ports.fetchCreditLimit(step.input.customerId));
        break;
      case "NeedKycStatus":
        step = step.next(await ports.fetchKycStatus(step.input.customerId));
        break;
      case "NeedGiftStock":
        step = step.next(await ports.fetchGiftStock(step.input.sku));
        break;
      case "NeedRemoteAreaFee":
        step = step.next(await ports.fetchRemoteAreaFee(step.input.addressId));
        break;
    }
  }

  return step.decision;
}
```


## 補足コード例: Shellが少しビジネスフローを担当するパターン
可読性優先で、Shellが「どの順で何を取りに行くか」を持つ形。
ただし条件式そのもの（適用可否・閾値判定）はCore関数に寄せる。

```ts
// ===== Core =====
type Customer = { id: string; tier: "normal" | "vip" };
type Coupon = { rate: number } | null;
type Line = { sku: string; qty: number; unitPrice: number };

type PriceBreakdown = {
  subtotal: number;
  couponDiscount: number;
  bulkDiscount: number;
  shippingFee: number;
  giftAdded: boolean;
  total: number;
};

export const isCouponEligible = (c: Customer) => c.tier === "vip";
export const shouldCheckGiftStock = (subtotal: number) => subtotal >= 8000;
export const canAddGift = (stockQty: number, requiredQty = 1) => stockQty >= requiredQty;

export const calcSubtotal = (line: Line) => line.qty * line.unitPrice;

export function calcFinal(input: {
  line: Line;
  coupon: Coupon;
  isRemote: boolean;
  giftAdded: boolean;
}): PriceBreakdown {
  const subtotal = calcSubtotal(input.line);
  const couponDiscount = input.coupon ? Math.floor(subtotal * input.coupon.rate) : 0;
  const afterCoupon = subtotal - couponDiscount;

  const bulkDiscount = input.line.qty >= 3 ? Math.floor(afterCoupon * 0.15) : 0;
  const discounted = afterCoupon - bulkDiscount;

  const shippingFee = discounted >= 5000 ? 0 : input.isRemote ? 1200 : 600;

  return {
    subtotal,
    couponDiscount,
    bulkDiscount,
    shippingFee,
    giftAdded: input.giftAdded,
    total: discounted + shippingFee,
  };
}

// ===== Shell =====
async function quoteOrder(
  cmd: { customerId: string; addressId: string; line: Line },
  ports: {
    findCustomer(id: string): Promise<Customer>;
    findCoupon(customerId: string): Promise<Coupon>;
    giftStock(sku: string): Promise<number>;
    isRemoteArea(addressId: string): Promise<boolean>;
  }
): Promise<PriceBreakdown> {
  const customer = await ports.findCustomer(cmd.customerId);

  // フロー制御はShellで担当（このパターンで許容する漏れ）
  const coupon = isCouponEligible(customer)
    ? await ports.findCoupon(customer.id)
    : null;

  const subtotal = calcSubtotal(cmd.line);
  const giftAdded = shouldCheckGiftStock(subtotal)
    ? canAddGift(await ports.giftStock("GIFT-001"))
    : false;

  const isRemote = await ports.isRemoteArea(cmd.addressId);

  // 金額計算・判定ロジックはCore
  return calcFinal({
    line: cmd.line,
    coupon,
    isRemote,
    giftAdded,
  });
}
```

### このパターンの使いどころ
- まずは実装速度と読みやすさを優先したい時。
- ただし、分岐が増えてShellが肥大化してきたら、NeedベースのCore主導遷移に戻す。


## 比較表: Core主導遷移 vs Shell薄漏れ

| 観点 | Core主導遷移（Need + next） | Shell薄漏れ（フローをShellで一部判断） |
|---|---|---|
| ルールの一元性 | 高い（判定と遷移がCoreに集約） | 中程度（条件式はCore化しても流れはShellに残る） |
| 可読性（初見） | 低〜中（慣れが必要） | 高い（直線的に読める） |
| 変更容易性（複雑化時） | 高い（step分割で拡張しやすい） | 中〜低（分岐増でShell肥大化） |
| テスト容易性 | 高い（Coreを純粋テストしやすい） | 中（Shell統合テストの比重が増える） |
| バグ混入リスク | 低め（責務境界が明確） | 中（Shellに業務判断が漏れやすい） |
| 実装速度（初期） | 中 | 高 |
| 向いている場面 | 長期運用・高頻度な要件変更 | 小〜中規模・まず動かす段階 |

### 使い分けの目安
- ドメインルールが増え続ける見込みなら、最初からCore主導遷移を選ぶ。
- まずは短期で価値検証したいならShell薄漏れで始め、分岐増加をトリガーにCoreへ引き上げる。
- どちらの場合も「条件式と閾値」はCoreに置くのを最低ラインにする。

## 関連ノート
- なし
