# 动态规划（DP）学习笔记

> 目标不是记模板，而是能从题目中推导出：**状态 → 转移 → 初始化 → 计算顺序 → 答案 → 复杂度**。
>
> 本笔记记录已经学习过的内容。每学完一个新模型，在文末继续追加“题型、代码、易错点、复盘结果”。

---

## 0. 通用解题框架

面对一道 DP 题，按下面的顺序思考：

1. `dp` 的一个状态究竟表示什么？边界是否清晰？
2. 最后一步（最后一次选择）有哪些可能？
3. 做完最后一步，问题变成哪个更小的子问题？
4. 如何写转移；它为什么不漏解、不重解？
5. 初始状态是什么？不可达状态该填什么？
6. 当前状态依赖哪些旧状态，因而该按什么顺序计算？
7. 最终答案是一个状态，还是所有状态的最大/最小值？
8. 时间、空间复杂度是多少；能否滚动压缩？

常见状态语义不要混淆：

```text
dp[i]       ：前 i 个元素 / 以第 i 个元素结尾（必须看题意区分）
dp[i][j]    ：两个前缀、一个区间、或两个维度的资源状态
dp[capacity]：容量恰好为 capacity 或容量不超过 capacity（初始化不同）
```

---

## 1. 一维线性 DP

### 1.1 识别方式

问题按顺序推进，当前位置通常只依赖前面有限个位置；可以从“最后一步”或“第 `i` 个决策”推导。

### 1.2 基础模板

```python
dp = [0] * (n + 1)
dp[0] = base

for i in range(1, n + 1):
    dp[i] = transition(dp, i)
```

若只依赖固定数量的前项，通常可压缩为若干变量。

### 1.3 例：爬楼梯

**状态**：`dp[i]` 表示到达第 `i` 级台阶的方法数。

**最后一步**：从 `i - 1` 跨一步，或从 `i - 2` 跨两步。

```text
dp[i] = dp[i - 1] + dp[i - 2]
dp[0] = 1, dp[1] = 1
```

```python
def climb_stairs(n: int) -> int:
    prev2, prev1 = 1, 1  # dp[0], dp[1]
    for _ in range(2, n + 1):
        prev2, prev1 = prev1, prev1 + prev2
    return prev1
```

**易错点**：`dp[0] = 1` 表示“什么都不做”的一种方式，是计数题常见的有效初始状态。

### 1.4 例：打家劫舍

**状态**：`dp[i]` 表示只考虑前 `i` 间房屋（下标 `0 ~ i-1`）时能获得的最高金额。

```text
不偷第 i 间：dp[i - 1]
偷第 i 间  ：dp[i - 2] + nums[i - 1]
dp[i] = max(dp[i - 1], dp[i - 2] + nums[i - 1])
```

```python
def rob(nums: list[int]) -> int:
    prev2 = prev1 = 0
    for money in nums:
        prev2, prev1 = prev1, max(prev1, prev2 + money)
    return prev1
```

### 1.5 例：最大子数组和（Kadane）

**状态**：`dp[i]` 表示**必须以** `nums[i]` 结尾的最大连续子数组和。

```text
dp[i] = max(nums[i], dp[i - 1] + nums[i])
```

```python
def max_sub_array(nums: list[int]) -> int:
    best_end = answer = nums[0]
    for x in nums[1:]:
        best_end = max(x, best_end + x)
        answer = max(answer, best_end)
    return answer
```

**为什么答案是所有状态的最大值？** 状态限定“以 `i` 结尾”，而最优子数组的结尾位置未知。

---

## 2. 背包 DP

### 2.1 总览：先问四个问题

```text
1. 每种物品最多选几次？一次 / 无限次 / 有上限
2. 目标是什么？最大值 / 最小值 / 可达性 / 方案数
3. 是否必须恰好装满？这决定初始化
4. 若是计数，物品在外层还是容量在外层？这决定组合还是排列
```

### 2.2 0/1 背包

每件物品最多选一次。

**状态**：`dp[j]` 为容量不超过 `j` 时的最大价值。

**转移**：对重量 `w`、价值 `v` 的物品，选或不选：

```text
dp[j] = max(dp[j], dp[j - w] + v)
```

```python
def zero_one_knapsack(items: list[tuple[int, int]], capacity: int) -> int:
    dp = [0] * (capacity + 1)
    for weight, value in items:
        for j in range(capacity, weight - 1, -1):  # 必须倒序
            dp[j] = max(dp[j], dp[j - weight] + value)
    return dp[capacity]
```

**为什么倒序？** 本轮更新 `dp[j]` 时读取的 `dp[j - weight]` 仍是“未处理当前物品”的旧状态，因此当前物品不会被重复选择。

### 2.3 完全背包

每种物品可选无限次。

```python
def complete_knapsack(items: list[tuple[int, int]], capacity: int) -> int:
    dp = [0] * (capacity + 1)
    for weight, value in items:
        for j in range(weight, capacity + 1):  # 必须正序
            dp[j] = max(dp[j], dp[j - weight] + value)
    return dp[capacity]
```

**为什么正序？** `dp[j - weight]` 可以是本轮刚更新过的状态，代表已多次使用当前物品。

### 2.4 多重背包

每种物品最多 `count` 次。

朴素方法是枚举当前物品拿多少个；更常用的基础优化是二进制拆分，例如数量 `13` 拆为 `1 + 2 + 4 + 6` 个 0/1 物品：

```python
def multiple_knapsack(items: list[tuple[int, int, int]], capacity: int) -> int:
    dp = [0] * (capacity + 1)
    for weight, value, count in items:
        k = 1
        while count:
            take = min(k, count)
            bundled_weight = take * weight
            bundled_value = take * value
            for j in range(capacity, bundled_weight - 1, -1):
                dp[j] = max(dp[j], dp[j - bundled_weight] + bundled_value)
            count -= take
            k *= 2
    return dp[capacity]
```

### 2.5 恰好装满 vs. 不超过容量

```python
# 不超过容量的最大价值：任何容量都可以什么都不放
dp = [0] * (capacity + 1)

# 恰好装满容量的最大价值：除 0 外均不可达
neg_inf = float('-inf')
dp = [0] + [neg_inf] * capacity
```

不可达状态的约定：最大化填 `-inf`，最小化填 `+inf`，可达性填 `False`。

### 2.6 凑和类问题的映射

```text
数字 / 硬币       → 物品重量
target            → 背包容量
每个数字一次      → 0/1 背包
每个数字无限次    → 完全背包
```

#### 例：分割等和子集（0/1 可达性）

```python
def can_partition(nums: list[int]) -> bool:
    total = sum(nums)
    if total % 2:
        return False

    target = total // 2
    dp = [False] * (target + 1)
    dp[0] = True
    for x in nums:
        for j in range(target, x - 1, -1):
            dp[j] = dp[j] or dp[j - x]
    return dp[target]
```

#### 例：零钱兑换（完全背包、最小值）

```python
def coin_change(coins: list[int], amount: int) -> int:
    inf = float('inf')
    dp = [0] + [inf] * amount
    for coin in coins:
        for j in range(coin, amount + 1):
            dp[j] = min(dp[j], dp[j - coin] + 1)
    return -1 if dp[amount] == inf else dp[amount]
```

#### 例：组合总和 IV（完全背包、排列计数）

容量放外层，表示“最后一步选哪个数”，不同顺序会分别计数。

```python
def combination_sum4(nums: list[int], target: int) -> int:
    dp = [0] * (target + 1)
    dp[0] = 1
    for total in range(1, target + 1):
        for x in nums:
            if x <= total:
                dp[total] += dp[total - x]
    return dp[target]
```

### 2.7 计数：组合与排列

```python
# 完全背包：组合数（[1, 2] 和 [2, 1] 视为同一种）
for coin in coins:
    for j in range(coin, target + 1):
        dp[j] += dp[j - coin]

# 完全背包：排列数（[1, 2] 和 [2, 1] 分别计数）
for j in range(1, target + 1):
    for coin in coins:
        if coin <= j:
            dp[j] += dp[j - coin]
```

---

## 3. 序列 DP：最长递增子序列（LIS）

题意：从数组中选取元素（不要求连续），使其严格递增且长度最大。

### 3.1 `O(n²)` DP 解法

**状态**：

```text
dp[i] = 必须以 nums[i] 结尾的最长严格递增子序列长度
```

**转移**：若 `j < i` 且 `nums[j] < nums[i]`，则可把 `nums[i]` 接在一个以 `nums[j]` 结尾的序列后：

```text
dp[i] = max(dp[i], dp[j] + 1)
```

**初始化**：每个元素本身就是长度为 `1` 的递增子序列，所以 `dp[i] = 1`。

```python
def length_of_lis_dp(nums: list[int]) -> int:
    if not nums:
        return 0

    dp = [1] * len(nums)
    for i in range(len(nums)):
        for j in range(i):
            if nums[j] < nums[i]:
                dp[i] = max(dp[i], dp[j] + 1)
    return max(dp)
```

**答案**：`max(dp)`，因为 LIS 的结尾下标未知。

**复杂度**：时间 `O(n²)`，空间 `O(n)`。

### 3.2 贪心 + 二分解法：`O(n log n)`

维护递增数组 `tails`：

```text
tails[length - 1] = 所有长度为 length 的递增子序列中，最小的可能结尾值
```

贪心依据：长度相同的情况下，结尾越小，未来越容易接上新的元素；所以只需要保留最小结尾。

对当前数字 `x`：

1. 在有序的 `tails` 内找第一个 `>= x` 的位置；
2. 若找到了，就用 `x` 替换它，使该长度的结尾更小；
3. 若不存在（即 `x` 比所有结尾都大），追加 `x`，长度增加一。

```python
from bisect import bisect_left


def length_of_lis(nums: list[int]) -> int:
    tails = []
    for x in nums:
        pos = bisect_left(tails, x)  # 二分找第一个 >= x 的位置
        if pos == len(tails):
            tails.append(x)
        else:
            tails[pos] = x
    return len(tails)
```

示例 `[3, 1, 2, 5, 4]`：

```text
3 → [3]
1 → [1]       # 长度 1 的更优结尾
2 → [1, 2]
5 → [1, 2, 5]
4 → [1, 2, 4] # 长度 3 的更优结尾
```

`tails` 是“各长度的最优结尾记录”，**不保证自身就是原数组中的一条完整 LIS**；本算法求长度没有问题，若要求恢复具体序列，需要额外记录前驱下标。

### 3.3 `bisect_left` 与 `bisect_right`

```python
from bisect import bisect_left, bisect_right

arr = [1, 2, 2, 2, 5]
bisect_left(arr, 2)   # 1，第一个 >= 2 的位置
bisect_right(arr, 2)  # 4，第一个 > 2 的位置
```

```text
最长严格递增：相等元素不能相接 → bisect_left（第一个 >= x）
最长非递减：相等元素可以相接   → bisect_right（第一个 > x）
```

---

## 4. 序列 DP：最长公共子序列（LCS）

题意：给定两个字符串（或数组），求两者最长公共**子序列**的长度。子序列不要求元素连续。

```text
text1 = "abcde"
text2 = "ace"
答案 = 3  # "ace"
```

### 4.1 状态与边界

```text
dp[i][j] = text1 的前 i 个字符与 text2 的前 j 个字符的 LCS 长度
```

使用“前 `i` 个字符”而非“下标到 `i`”，能自然多开一行一列：当任一前缀为空时，答案为 `0`。

```text
dp[0][j] = 0
dp[i][0] = 0
```

当前格比较的末尾字符是 `text1[i - 1]` 和 `text2[j - 1]`。

### 4.2 状态转移

```text
末尾字符相等：
dp[i][j] = dp[i - 1][j - 1] + 1

末尾字符不同：
dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])
```

末尾不同，两个字符不能同时成为同一个公共子序列的最后一个元素；任何最优解至少舍弃其中一个末尾字符，所以“舍弃 `text1` 末尾”与“舍弃 `text2` 末尾”覆盖了全部情况。

末尾相同，可以把该字符接到两个更短前缀的 LCS 后，得到左上角 `+ 1`。它不比上方、左方差：一个前缀最多只比去掉其最后一个字符多贡献一个元素。

### 4.3 完整代码

```python
def longest_common_subsequence(text1: str, text2: str) -> int:
    m, n = len(text1), len(text2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if text1[i - 1] == text2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1] + 1
            else:
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])

    return dp[m][n]
```

**复杂度**：时间 `O(m × n)`，空间 `O(m × n)`。

### 4.4 空间压缩（理解即可）

第 `i` 行只依赖第 `i - 1` 行和当前行左侧，可以把二维数组压成一维。更新时需额外保存更新前的左上角值。

```python
def longest_common_subsequence_compressed(text1: str, text2: str) -> int:
    # 让 text2 更短，可将空间降为 O(min(m, n))
    if len(text1) < len(text2):
        text1, text2 = text2, text1

    dp = [0] * (len(text2) + 1)
    for c1 in text1:
        diagonal = 0  # 更新前的 dp[j - 1]，即上一行左上角
        for j, c2 in enumerate(text2, start=1):
            up = dp[j]  # 更新前的 dp[j]，即上一行同列
            if c1 == c2:
                dp[j] = diagonal + 1
            else:
                dp[j] = max(dp[j], dp[j - 1])
            diagonal = up
    return dp[-1]
```

### 4.5 LCS 复盘清单

- [x] 知道 `dp[i][j]` 是两个前缀的 LCS，而非两个下标“结尾”的答案。
- [x] 知道第 `i`、`j` 个字符实际是 `text1[i - 1]`、`text2[j - 1]`。
- [x] 末尾相等：左上角 `+ 1`。
- [x] 末尾不同：上方、左方取最大；二者覆盖所有最优解。
- [x] 空字符串对应的第 `0` 行和第 `0` 列都初始化为 `0`。

---

## 5. 序列 DP：编辑距离

题意：将 `word1` 转换为 `word2`，每次可执行一次插入、删除或替换，求最少操作次数。

```text
horse → rorse  # 替换 h 为 r
rorse → rose   # 删除 r
rose  → ros    # 删除 e

"horse" → "ros" 的编辑距离为 3
```

### 5.1 状态与边界

```text
dp[i][j] = 将 word1 的前 i 个字符转换为 word2 的前 j 个字符，
           所需的最少操作次数
```

两段前缀长度可以不同；DP 正是在求如何通过插入、删除把长度和内容逐步调整到一致。

```text
dp[i][0] = i  # 目标为空：删除 word1 的前 i 个字符
dp[0][j] = j  # 来源为空：插入 word2 的前 j 个字符
```

当前要比较的末尾字符是 `word1[i - 1]`、`word2[j - 1]`。比较它们的目的，是判断两者能否无成本对应为最终字符串的末尾，而不是要求两个前缀长度相同。

### 5.2 状态转移：从最后一次操作推导

如果末尾字符相等，无须对它们做操作：

```text
dp[i][j] = dp[i - 1][j - 1]
```

末尾字符不同时，最后一次操作恰为以下三种之一：

```text
删除 word1[i - 1]：
word1[:i] → word1[:i - 1] → word2[:j]
dp[i][j] = dp[i - 1][j] + 1

插入 word2[j - 1]：
word1[:i] → word2[:j - 1] → word2[:j]
dp[i][j] = dp[i][j - 1] + 1

替换 word1[i - 1] 为 word2[j - 1]：
word1[:i - 1] → word2[:j - 1] → word2[:j]
dp[i][j] = dp[i - 1][j - 1] + 1
```

取三种方案的最小值：

```python
dp[i][j] = min(
    dp[i - 1][j] + 1,      # 删除
    dp[i][j - 1] + 1,      # 插入
    dp[i - 1][j - 1] + 1,  # 替换
)
```

特别注意插入的方向：本题统一按 `word1 → word2` 操作。最后插入的是 `word2[j - 1]`；插入前，`word1[:i]` 已经变成 `word2[:j - 1]`，所以是 `dp[i][j - 1] + 1`，不是 `dp[i - 1][j] + 1`。

### 5.3 完整代码

```python
def min_distance(word1: str, word2: str) -> int:
    m, n = len(word1), len(word2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(m + 1):
        dp[i][0] = i
    for j in range(n + 1):
        dp[0][j] = j

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if word1[i - 1] == word2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1]
            else:
                dp[i][j] = min(
                    dp[i - 1][j] + 1,      # 删除
                    dp[i][j - 1] + 1,      # 插入
                    dp[i - 1][j - 1] + 1,  # 替换
                )

    return dp[m][n]
```

**复杂度**：时间 `O(m × n)`，空间 `O(m × n)`；可像 LCS 一样压缩为 `O(n)` 空间。

### 5.4 编辑距离复盘清单

- [x] 能准确说出操作方向是 `word1 → word2`。
- [x] 理解两个前缀长度不同是正常状态，插入/删除会处理长度差。
- [x] 空目标只能删除，因此 `dp[i][0] = i`；空来源只能插入，因此 `dp[0][j] = j`。
- [x] 末尾相等：左上角不加代价。
- [x] 末尾不同：删除取上方、插入取左方、替换取左上角，均加 `1`。
- [x] 插入对应 `dp[i][j - 1] + 1`，因为插入前目标的末尾字符尚未构造。

---

## 6. 区间 DP

### 6.1 识别与统一框架

适用信号：问题作用于一个连续区间，且一个大区间的答案能由更小的连续区间构成。常见状态为：

```text
dp[l][r] = 区间 [l, r] 上的最优答案
```

如果状态依赖更短区间，必须按区间长度递增计算：

```python
for length in range(2, n + 1):
    for l in range(n - length + 1):
        r = l + length - 1
        # 计算 dp[l][r]
```

一条高频思路：若正向过程会改变相邻关系或难以描述，改为枚举**最后一次操作**；最后操作发生时，子区间通常都已处理完成，边界和代价更容易确定。

### 6.2 合并石子

每次合并相邻两堆，代价为两堆石子数之和；求合并全部石子的最小总代价。

**状态**：`dp[l][r]` 表示将 `stones[l:r+1]` 合并为一堆的最小代价。

**边界**：`dp[l][l] = 0`，一堆石子不用合并。

**最后一次合并**：设最后分割点为 `k`，先将 `[l, k]` 和 `[k+1, r]` 分别合为一堆，再合并两堆：

```text
dp[l][r] = min(dp[l][k] + dp[k + 1][r] + sum(stones[l:r+1]))
           (l <= k < r)
```

```python
def min_merge_cost(stones: list[int]) -> int:
    n = len(stones)
    if n <= 1:
        return 0

    prefix = [0] * (n + 1)
    for i, x in enumerate(stones):
        prefix[i + 1] = prefix[i] + x

    def interval_sum(l: int, r: int) -> int:
        return prefix[r + 1] - prefix[l]

    dp = [[0] * n for _ in range(n)]
    for length in range(2, n + 1):
        for l in range(n - length + 1):
            r = l + length - 1
            dp[l][r] = float('inf')
            for k in range(l, r):
                dp[l][r] = min(
                    dp[l][r],
                    dp[l][k] + dp[k + 1][r] + interval_sum(l, r),
                )
    return dp[0][n - 1]
```

**复杂度**：时间 `O(n³)`，空间 `O(n²)`。

### 6.3 矩阵链乘法

若 `dims = [d0, d1, ..., dn]`，则第 `i` 个矩阵（从 `0` 开始）维度为 `dims[i] × dims[i+1]`。目标是寻找括号划分，使标量乘法次数最少。

**状态**：`dp[l][r]` 表示第 `l` 到第 `r` 个矩阵相乘的最小计算量。

```text
dp[l][l] = 0
dp[l][r] = min(dp[l][k] + dp[k+1][r] + dims[l] * dims[k+1] * dims[r+1])
```

最后一项是左右两段结果相乘的代价；它与合并石子的骨架相同，只是区间合并代价换成了三维乘积。

```python
def matrix_chain_min_cost(dims: list[int]) -> int:
    matrix_count = len(dims) - 1
    if matrix_count <= 1:
        return 0

    dp = [[0] * matrix_count for _ in range(matrix_count)]
    for length in range(2, matrix_count + 1):
        for l in range(matrix_count - length + 1):
            r = l + length - 1
            dp[l][r] = float('inf')
            for k in range(l, r):
                dp[l][r] = min(
                    dp[l][r],
                    dp[l][k] + dp[k + 1][r]
                    + dims[l] * dims[k + 1] * dims[r + 1],
                )
    return dp[0][matrix_count - 1]
```

### 6.4 戳气球

正向戳气球会不断改变相邻关系；选择“区间内最后戳哪个气球”后，两端边界固定。

```python
def max_coins(nums: list[int]) -> int:
    points = [1] + nums + [1]
    n = len(points)
    dp = [[0] * n for _ in range(n)]

    # dp[l][r]：戳破开区间 (l, r) 内所有气球的最大收益
    for length in range(2, n):
        for l in range(n - length):
            r = l + length
            for k in range(l + 1, r):
                dp[l][r] = max(
                    dp[l][r],
                    dp[l][k] + dp[k][r] + points[l] * points[k] * points[r],
                )
    return dp[0][n - 1]
```

这里 `l`、`r` 不属于被戳的开区间，而是保留到最后的边界。若 `k` 最后戳破，它的相邻气球正好是 `l`、`r`，收益才可确定。

### 6.5 最长回文子序列

```text
dp[l][r] = s[l:r+1] 中最长回文子序列的长度
dp[i][i] = 1

s[l] == s[r]：dp[l][r] = dp[l+1][r-1] + 2
s[l] != s[r]：dp[l][r] = max(dp[l+1][r], dp[l][r-1])
```

```python
def longest_palindrome_subseq(s: str) -> int:
    n = len(s)
    dp = [[0] * n for _ in range(n)]
    for i in range(n):
        dp[i][i] = 1

    for length in range(2, n + 1):
        for l in range(n - length + 1):
            r = l + length - 1
            if s[l] == s[r]:
                dp[l][r] = dp[l + 1][r - 1] + 2
            else:
                dp[l][r] = max(dp[l + 1][r], dp[l][r - 1])
    return dp[0][n - 1]
```

### 6.6 区间 DP 复盘清单

- [x] 能识别连续区间拆分而成的大问题，并定义 `dp[l][r]`。
- [x] 理解状态依赖更短区间，所以要按区间长度从短到长计算。
- [x] 会枚举分割点 `k`，处理 `[l, k]` 与 `[k+1, r]`。
- [x] 能从“最后一次操作”推导合并石子、矩阵链、戳气球的转移。
- [x] 知道戳气球采用开区间，是为了固定最后戳破气球的相邻边界。
- [x] 会按两端是否相等推导回文子序列转移。

---

## 7. 树形 DP

树形 DP 的状态定义在节点或子树上。父节点通常依赖子节点的信息，所以使用 DFS 的**后序过程**：先处理孩子，再计算当前节点。

```text
线性 DP：状态依赖前面的元素
区间 DP：状态依赖更短区间
树形 DP：节点状态依赖子节点状态
```

### 7.1 树上选择：打家劫舍 III

每个节点有一个价值，父子节点不能同时选择，求可选节点价值和的最大值。

```text
dp[u][0] = 不选 u 时，以 u 为根子树的最大收益
dp[u][1] = 选 u 时，以 u 为根子树的最大收益
```

```text
选 u：孩子不能选
dp[u][1] = value[u] + Σ dp[child][0]

不选 u：每个孩子独立取“选/不选”的较大值
dp[u][0] = Σ max(dp[child][0], dp[child][1])
```

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right


def rob_tree(root: TreeNode | None) -> int:
    def dfs(node: TreeNode | None) -> tuple[int, int]:
        # 返回 (不选 node 的收益, 选 node 的收益)
        if node is None:
            return 0, 0

        left_skip, left_take = dfs(node.left)
        right_skip, right_take = dfs(node.right)

        skip = max(left_skip, left_take) + max(right_skip, right_take)
        take = node.val + left_skip + right_skip
        return skip, take

    return max(dfs(root))
```

父节点状态一旦确定，各子树之间互不影响；例如左右孩子是兄弟关系，不存在父子冲突，所以可独立取最优答案后相加。

**复杂度**：时间 `O(n)`，递归栈空间 `O(h)`，`h` 为树高。

### 7.2 树的直径：返回“可向父节点延伸的信息”

树的直径是任意两节点之间的最长路径（按边数计）。

```text
depth(u) = 从 u 向下走到某个后代的最长路径边数
depth(u) = max(depth(left), depth(right)) + 1
```

经过当前节点的最佳完整路径，会连接左、右子树向下的最长单链：

```text
through_u = depth(left) + depth(right)
```

```python
def diameter_of_binary_tree(root: TreeNode | None) -> int:
    answer = 0

    def dfs(node: TreeNode | None) -> int:
        nonlocal answer
        if node is None:
            return 0

        left_depth = dfs(node.left)
        right_depth = dfs(node.right)
        answer = max(answer, left_depth + right_depth)

        # 只能向父节点返回一条向下链，不能同时带左右两条
        return max(left_depth, right_depth) + 1

    dfs(root)
    return answer
```

关键区分：DFS 返回的是“当前节点能向父节点提供的最长单链”；全局答案则可在当前节点连接左右两条链。

### 7.3 换根 DP：所有节点作为根的答案

题意：无权树中，求每个节点到所有其他节点的距离和。

先任选 `0` 为根，第一遍 DFS 求：

```text
size[u]  = u 的子树节点数
answer[0] = 节点 0 到所有节点的距离和
```

把根从父节点 `u` 换到孩子 `v`：

- `v` 子树的 `size[v]` 个节点距离各减少 `1`；
- 其他 `n - size[v]` 个节点距离各增加 `1`。

```text
answer[v] = answer[u] - size[v] + (n - size[v])
```

```python
def sum_of_distances_in_tree(n: int, edges: list[list[int]]) -> list[int]:
    graph = [[] for _ in range(n)]
    for u, v in edges:
        graph[u].append(v)
        graph[v].append(u)

    size = [1] * n
    answer = [0] * n

    def dfs1(u: int, parent: int, depth: int) -> None:
        answer[0] += depth
        for v in graph[u]:
            if v == parent:
                continue
            dfs1(v, u, depth + 1)
            size[u] += size[v]

    def dfs2(u: int, parent: int) -> None:
        for v in graph[u]:
            if v == parent:
                continue
            answer[v] = answer[u] - size[v] + (n - size[v])
            dfs2(v, u)

    dfs1(0, -1, 0)
    dfs2(0, -1)
    return answer
```

换根 DP 的固定流程：先选一个根并计算基础答案、子树信息；然后将根沿父子边移动，逐项分析哪些节点的贡献增加或减少。

### 7.4 树形 DP 复盘清单

- [x] 知道树形 DP 的计算顺序通常是 DFS 后序。
- [x] 会用 `dp[u][0/1]` 表达“选/不选节点”的子树最优值。
- [x] 理解父节点状态确定后，各子树通常可以独立合并。
- [x] 能区分递归返回给父节点的信息与在当前节点更新的完整答案。
- [x] 能推导换根时子树与非子树节点贡献的增减。

---

## 8. 下一阶段：状态压缩 DP

用二进制 bitmask 表示一个小规模集合；常见状态为 `dp[mask]` 或 `dp[mask][i]`，适用于集合选择、匹配、旅行商等问题。
