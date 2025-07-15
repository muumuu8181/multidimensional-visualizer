# アルゴリズム詳細

## 概要

Multidimensional Visualizer で使用される次元削減および可視化アルゴリズムの詳細な説明です。各アルゴリズムの数学的基礎、実装方法、適用場面について解説します。

## 次元削減アルゴリズム

### 1. 主成分分析 (PCA: Principal Component Analysis)

#### 数学的基礎

PCAは、データの分散を最大化する方向（主成分）を見つける線形次元削減手法です。

**基本原理:**
1. データの共分散行列を計算
2. 共分散行列の固有値・固有ベクトルを求める
3. 固有値の大きい順に固有ベクトルを選択
4. 選択した固有ベクトルに投影

**数学的定式化:**

データ行列 X ∈ ℝⁿˣᵈ（n個のサンプル、d次元）に対して：

1. **データの中心化:**
   ```
   X̃ = X - μ
   μ = (1/n) Σᵢ xᵢ
   ```

2. **共分散行列の計算:**
   ```
   C = (1/(n-1)) X̃ᵀ X̃
   ```

3. **固有値分解:**
   ```
   C = V Λ Vᵀ
   ```
   ここで、V は固有ベクトル行列、Λ は固有値の対角行列

4. **次元削減:**
   ```
   Y = X̃ Vₖ
   ```
   Vₖ は上位k個の固有ベクトル

#### 実装例

```python
import numpy as np
from typing import Tuple

class PCA:
    def __init__(self, n_components: int = 2):
        self.n_components = n_components
        self.components_ = None
        self.explained_variance_ = None
        self.explained_variance_ratio_ = None
        self.mean_ = None
    
    def fit(self, X: np.ndarray) -> 'PCA':
        """PCAを学習"""
        # データの中心化
        self.mean_ = np.mean(X, axis=0)
        X_centered = X - self.mean_
        
        # 共分散行列の計算
        cov_matrix = np.cov(X_centered.T)
        
        # 固有値分解
        eigenvalues, eigenvectors = np.linalg.eigh(cov_matrix)
        
        # 固有値の降順でソート
        idx = eigenvalues.argsort()[::-1]
        eigenvalues = eigenvalues[idx]
        eigenvectors = eigenvectors[:, idx]
        
        # 上位k個の成分を選択
        self.components_ = eigenvectors[:, :self.n_components].T
        self.explained_variance_ = eigenvalues[:self.n_components]
        self.explained_variance_ratio_ = self.explained_variance_ / np.sum(eigenvalues)
        
        return self
    
    def transform(self, X: np.ndarray) -> np.ndarray:
        """データを変換"""
        X_centered = X - self.mean_
        return X_centered @ self.components_.T
    
    def fit_transform(self, X: np.ndarray) -> np.ndarray:
        """学習と変換を同時実行"""
        return self.fit(X).transform(X)
    
    def inverse_transform(self, X_transformed: np.ndarray) -> np.ndarray:
        """元の空間に逆変換"""
        return X_transformed @ self.components_ + self.mean_
```

#### 長所・短所

**長所:**
- 計算が高速
- 線形変換なので解釈しやすい
- 寄与率により情報量を定量化可能
- ノイズ除去効果

**短所:**
- 線形関係しか捉えられない
- 外れ値に敏感
- 全ての特徴量が数値である必要

#### 適用場面
- 高次元データの前処理
- ノイズ除去
- データ圧縮
- 特徴量の重要度分析

### 2. t-SNE (t-Distributed Stochastic Neighbor Embedding)

#### 数学的基礎

t-SNEは、高次元空間での近傍関係を低次元空間で保持する非線形次元削減手法です。

**基本原理:**
1. 高次元空間で各点ペアの類似度を計算
2. 低次元空間での対応する類似度を定義
3. 両者の分布の差（KLダイバージェンス）を最小化

**数学的定式化:**

1. **高次元空間での類似度（ガウシアン分布）:**
   ```
   pⱼ|ᵢ = exp(-||xᵢ - xⱼ||² / 2σᵢ²) / Σₖ≠ᵢ exp(-||xᵢ - xₖ||² / 2σᵢ²)
   
   pᵢⱼ = (pⱼ|ᵢ + pᵢ|ⱼ) / 2n
   ```

2. **低次元空間での類似度（t分布）:**
   ```
   qᵢⱼ = (1 + ||yᵢ - yⱼ||²)⁻¹ / Σₖ≠ₗ (1 + ||yₖ - yₗ||²)⁻¹
   ```

3. **目的関数（KLダイバージェンス）:**
   ```
   C = Σᵢ Σⱼ pᵢⱼ log(pᵢⱼ / qᵢⱼ)
   ```

4. **勾配:**
   ```
   ∂C/∂yᵢ = 4 Σⱼ (pᵢⱼ - qᵢⱼ)(yᵢ - yⱼ)(1 + ||yᵢ - yⱼ||²)⁻¹
   ```

#### 実装例

```python
import numpy as np
from scipy.spatial.distance import pdist, squareform

class TSNE:
    def __init__(self, n_components: int = 2, perplexity: float = 30.0, 
                 learning_rate: float = 200.0, n_iter: int = 1000):
        self.n_components = n_components
        self.perplexity = perplexity
        self.learning_rate = learning_rate
        self.n_iter = n_iter
    
    def _compute_pairwise_affinities(self, X: np.ndarray) -> np.ndarray:
        """高次元空間での類似度行列を計算"""
        n = X.shape[0]
        P = np.zeros((n, n))
        
        # 各点についてガウシアン分布のσを調整
        for i in range(n):
            # 二分探索でperplexityに合うσを見つける
            sigma = self._binary_search_sigma(X, i, self.perplexity)
            
            # 条件付き確率を計算
            diff = X - X[i]
            distances_sq = np.sum(diff**2, axis=1)
            distances_sq[i] = np.inf  # 自分自身を除外
            
            P[i] = np.exp(-distances_sq / (2 * sigma**2))
            P[i] /= np.sum(P[i])
        
        # 対称化
        P = (P + P.T) / (2 * n)
        P = np.maximum(P, 1e-12)  # 数値安定性
        
        return P
    
    def _binary_search_sigma(self, X: np.ndarray, i: int, 
                           target_perplexity: float) -> float:
        """perplexityに合うσを二分探索で見つける"""
        sigma_min, sigma_max = 1e-20, 1e20
        
        for _ in range(50):  # 最大50回の反復
            sigma = (sigma_min + sigma_max) / 2
            
            # 現在のσでのperplexityを計算
            diff = X - X[i]
            distances_sq = np.sum(diff**2, axis=1)
            distances_sq[i] = np.inf
            
            prob = np.exp(-distances_sq / (2 * sigma**2))
            prob /= np.sum(prob)
            
            entropy = -np.sum(prob * np.log2(prob + 1e-12))
            perplexity = 2**entropy
            
            if abs(perplexity - target_perplexity) < 1e-5:
                break
            elif perplexity > target_perplexity:
                sigma_max = sigma
            else:
                sigma_min = sigma
        
        return sigma
    
    def fit_transform(self, X: np.ndarray) -> np.ndarray:
        """t-SNEを実行"""
        n = X.shape[0]
        
        # 高次元での類似度を計算
        P = self._compute_pairwise_affinities(X)
        
        # 低次元での初期化（PCAで初期化すると良い）
        Y = np.random.randn(n, self.n_components) * 1e-4
        
        # 最適化
        momentum = 0.5
        velocity = np.zeros_like(Y)
        
        for iter in range(self.n_iter):
            # 勾配計算
            gradient = self._compute_gradient(P, Y)
            
            # モメンタム更新
            velocity = momentum * velocity - self.learning_rate * gradient
            Y += velocity
            
            # 中心化
            Y -= np.mean(Y, axis=0)
            
            # モメンタムの調整
            if iter == 250:
                momentum = 0.8
        
        return Y
    
    def _compute_gradient(self, P: np.ndarray, Y: np.ndarray) -> np.ndarray:
        """勾配を計算"""
        n, dim = Y.shape
        
        # 低次元での類似度を計算
        distances_sq = np.sum((Y[:, None, :] - Y[None, :, :])**2, axis=2)
        Q = (1 + distances_sq)**(-1)
        np.fill_diagonal(Q, 0)
        Q /= np.sum(Q)
        Q = np.maximum(Q, 1e-12)
        
        # 勾配計算
        PQ_diff = P - Q
        gradient = np.zeros_like(Y)
        
        for i in range(n):
            diff = Y[i] - Y
            gradient[i] = 4 * np.sum(
                (PQ_diff[i, :, None] * diff.T * Q[i, :]).T, axis=0
            )
        
        return gradient
```

#### 長所・短所

**長所:**
- 非線形関係を捉えられる
- 局所的な構造をよく保持
- クラスター構造が明確に見える

**短所:**
- 計算時間が長い（O(n²)）
- パラメータ（perplexity）に敏感
- 大域的な構造は保持されない
- 再現性が低い（乱数に依存）

#### 適用場面
- 高次元データの可視化
- クラスター分析
- 異常検知
- 画像・テキストデータの探索

### 3. UMAP (Uniform Manifold Approximation and Projection)

#### 数学的基礎

UMAPは位相的データ解析に基づく次元削減手法で、t-SNEより高速で大域的構造も保持します。

**基本原理:**
1. データの局所的な多様体構造をグラフで近似
2. ファジィ単体複体を構築
3. 低次元空間で同様の構造を最適化

**数学的定式化:**

1. **局所距離の計算:**
   ```
   ρᵢ = min{d(xᵢ, xⱼ) : d(xᵢ, xⱼ) > 0}
   σᵢ = arg max{σ : Σⱼ exp(-(max(0, d(xᵢ, xⱼ) - ρᵢ))/σ)) = log₂(k)}
   ```

2. **高次元グラフの重み:**
   ```
   wᵢⱼ = exp(-(max(0, d(xᵢ, xⱼ) - ρᵢ))/σᵢ)
   ```

3. **対称化:**
   ```
   w'ᵢⱼ = wᵢⱼ + wⱼᵢ - wᵢⱼ × wⱼᵢ
   ```

4. **低次元での重み:**
   ```
   qᵢⱼ = (1 + a × ||yᵢ - yⱼ||₂^(2b))⁻¹
   ```

5. **損失関数（クロスエントロピー）:**
   ```
   C = Σᵢⱼ w'ᵢⱼ log(qᵢⱼ) + (1 - w'ᵢⱼ) log(1 - qᵢⱼ)
   ```

#### 実装例

```python
import numpy as np
from sklearn.neighbors import NearestNeighbors
from scipy.optimize import minimize

class UMAP:
    def __init__(self, n_components: int = 2, n_neighbors: int = 15,
                 min_dist: float = 0.1, n_epochs: int = 200):
        self.n_components = n_components
        self.n_neighbors = n_neighbors
        self.min_dist = min_dist
        self.n_epochs = n_epochs
    
    def fit_transform(self, X: np.ndarray) -> np.ndarray:
        """UMAPを実行"""
        n = X.shape[0]
        
        # 近傍グラフを構築
        graph = self._build_fuzzy_simplicial_set(X)
        
        # 低次元埋め込みを初期化
        embedding = self._initialize_embedding(X)
        
        # 最適化
        embedding = self._optimize_embedding(graph, embedding)
        
        return embedding
    
    def _build_fuzzy_simplicial_set(self, X: np.ndarray) -> np.ndarray:
        """ファジィ単体複体を構築"""
        n = X.shape[0]
        
        # k近傍を見つける
        nbrs = NearestNeighbors(n_neighbors=self.n_neighbors + 1)
        nbrs.fit(X)
        distances, indices = nbrs.kneighbors(X)
        
        # 自分自身を除去
        distances = distances[:, 1:]
        indices = indices[:, 1:]
        
        # 局所距離パラメータを計算
        rho = distances[:, 0]  # 最近傍への距離
        
        # σの計算（二分探索）
        sigma = np.zeros(n)
        for i in range(n):
            sigma[i] = self._smooth_knn_distance(distances[i], self.n_neighbors)
        
        # グラフの重みを計算
        graph = np.zeros((n, n))
        for i in range(n):
            for j_idx, j in enumerate(indices[i]):
                dist = max(0, distances[i, j_idx] - rho[i])
                graph[i, j] = np.exp(-dist / sigma[i])
        
        # 対称化（ファジィ和）
        graph = graph + graph.T - np.multiply(graph, graph.T)
        
        return graph
    
    def _smooth_knn_distance(self, distances: np.ndarray, k: int,
                           bandwidth: float = 1.0) -> float:
        """滑らかなk近傍距離を計算"""
        target = np.log2(k) * bandwidth
        
        def objective(sigma):
            if sigma <= 0:
                return float('inf')
            sum_exp = np.sum(np.exp(-distances / sigma))
            return abs(sum_exp - target)
        
        result = minimize(objective, x0=1.0, method='L-BFGS-B',
                         bounds=[(1e-20, None)])
        return result.x[0]
    
    def _initialize_embedding(self, X: np.ndarray) -> np.ndarray:
        """埋め込みを初期化（spectral embedding使用）"""
        from sklearn.manifold import SpectralEmbedding
        
        spec = SpectralEmbedding(n_components=self.n_components)
        return spec.fit_transform(X)
    
    def _optimize_embedding(self, graph: np.ndarray, 
                          embedding: np.ndarray) -> np.ndarray:
        """確率的勾配降下で埋め込みを最適化"""
        n = embedding.shape[0]
        
        # パラメータa, bを計算
        a, b = self._find_ab_params(self.min_dist)
        
        # 負サンプリング用の確率分布
        degrees = np.array(graph.sum(axis=1)).flatten()
        neg_sample_prob = degrees / degrees.sum()
        
        for epoch in range(self.n_epochs):
            # 学習率の調整
            alpha = 1.0 - epoch / self.n_epochs
            
            # 正サンプル（グラフのエッジ）
            for i in range(n):
                for j in range(i + 1, n):
                    if graph[i, j] > 0:
                        self._update_embedding_positive(
                            embedding, i, j, graph[i, j], a, b, alpha
                        )
            
            # 負サンプル
            for _ in range(int(n * self.n_neighbors / 4)):
                i = np.random.randint(n)
                j = np.random.choice(n, p=neg_sample_prob)
                if i != j and graph[i, j] == 0:
                    self._update_embedding_negative(
                        embedding, i, j, a, b, alpha
                    )
        
        return embedding
    
    def _find_ab_params(self, min_dist: float) -> tuple:
        """パラメータa, bを計算"""
        def curve(x, a, b):
            return 1.0 / (1.0 + a * x**(2 * b))
        
        from scipy.optimize import curve_fit
        
        x = np.linspace(0, 3 * min_dist, 300)
        y = np.where(x < min_dist, 1.0, np.exp(-(x - min_dist)))
        
        popt, _ = curve_fit(curve, x, y)
        return popt[0], popt[1]
    
    def _update_embedding_positive(self, embedding: np.ndarray, i: int, j: int,
                                 weight: float, a: float, b: float, alpha: float):
        """正サンプルでの勾配更新"""
        diff = embedding[i] - embedding[j]
        dist_sq = np.sum(diff**2)
        
        if dist_sq > 0:
            factor = -2 * a * b * (dist_sq**(b - 1)) / (1 + a * dist_sq**b)**2
            grad = factor * weight * alpha * diff
            
            embedding[i] += grad
            embedding[j] -= grad
    
    def _update_embedding_negative(self, embedding: np.ndarray, i: int, j: int,
                                 a: float, b: float, alpha: float):
        """負サンプルでの勾配更新"""
        diff = embedding[i] - embedding[j]
        dist_sq = np.sum(diff**2)
        
        if dist_sq > 0:
            factor = 2 * b / ((0.001 + dist_sq) * (1 + a * dist_sq**b))
            grad = factor * alpha * diff
            
            embedding[i] += grad
            embedding[j] -= grad
```

#### 長所・短所

**長所:**
- t-SNEより高速
- 大域的構造も保持
- パラメータに対してロバスト
- 新しいデータポイントの埋め込みが可能

**短所:**
- 理論的基礎が複雑
- パラメータの調整が必要
- 実装が複雑

### 4. 多次元尺度法 (MDS: Multi-Dimensional Scaling)

#### 数学的基礎

MDSは、高次元空間での距離関係を低次元空間で保持する手法です。

**基本原理:**
距離行列 D から座標を復元する問題として定式化

**古典的MDS（主座標分析）:**

1. **距離行列の二乗:**
   ```
   D² = [dᵢⱼ²]
   ```

2. **中心化行列:**
   ```
   H = I - (1/n)11ᵀ
   ```

3. **グラム行列:**
   ```
   G = -½HD²H
   ```

4. **固有値分解:**
   ```
   G = UΛUᵀ
   ```

5. **座標の復元:**
   ```
   X = U₍ₖ₎Λ₍ₖ₎^(1/2)
   ```

#### 実装例

```python
import numpy as np
from sklearn.metrics import euclidean_distances

class MDS:
    def __init__(self, n_components: int = 2, metric: bool = True):
        self.n_components = n_components
        self.metric = metric
        self.stress_ = None
        self.embedding_ = None
    
    def fit_transform(self, X: np.ndarray) -> np.ndarray:
        """MDSを実行"""
        if self.metric:
            return self._classical_mds(X)
        else:
            return self._non_metric_mds(X)
    
    def _classical_mds(self, X: np.ndarray) -> np.ndarray:
        """古典的MDS"""
        n = X.shape[0]
        
        # 距離行列を計算
        D = euclidean_distances(X)
        D_sq = D**2
        
        # 中心化行列
        H = np.eye(n) - np.ones((n, n)) / n
        
        # グラム行列
        G = -0.5 * H @ D_sq @ H
        
        # 固有値分解
        eigenvals, eigenvecs = np.linalg.eigh(G)
        
        # 固有値の降順でソート
        idx = eigenvals.argsort()[::-1]
        eigenvals = eigenvals[idx]
        eigenvecs = eigenvecs[:, idx]
        
        # 正の固有値のみ使用
        pos_eigenvals = eigenvals[:self.n_components]
        pos_eigenvals = np.maximum(pos_eigenvals, 0)
        
        # 座標を計算
        self.embedding_ = eigenvecs[:, :self.n_components] @ np.diag(np.sqrt(pos_eigenvals))
        
        # ストレスを計算
        self._compute_stress(X, self.embedding_)
        
        return self.embedding_
    
    def _non_metric_mds(self, X: np.ndarray) -> np.ndarray:
        """非計量MDS（Kruskalアルゴリズム）"""
        from scipy.optimize import minimize
        
        n = X.shape[0]
        
        # 距離行列を計算
        D = euclidean_distances(X)
        
        # 順位を計算
        ranks = self._compute_ranks(D)
        
        # 初期座標（古典的MDSで初期化）
        initial_embedding = self._classical_mds(X)
        
        # 最適化
        def objective(coords):
            coords = coords.reshape(n, self.n_components)
            embedded_distances = euclidean_distances(coords)
            return self._kruskal_stress(embedded_distances, ranks)
        
        result = minimize(
            objective,
            initial_embedding.flatten(),
            method='L-BFGS-B'
        )
        
        self.embedding_ = result.x.reshape(n, self.n_components)
        self.stress_ = result.fun
        
        return self.embedding_
    
    def _compute_ranks(self, D: np.ndarray) -> np.ndarray:
        """距離の順位を計算"""
        n = D.shape[0]
        ranks = np.zeros_like(D)
        
        # 上三角部分のみ処理
        triu_indices = np.triu_indices(n, k=1)
        distances = D[triu_indices]
        
        # 順位を計算
        sorted_indices = distances.argsort()
        rank_values = np.arange(1, len(distances) + 1)
        ranks_flat = np.zeros_like(distances)
        ranks_flat[sorted_indices] = rank_values
        
        # 対称行列に復元
        ranks[triu_indices] = ranks_flat
        ranks = ranks + ranks.T
        
        return ranks
    
    def _kruskal_stress(self, embedded_distances: np.ndarray, 
                       ranks: np.ndarray) -> float:
        """Kruskalのストレス関数"""
        # 単調回帰により距離を変換
        transformed_distances = self._isotonic_regression(embedded_distances, ranks)
        
        # ストレスを計算
        diff = embedded_distances - transformed_distances
        stress = np.sqrt(np.sum(diff**2) / np.sum(embedded_distances**2))
        
        return stress
    
    def _isotonic_regression(self, distances: np.ndarray, 
                           ranks: np.ndarray) -> np.ndarray:
        """単調回帰"""
        from sklearn.isotonic import IsotonicRegression
        
        # 上三角部分のみ使用
        n = distances.shape[0]
        triu_indices = np.triu_indices(n, k=1)
        
        dist_flat = distances[triu_indices]
        rank_flat = ranks[triu_indices]
        
        # 単調回帰
        ir = IsotonicRegression()
        transformed_flat = ir.fit_transform(rank_flat, dist_flat)
        
        # 対称行列に復元
        transformed = np.zeros_like(distances)
        transformed[triu_indices] = transformed_flat
        transformed = transformed + transformed.T
        
        return transformed
    
    def _compute_stress(self, X: np.ndarray, embedding: np.ndarray):
        """ストレスを計算"""
        original_distances = euclidean_distances(X)
        embedded_distances = euclidean_distances(embedding)
        
        diff = original_distances - embedded_distances
        self.stress_ = np.sqrt(np.sum(diff**2) / np.sum(original_distances**2))
```

## 可視化手法

### 1. 散布図 (Scatter Plot)

#### 2D散布図
```python
import matplotlib.pyplot as plt
import seaborn as sns

def create_scatter_2d(data: np.ndarray, labels: np.ndarray = None,
                     colors: np.ndarray = None, sizes: np.ndarray = None):
    """2D散布図を作成"""
    fig, ax = plt.subplots(figsize=(10, 8))
    
    if labels is not None:
        # ラベルごとに色分け
        unique_labels = np.unique(labels)
        colors = plt.cm.tab10(np.linspace(0, 1, len(unique_labels)))
        
        for i, label in enumerate(unique_labels):
            mask = labels == label
            ax.scatter(data[mask, 0], data[mask, 1], 
                      c=colors[i], label=label, alpha=0.7)
        ax.legend()
    else:
        scatter = ax.scatter(data[:, 0], data[:, 1], 
                           c=colors, s=sizes, alpha=0.7)
        if colors is not None:
            plt.colorbar(scatter)
    
    ax.set_xlabel('Dimension 1')
    ax.set_ylabel('Dimension 2')
    ax.set_title('2D Scatter Plot')
    
    return fig, ax
```

#### 3D散布図
```python
from mpl_toolkits.mplot3d import Axes3D

def create_scatter_3d(data: np.ndarray, labels: np.ndarray = None):
    """3D散布図を作成"""
    fig = plt.figure(figsize=(12, 9))
    ax = fig.add_subplot(111, projection='3d')
    
    if labels is not None:
        unique_labels = np.unique(labels)
        colors = plt.cm.tab10(np.linspace(0, 1, len(unique_labels)))
        
        for i, label in enumerate(unique_labels):
            mask = labels == label
            ax.scatter(data[mask, 0], data[mask, 1], data[mask, 2],
                      c=colors[i], label=label, alpha=0.7)
        ax.legend()
    else:
        ax.scatter(data[:, 0], data[:, 1], data[:, 2], alpha=0.7)
    
    ax.set_xlabel('Dimension 1')
    ax.set_ylabel('Dimension 2')
    ax.set_zlabel('Dimension 3')
    ax.set_title('3D Scatter Plot')
    
    return fig, ax
```

### 2. 並行座標プロット (Parallel Coordinates Plot)

```python
import pandas as pd

def create_parallel_coordinates(data: np.ndarray, labels: np.ndarray = None,
                              feature_names: list = None):
    """並行座標プロットを作成"""
    # データフレームに変換
    if feature_names is None:
        feature_names = [f'Feature_{i}' for i in range(data.shape[1])]
    
    df = pd.DataFrame(data, columns=feature_names)
    
    if labels is not None:
        df['Label'] = labels
        
        # ラベルごとに描画
        fig, ax = plt.subplots(figsize=(15, 8))
        
        unique_labels = np.unique(labels)
        colors = plt.cm.tab10(np.linspace(0, 1, len(unique_labels)))
        
        for i, label in enumerate(unique_labels):
            label_data = df[df['Label'] == label].drop('Label', axis=1)
            
            # 正規化
            normalized_data = (label_data - label_data.min()) / (label_data.max() - label_data.min())
            
            for idx, row in normalized_data.iterrows():
                ax.plot(range(len(feature_names)), row, 
                       color=colors[i], alpha=0.3)
        
        # 凡例用の線
        for i, label in enumerate(unique_labels):
            ax.plot([], [], color=colors[i], label=label, linewidth=2)
        
        ax.legend()
    else:
        # 全データを単色で描画
        fig, ax = plt.subplots(figsize=(15, 8))
        
        # 正規化
        normalized_data = (data - data.min(axis=0)) / (data.max(axis=0) - data.min(axis=0))
        
        for row in normalized_data:
            ax.plot(range(len(feature_names)), row, color='blue', alpha=0.3)
    
    ax.set_xticks(range(len(feature_names)))
    ax.set_xticklabels(feature_names, rotation=45)
    ax.set_ylabel('Normalized Value')
    ax.set_title('Parallel Coordinates Plot')
    ax.grid(True, alpha=0.3)
    
    return fig, ax
```

### 3. ヒートマップ (Heatmap)

```python
def create_correlation_heatmap(data: np.ndarray, feature_names: list = None):
    """相関行列のヒートマップを作成"""
    # 相関行列を計算
    corr_matrix = np.corrcoef(data.T)
    
    if feature_names is None:
        feature_names = [f'Feature_{i}' for i in range(data.shape[1])]
    
    # ヒートマップを作成
    fig, ax = plt.subplots(figsize=(12, 10))
    
    im = ax.imshow(corr_matrix, cmap='RdBu_r', aspect='auto', vmin=-1, vmax=1)
    
    # カラーバー
    cbar = plt.colorbar(im, ax=ax)
    cbar.set_label('Correlation Coefficient')
    
    # 軸ラベル
    ax.set_xticks(range(len(feature_names)))
    ax.set_yticks(range(len(feature_names)))
    ax.set_xticklabels(feature_names, rotation=45, ha='right')
    ax.set_yticklabels(feature_names)
    
    # 値を表示
    for i in range(len(feature_names)):
        for j in range(len(feature_names)):
            text = ax.text(j, i, f'{corr_matrix[i, j]:.2f}',
                          ha="center", va="center", color="black")
    
    ax.set_title('Feature Correlation Heatmap')
    
    return fig, ax

def create_distance_heatmap(data: np.ndarray, sample_names: list = None):
    """距離行列のヒートマップを作成"""
    from sklearn.metrics import pairwise_distances
    
    # 距離行列を計算
    distance_matrix = pairwise_distances(data)
    
    if sample_names is None:
        sample_names = [f'Sample_{i}' for i in range(data.shape[0])]
    
    # ヒートマップを作成
    fig, ax = plt.subplots(figsize=(12, 10))
    
    im = ax.imshow(distance_matrix, cmap='viridis', aspect='auto')
    
    # カラーバー
    cbar = plt.colorbar(im, ax=ax)
    cbar.set_label('Euclidean Distance')
    
    # 軸ラベル
    ax.set_xticks(range(0, len(sample_names), max(1, len(sample_names)//20)))
    ax.set_yticks(range(0, len(sample_names), max(1, len(sample_names)//20)))
    ax.set_xticklabels([sample_names[i] for i in ax.get_xticks()], rotation=45)
    ax.set_yticklabels([sample_names[i] for i in ax.get_yticks()])
    
    ax.set_title('Sample Distance Heatmap')
    
    return fig, ax
```

## 評価メトリクス

### 1. 次元削減の品質評価

```python
def evaluate_dimension_reduction(original_data: np.ndarray, 
                               reduced_data: np.ndarray,
                               k: int = 10) -> dict:
    """次元削減の品質を評価"""
    from sklearn.neighbors import NearestNeighbors
    from scipy.stats import spearmanr
    
    n = original_data.shape[0]
    
    # 1. 近傍保持率 (Neighborhood Preservation)
    nbrs_orig = NearestNeighbors(n_neighbors=k+1).fit(original_data)
    _, indices_orig = nbrs_orig.kneighbors(original_data)
    indices_orig = indices_orig[:, 1:]  # 自分自身を除く
    
    nbrs_reduced = NearestNeighbors(n_neighbors=k+1).fit(reduced_data)
    _, indices_reduced = nbrs_reduced.kneighbors(reduced_data)
    indices_reduced = indices_reduced[:, 1:]  # 自分自身を除く
    
    preservation_scores = []
    for i in range(n):
        intersection = len(set(indices_orig[i]) & set(indices_reduced[i]))
        preservation_scores.append(intersection / k)
    
    neighborhood_preservation = np.mean(preservation_scores)
    
    # 2. ランク相関
    from sklearn.metrics import pairwise_distances
    
    orig_distances = pairwise_distances(original_data).flatten()
    reduced_distances = pairwise_distances(reduced_data).flatten()
    
    # 自分自身との距離（0）を除去
    mask = orig_distances > 0
    orig_distances = orig_distances[mask]
    reduced_distances = reduced_distances[mask]
    
    rank_correlation, _ = spearmanr(orig_distances, reduced_distances)
    
    # 3. 寄与率（PCAの場合のみ有効）
    explained_variance_ratio = None
    if hasattr(reduced_data, 'explained_variance_ratio_'):
        explained_variance_ratio = np.sum(reduced_data.explained_variance_ratio_)
    
    # 4. ストレス（MDSの場合のみ有効）
    stress = None
    if hasattr(reduced_data, 'stress_'):
        stress = reduced_data.stress_
    
    return {
        'neighborhood_preservation': neighborhood_preservation,
        'rank_correlation': rank_correlation,
        'explained_variance_ratio': explained_variance_ratio,
        'stress': stress
    }
```

### 2. クラスタリング品質評価

```python
from sklearn.metrics import silhouette_score, adjusted_rand_score, normalized_mutual_info_score

def evaluate_clustering(data: np.ndarray, labels: np.ndarray, 
                       true_labels: np.ndarray = None) -> dict:
    """クラスタリングの品質を評価"""
    
    results = {}
    
    # 1. シルエット係数
    if len(set(labels)) > 1:
        results['silhouette_score'] = silhouette_score(data, labels)
    else:
        results['silhouette_score'] = 0
    
    # 2. 真のラベルがある場合の外部指標
    if true_labels is not None:
        results['adjusted_rand_index'] = adjusted_rand_score(true_labels, labels)
        results['normalized_mutual_info'] = normalized_mutual_info_score(true_labels, labels)
    
    # 3. クラスター内分散
    unique_labels = np.unique(labels)
    within_cluster_variance = 0
    
    for label in unique_labels:
        if label == -1:  # ノイズポイント（DBSCANなど）
            continue
        cluster_points = data[labels == label]
        if len(cluster_points) > 1:
            centroid = np.mean(cluster_points, axis=0)
            variance = np.mean(np.sum((cluster_points - centroid)**2, axis=1))
            within_cluster_variance += variance
    
    results['within_cluster_variance'] = within_cluster_variance / len(unique_labels)
    
    # 4. クラスター間分散
    centroids = []
    for label in unique_labels:
        if label == -1:
            continue
        cluster_points = data[labels == label]
        centroids.append(np.mean(cluster_points, axis=0))
    
    if len(centroids) > 1:
        centroids = np.array(centroids)
        overall_centroid = np.mean(centroids, axis=0)
        between_cluster_variance = np.mean(np.sum((centroids - overall_centroid)**2, axis=1))
        results['between_cluster_variance'] = between_cluster_variance
    else:
        results['between_cluster_variance'] = 0
    
    return results
```

## パフォーマンス最適化

### 1. 大規模データ対応

```python
import numpy as np
from sklearn.random_projection import GaussianRandomProjection

def large_scale_dimension_reduction(data: np.ndarray, 
                                  method: str = 'random_projection',
                                  target_dim: int = 50,
                                  final_dim: int = 2) -> np.ndarray:
    """大規模データの次元削減"""
    
    n_samples, n_features = data.shape
    
    # 1段階目: ランダム投影で中間次元まで削減
    if n_features > target_dim:
        rp = GaussianRandomProjection(n_components=target_dim, random_state=42)
        data_projected = rp.fit_transform(data)
    else:
        data_projected = data
    
    # 2段階目: 最終的な次元削減
    if method == 'pca':
        from sklearn.decomposition import IncrementalPCA
        pca = IncrementalPCA(n_components=final_dim, batch_size=1000)
        result = pca.fit_transform(data_projected)
    
    elif method == 'umap':
        # UMAPは近似的な実装を使用
        from umap import UMAP
        umap_reducer = UMAP(n_components=final_dim, n_neighbors=15, 
                           min_dist=0.1, metric='euclidean')
        result = umap_reducer.fit_transform(data_projected)
    
    elif method == 'tsne':
        # t-SNEは計算量が多いため、サンプリングを併用
        if n_samples > 5000:
            indices = np.random.choice(n_samples, 5000, replace=False)
            sampled_data = data_projected[indices]
            
            from sklearn.manifold import TSNE
            tsne = TSNE(n_components=final_dim, random_state=42)
            result_sampled = tsne.fit_transform(sampled_data)
            
            # 残りのデータポイントを近似的に埋め込み
            result = np.zeros((n_samples, final_dim))
            result[indices] = result_sampled
            
            # k近傍を使って残りのポイントを補間
            from sklearn.neighbors import KNeighborsRegressor
            for dim in range(final_dim):
                knn = KNeighborsRegressor(n_neighbors=5)
                knn.fit(sampled_data, result_sampled[:, dim])
                remaining_indices = np.setdiff1d(np.arange(n_samples), indices)
                result[remaining_indices, dim] = knn.predict(data_projected[remaining_indices])
        else:
            from sklearn.manifold import TSNE
            tsne = TSNE(n_components=final_dim, random_state=42)
            result = tsne.fit_transform(data_projected)
    
    return result
```

### 2. GPU加速実装

```python
try:
    import cupy as cp
    import cupyx.scipy.linalg as cp_linalg
    GPU_AVAILABLE = True
except ImportError:
    GPU_AVAILABLE = False

def gpu_pca(data: np.ndarray, n_components: int = 2) -> np.ndarray:
    """GPU加速PCA"""
    if not GPU_AVAILABLE:
        # CPU実装にフォールバック
        from sklearn.decomposition import PCA
        pca = PCA(n_components=n_components)
        return pca.fit_transform(data)
    
    # GPU実装
    gpu_data = cp.asarray(data)
    
    # データの中心化
    mean = cp.mean(gpu_data, axis=0)
    gpu_data_centered = gpu_data - mean
    
    # SVDを使用してPCAを計算
    U, s, Vt = cp_linalg.svd(gpu_data_centered, full_matrices=False)
    
    # 結果を計算
    result = U[:, :n_components] * s[:n_components]
    
    # CPUに戻す
    return cp.asnumpy(result)

def gpu_distance_matrix(data: np.ndarray) -> np.ndarray:
    """GPU加速距離行列計算"""
    if not GPU_AVAILABLE:
        from sklearn.metrics import pairwise_distances
        return pairwise_distances(data)
    
    gpu_data = cp.asarray(data)
    n = gpu_data.shape[0]
    
    # 効率的な距離計算
    # ||x - y||² = ||x||² + ||y||² - 2⟨x, y⟩
    norms_sq = cp.sum(gpu_data**2, axis=1)
    distances_sq = (norms_sq[:, None] + norms_sq[None, :] - 
                   2 * cp.dot(gpu_data, gpu_data.T))
    distances_sq = cp.maximum(distances_sq, 0)  # 数値誤差対策
    distances = cp.sqrt(distances_sq)
    
    return cp.asnumpy(distances)
```

## 今後の拡張予定

### 1. 新しいアルゴリズム
- **VAE (Variational Autoencoder)**: 生成モデルベースの次元削減
- **Diffusion Maps**: 拡散過程に基づく手法
- **LargeVis**: 大規模データ向けt-SNE
- **TriMap**: トリプレット制約ベースの埋め込み

### 2. 特殊データ対応
- **時系列データ**: DTW距離による可視化
- **グラフデータ**: ネットワーク構造の可視化
- **テキストデータ**: 意味的類似性の可視化
- **画像データ**: 畳み込み特徴量の可視化

### 3. インタラクティブ機能
- **リアルタイム更新**: パラメータ変更の即座反映
- **ブラッシング・リンキング**: 複数ビューの連動
- **アニメーション**: 次元削減過程の可視化
- **VR/AR対応**: 没入型データ探索

---

このドキュメントは実装の進行に応じて継続的に更新されます。

最終更新日: 2025年7月15日