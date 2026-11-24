# 設計書：n重振り子シミュレーター

## 1. 全体の作り

```text
[画面 frontend]  --条件(JSON)-->  [計算サーバー backend]
 Vite + TS                         FastAPI
 Canvasに描く   <--結果(JSON)--    physics/ で計算
```

1. 画面で n や初期条件を決め、`POST /api/simulate` に送る
2. 計算サーバーが指定の秒数ぶんを計算し、時刻ごとの状態をまとめて返す
3. 画面は受け取った結果を、時刻に合わせて再生する（再生中は通信しない）

## 2. 物理モデル

### 2.1 記号

| 記号 | 意味 | 単位 |
| --- | --- | --- |
| n | 振り子の数（1〜8） | - |
| θi | 節 i の棒が真下となす角度。反時計回りを正 | rad |
| ωi = dθi/dt | 節 i の角速度 | rad/s |
| αi = dωi/dt | 節 i の角加速度 | rad/s² |
| mi | 節 i の先にあるおもりの質量 | kg |
| li | 節 i の棒の長さ（棒の質量は0とする） | m |
| g | 重力加速度 | m/s² |
| μj = mj + mj+1 + … + mn | 節 j より先にある質量の合計 | kg |

i = 1 が支点に近い側、i = n が先端です。

**向きと単位の約束**（計算・API・画面のすべてで同じにする）

| 項目 | 約束 |
| --- | --- |
| 角度の基準 | 支点から真下の向きを θ = 0 とする |
| 角度の正の向き | 反時計回り。θ = +π/2 で棒は右を向く（x > 0） |
| 座標 | 支点が原点。x は右が正、y は上が正 |
| 単位 | SI単位（rad、rad/s、kg、m、s、m/s²）。度は使わない。API もラジアンでやりとりする |
| 角度の範囲 | 計算中は −π〜π に折り返さない（何周回ったかが分かるように、そのまま足していく） |
| 画面への変換 | Canvas は y が下向きなので、`canvasY = 原点Y − y × 拡大率`、`canvasX = 原点X + x × 拡大率` で変換する。画面で度を表示するときだけ、表示の直前に度に変える |

### 2.2 おもりの位置

支点を原点、右を x、上を y とします。

```math
x_i = \sum_{j=1}^{i} l_j \sin\theta_j,\qquad y_i = -\sum_{j=1}^{i} l_j \cos\theta_j
```

### 2.3 エネルギーとラグランジアン

おもり i の速さの2乗は、位置を微分して

```math
\dot x_i^2+\dot y_i^2=\sum_{j=1}^{i}\sum_{k=1}^{i} l_j l_k\,\omega_j\omega_k\cos(\theta_j-\theta_k)
```

となります。全部のおもりについて足すと、項 (j, k) は i ≥ max(j, k) のおもりすべてに現れるので、係数は μmax(j,k) になります。

```math
T=\frac12\sum_{j=1}^{n}\sum_{k=1}^{n}\mu_{\max(j,k)}\,l_j l_k\,\omega_j\omega_k\cos(\theta_j-\theta_k)
```

```math
V=\sum_{i=1}^{n} m_i g y_i=-g\sum_{j=1}^{n}\mu_j\,l_j\cos\theta_j
```

ラグランジアンは L = T − V、力学的エネルギーは E = T + V です（ガイドの式と同じ）。

### 2.4 運動方程式

各 j について、ラグランジュの運動方程式

```math
\frac{d}{dt}\frac{\partial L}{\partial \omega_j}-\frac{\partial L}{\partial \theta_j}=0
```

を計算します。途中の計算は次のとおりです（Mjk = μmax(j,k)、cjk = cos(θj − θk)、sjk = sin(θj − θk) と書く）。

```math
\frac{\partial T}{\partial \omega_j}=\sum_k M_{jk} l_j l_k\,\omega_k c_{jk}
```

```math
\frac{d}{dt}\frac{\partial T}{\partial \omega_j}=\sum_k M_{jk} l_j l_k\left[\alpha_k c_{jk}-\omega_k(\omega_j-\omega_k)s_{jk}\right]
```

```math
\frac{\partial T}{\partial \theta_j}=-\sum_k M_{jk} l_j l_k\,\omega_j\omega_k s_{jk},\qquad
\frac{\partial V}{\partial \theta_j}=g\,\mu_j l_j\sin\theta_j
```

これらを合わせると、ωjωk の項が打ち消し合って次の式になります。

```math
\sum_{k=1}^{n} M_{jk}l_j l_k\cos(\theta_j-\theta_k)\,\alpha_k
=-\sum_{k=1}^{n} M_{jk}l_j l_k\sin(\theta_j-\theta_k)\,\omega_k^2-g\,\mu_j l_j\sin\theta_j
```

### 2.5 行列の形

上の式を A(θ) α = b(θ, ω) とまとめます。

```math
A_{jk}=\mu_{\max(j,k)}\,l_j l_k\cos(\theta_j-\theta_k)
```

```math
b_j=-\sum_{k=1}^{n}\mu_{\max(j,k)}\,l_j l_k\sin(\theta_j-\theta_k)\,\omega_k^2-g\,\mu_j l_j\sin\theta_j
```

- A は対称行列で、運動エネルギーが T = ½ ωᵀAω と書けることから正定値です。そのため、いつでも α が一意に求まります
- α は `numpy.linalg.solve(A, b)` で求めます（逆行列は作らない）

### 2.6 確かめのための特別な場合

**n = 1（単振り子）**：A = m₁l₁²、b = −g m₁ l₁ sin θ₁ なので

```math
\alpha_1=-\frac{g}{l_1}\sin\theta_1
```

小さい角度では周期 T₀ = 2π√(l₁/g) になります。

**n = 2（二重振り子）**：よく知られた次の式と一致するはずです（Δ = θ₁ − θ₂、D = 2m₁ + m₂ − m₂cos 2Δ）。

```math
\alpha_1=\frac{-g(2m_1+m_2)\sin\theta_1-m_2 g\sin(\theta_1-2\theta_2)-2\sin\Delta\,m_2(\omega_2^2 l_2+\omega_1^2 l_1\cos\Delta)}{l_1 D}
```

```math
\alpha_2=\frac{2\sin\Delta\left(\omega_1^2 l_1(m_1+m_2)+g(m_1+m_2)\cos\theta_1+\omega_2^2 l_2 m_2\cos\Delta\right)}{l_2 D}
```

### 2.7 先端の加速度

先端（おもり n）の加速度は、位置を2回微分して求めます。

```math
\ddot x_n=\sum_{j=1}^{n} l_j\left(\alpha_j\cos\theta_j-\omega_j^2\sin\theta_j\right),\qquad
\ddot y_n=\sum_{j=1}^{n} l_j\left(\alpha_j\sin\theta_j+\omega_j^2\cos\theta_j\right)
```

大きさは |a| = √(ẍn² + ÿn²) です。これは空間に対する加速度なので、真下で止まっているときは0、自由落下しているときは g になります。
