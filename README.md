# 艦これ 砲擊順序機制解析

**[▶ 開啟計算器](https://dube116.github.io/kancolle-shelling-order/)** ｜ 本頁面僅供演示，**不會維護**

*KanColle shelling order: same-range ties are resolved by a non-transitive random comparator inside
PHP 5's `zend_qsort`, producing a non-uniform permutation with an exact closed form.*

---

## 1. 機制

砲擊順序 = 對「本階段可砲擊艦清單」依**射程遞減**排序，**同射程時比較器回傳公平硬幣**。

```php
usort($ships, function ($a, $b) {
    if ($a->range != $b->range) return $b->range - $a->range;  // 射程遞減
    return mt_rand(0, 1);                                       // 同射程 → 亂數
});
```

關鍵在於比較器**不具傳遞性** —— 同一對艦每次比較都重擲，不是一個一致的全序。這讓排序結果
**不是均勻亂序**，而是所用排序實作的指紋。

PHP 5 的 `usort` 走 `zend_qsort`：

```c
/* php-src PHP-5.6  ext/standard/array.c */
zend_hash_sort(Z_ARRVAL_P(array), zend_qsort, php_array_user_compare, 1);

/* php-src PHP-5.6  Zend/zend_qsort.c */
offset = (end - begin) >> 1;
_zend_qsort_swap(begin, begin + (offset - (offset % siz)), siz);   /* 中點樞紐 → swap 到 begin */

for (; seg1 < seg2 && compare(begin, seg1) > 0; seg1 += siz);
for (; seg2 >= seg1 && compare(seg2, begin) > 0; seg2 -= siz);

_zend_qsort_swap(begin, seg2, siz);                                /* 樞紐定位 */
```

`php_array_user_compare` 會把回傳值正規化成 `-1 / 0 / 1`，而 `zend_qsort` 兩個掃描迴圈都只測 `> 0`。
因此平手時回傳 `0` 與 `-1` **行為完全等價**，能確定的只有 `P(回傳 > 0) = 1/2`。

## 2. 哪些艦會進入排序陣列

陣列長度與成員直接決定樞紐位置，所以這層規則跟演算法一樣重要。

| 情況 | 是否進入陣列 |
|---|---|
| 開戰前被撃沉 | ✗ 排除 |
| FCF 撤退 | ✗ 排除 |
| 未搭載爆戰的空母 | ✗ 排除 |
| 先制對潛階段中不能先制對潛的艦 | ✗ 排除 |
| 開戰前大破的空母 | ✓ 計入（但輪到時打不出來）|
| 砲擊戰中不能對潛的艦 | ✓ 計入 |

被排除的艦會讓陣列變短，**樞紐位置跟著移動** —— 同一支艦隊在砲擊戰與先制對潛階段的順序結構不同。

## 3. 由機制推出的性質

**只有相對射程有影響。** 比較器只看差值的符號，絕對射程值不可能進入結果。

**樞紐艦。** 排序樞紐是陣列中點 `floor((n-1)/2)`（6 艦時是站位 #3，7 艦時是 #4）。
它先手的機率恰好是 `2⁻ⁿ`，順位分佈有 closed form：

$$P(\text{樞紐排第 }k) = \frac{[x^{k-1}]\,(1+x)^{n-3}(1+5x+2x^2)}{2^n}\qquad(n\ge3)$$

```
n=3 → 1, 5, 2                 / 8
n=4 → 1, 6, 7, 2              / 16
n=5 → 1, 7, 13, 9, 2          / 32
n=6 → 1, 8, 20, 22, 11, 2     / 64
n=7 → 1, 9, 28, 42, 33, 13, 2 / 128
```

**6 艦全同射程時各艦的先手機率：**

| 站位 | #1 | #2 | #3 | #4 | #5 | #6 |
|---|---|---|---|---|---|---|
| 先手率 | 921/4096 | 129/1024 | **1/64** | 2097/8192 | 6843/32768 | 5529/32768 |
| | 22.5% | 12.6% | **1.6%** | 25.6% | 20.9% | 16.9% |

**平手群不可分解。** 同一組平手艦的內部順序會隨艦隊其餘射程配置劇變：

| 樣式 | 123 | 132 | 213 | 231 | 312 | 321 |
|---|---|---|---|---|---|---|
| `AAA` | .375 | .125 | .063 | .063 | .125 | .250 |
| `AAAB` | .250 | .188 | .094 | .031 | .188 | .250 |
| `AAABB` | .188 | .250 | .188 | .250 | **.031** | .094 |

任何「每個平手群獨立擲骰」的模型都是錯的。

**先手率的上限只由「同射程艦數 m」決定**，與其餘艦如何分層無關：

| m | 1 | 2 | 3 | 4 | 5 | 6 |
|---|---|---|---|---|---|---|
| 上限 | 1 | 3/4 | 1/2 | 23/64 | 81/256 | 2097/8192 |

**站位有免費的差距。** 兩艘射程完全相同的艦放站位 `#2+#3` 是 **75/25**，放 `#1+#3` 卻是 50/50。
6 艦全同射程時，站位 #4 與 #3 的先手率差 **16 倍**。

## 4. 實作

```js
/** PHP 5 zend_qsort。cmp(x, y) > 0 表示 x 排在 y 之後。 */
function zendQsort(a, cmp) {
  if (a.length < 2) return;
  const stack = [[0, a.length - 1]];
  while (stack.length) {
    let [begin, end] = stack.pop();
    while (begin < end) {
      const mid = begin + ((end - begin) >> 1);
      const t = a[begin]; a[begin] = a[mid]; a[mid] = t;      // 樞紐 → begin
      let seg1 = begin + 1, seg2 = end;
      for (;;) {
        while (seg1 < seg2 && cmp(a[begin], a[seg1]) > 0) seg1++;
        while (seg2 >= seg1 && cmp(a[seg2], a[begin]) > 0) seg2--;
        if (seg1 >= seg2) break;
        const u = a[seg1]; a[seg1] = a[seg2]; a[seg2] = u;
        seg1++; seg2--;
      }
      const v = a[begin]; a[begin] = a[seg2]; a[seg2] = v;    // 樞紐定位
      if (seg2 - begin <= end - seg2) {
        if (seg2 + 1 < end) stack.push([seg2 + 1, end]);
        if (seg2 === begin) break;
        end = seg2 - 1;
      } else {
        if (begin + 1 < seg2) stack.push([begin, seg2 - 1]);
        begin = seg2 + 1;
      }
    }
  }
}

/** ships 只傳「本階段參與排序」的艦，依站位順序。getRange 需為 max(艦本體, 各裝備)。 */
function shellingOrder(ships, getRange, random = Math.random) {
  const order = ships.slice();
  zendQsort(order, (x, y) => {
    const rx = getRange(x), ry = getRange(y);
    if (rx !== ry) return ry - rx;          // 射程長者排前面
    return random() < 0.5 ? 1 : -1;         // 每次呼叫都要重擲，不可快取
  });
  return order;
}
```

三個會弄壞它的地雷：

1. **改用 `Array.prototype.sort`** —— V8 跑 TimSort，分佈完全不同。
2. **把平手決定變成一致的**（給每艦快取亂數 key、或 memoize 比較器）—— 非傳遞性正是分佈的來源，
   一致的順序會退化成接近均勻洗牌。
3. **動到 floor 中點 / 樞紐先 swap 到 `begin` / 兩個掃描的 `> 0`** —— 任一項改動都會讓分佈偏離數個數量級。

## 5. 驗證

對 TsunDB 抽樣資料，**零擬合參數**：

| 檢定 | 結果 |
|---|---|
| 4,037 個射程樣式 / 2,063,819 筆完整順序觀測 | **支撐違反 0** |
| 550 個樣式 / 419 萬筆邊際觀測，第 1 攻擊層 | χ² = 511.4 / 490 dof = **1.044** |
| 全部攻擊層 | χ² = 5669.6 / 5728 dof = **0.990** |
| 其他排序演算法（插入 / 選擇 / 氣泡 / 合併 / Lomuto / PHP 7 `zend_sort`） | 差 2–4 個數量級 |
| 機械窮舉的 32 種 quicksort 變體 | 僅 **1 種**存活，次佳差 25 倍 |
| 平手硬幣偏度 | q = 0.5006 ± 0.0003（公平）|

「支撐違反 0」= 206 萬筆觀測中沒有任何一筆落在模型機率為 0 的排列上。混合射程樣式的合法排列
集合被嚴格限制（`AAABBB` 只有 36 種而非 720 種），這是比 χ² 更硬的一種檢定。

## 6. 資料來源

觀測資料為 **TsunDB** eventbattle 抽樣，由 *KanColle Shelling Range Tie Randomness (2024-02)* 整理發布
（[原始資料集](https://docs.google.com/spreadsheets/d/1grAtIhlm2YMN9iyvniJA-j1YNCC-SCnA8q6WUi-Ctpg/)）。
本 repo 不轉載該資料集。

---

⚠ **本頁面與本 repo 僅供演示，不會維護** —— 不保證後續更新、修正，或與遊戲改版同步。
