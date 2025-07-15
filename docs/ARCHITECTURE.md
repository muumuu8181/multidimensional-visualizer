# アーキテクチャ設計

## 概要

Multidimensional Visualizer のアーキテクチャ設計について詳細に説明します。このドキュメントは、開発者がシステムの構造と設計思想を理解するためのものです。

## 設計原則

### 1. モジュラー設計
- 各機能を独立したモジュールとして実装
- 疎結合・高凝集の原則に従う
- プラグイン機能による拡張性確保

### 2. スケーラビリティ
- 大規模データセットへの対応
- 水平・垂直スケーリングの考慮
- 非同期処理による応答性向上

### 3. パフォーマンス最適化
- GPU加速の活用
- メモリ効率的なデータ構造
- ローディング時間の最小化

### 4. ユーザビリティ
- 直感的なユーザーインターフェース
- リアルタイムフィードバック
- アクセシビリティ対応

## システム構成

### 全体アーキテクチャ

```
┌─────────────────────────────────────────────────────────┐
│                     Frontend                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │
│  │    UI       │  │   Chart     │  │   Interaction   │  │
│  │ Components  │  │  Engine     │  │    Manager      │  │
│  └─────────────┘  └─────────────┘  └─────────────────┘  │
│           │              │                    │         │
└───────────┼──────────────┼────────────────────┼─────────┘
            │              │                    │
            └──────────────┼────────────────────┘
                           │
┌─────────────────────────────────────────────────────────┐
│                      API Layer                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │
│  │   REST      │  │  WebSocket  │  │    GraphQL      │  │
│  │    API      │  │     API     │  │      API        │  │
│  └─────────────┘  └─────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────┘
            │              │                    │
┌─────────────────────────────────────────────────────────┐
│                  Business Logic                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │
│  │ Dimension   │  │ Visualization│  │    Analysis     │  │
│  │ Reduction   │  │   Engine     │  │    Engine       │  │
│  │  Service    │  │              │  │                 │  │
│  └─────────────┘  └─────────────┘  └─────────────────┘  │
│           │              │                    │         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │
│  │    Data     │  │   Export    │  │    Plugin       │  │
│  │ Processing  │  │   Service   │  │    Manager      │  │
│  │  Service    │  │             │  │                 │  │
│  └─────────────┘  └─────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────┘
            │              │                    │
┌─────────────────────────────────────────────────────────┐
│                    Data Layer                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │
│  │   File      │  │  Database   │  │    Cache        │  │
│  │  Storage    │  │   Access    │  │   Manager       │  │
│  └─────────────┘  └─────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## コンポーネント詳細

### Frontend Layer

#### UI Components
```typescript
// React コンポーネント例
interface VisualizationPanelProps {
  data: DataSet;
  config: VisualizationConfig;
  onConfigChange: (config: VisualizationConfig) => void;
}

const VisualizationPanel: React.FC<VisualizationPanelProps> = ({
  data,
  config,
  onConfigChange
}) => {
  // 実装
};
```

**責務:**
- ユーザーインターフェースの描画
- ユーザー操作の受付
- 状態管理（Redux/Zustand）

**主要コンポーネント:**
- `DataUploader`: データファイルのアップロード
- `DimensionSelector`: 次元選択UI
- `VisualizationPanel`: 可視化表示領域
- `ConfigPanel`: 設定パネル
- `ExportDialog`: エクスポート機能

#### Chart Engine
```javascript
class ChartEngine {
  constructor(container, options) {
    this.container = container;
    this.options = options;
    this.renderer = new WebGLRenderer(); // or CanvasRenderer
  }
  
  render(data, config) {
    // レンダリング実装
  }
  
  update(data) {
    // 更新処理
  }
  
  destroy() {
    // クリーンアップ
  }
}
```

**技術選択肢:**
- **D3.js**: カスタム可視化
- **Three.js**: 3D可視化
- **Plot.ly**: インタラクティブグラフ
- **Observable Plot**: 統計的可視化

#### Interaction Manager
```typescript
interface InteractionManager {
  on(event: string, callback: Function): void;
  off(event: string, callback: Function): void;
  trigger(event: string, data: any): void;
}

class ZoomInteraction implements InteractionManager {
  // ズーム操作の実装
}

class SelectionInteraction implements InteractionManager {
  // 選択操作の実装
}
```

### API Layer

#### REST API
```python
# FastAPI 例
from fastapi import FastAPI, UploadFile
from typing import List, Optional

app = FastAPI()

@app.post("/api/data/upload")
async def upload_data(file: UploadFile):
    """データファイルをアップロード"""
    pass

@app.post("/api/visualization/pca")
async def apply_pca(
    data_id: str,
    n_components: int = 2,
    random_state: Optional[int] = None
):
    """PCAを適用"""
    pass

@app.get("/api/visualization/{viz_id}")
async def get_visualization(viz_id: str):
    """可視化結果を取得"""
    pass
```

#### WebSocket API
```python
from fastapi import WebSocket
import asyncio

@app.websocket("/ws/visualization")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    
    try:
        while True:
            # リアルタイム更新の処理
            data = await websocket.receive_json()
            result = await process_visualization(data)
            await websocket.send_json(result)
    except Exception as e:
        await websocket.close(code=1000)
```

### Business Logic Layer

#### Dimension Reduction Service
```python
from abc import ABC, abstractmethod
import numpy as np
from sklearn.decomposition import PCA
from sklearn.manifold import TSNE
import umap

class DimensionReducer(ABC):
    @abstractmethod
    def fit_transform(self, data: np.ndarray) -> np.ndarray:
        pass
    
    @abstractmethod
    def get_explained_variance(self) -> np.ndarray:
        pass

class PCAReducer(DimensionReducer):
    def __init__(self, n_components: int = 2):
        self.pca = PCA(n_components=n_components)
    
    def fit_transform(self, data: np.ndarray) -> np.ndarray:
        return self.pca.fit_transform(data)
    
    def get_explained_variance(self) -> np.ndarray:
        return self.pca.explained_variance_ratio_

class TSNEReducer(DimensionReducer):
    def __init__(self, n_components: int = 2, perplexity: float = 30.0):
        self.tsne = TSNE(n_components=n_components, perplexity=perplexity)
    
    def fit_transform(self, data: np.ndarray) -> np.ndarray:
        return self.tsne.fit_transform(data)
    
    def get_explained_variance(self) -> np.ndarray:
        return np.array([])  # t-SNEは説明分散を持たない

class DimensionReductionService:
    def __init__(self):
        self.reducers = {
            'pca': PCAReducer,
            'tsne': TSNEReducer,
            'umap': lambda **kwargs: UMAPReducer(**kwargs)
        }
    
    def reduce(self, data: np.ndarray, method: str, **kwargs) -> np.ndarray:
        if method not in self.reducers:
            raise ValueError(f"Unknown method: {method}")
        
        reducer = self.reducers[method](**kwargs)
        return reducer.fit_transform(data)
```

#### Visualization Engine
```python
from dataclasses import dataclass
from typing import Dict, Any, Optional
import plotly.graph_objects as go

@dataclass
class VisualizationConfig:
    chart_type: str
    x_column: str
    y_column: str
    z_column: Optional[str] = None
    color_column: Optional[str] = None
    size_column: Optional[str] = None
    opacity: float = 0.7
    marker_size: int = 5

class VisualizationEngine:
    def __init__(self):
        self.chart_types = {
            'scatter_2d': self._create_scatter_2d,
            'scatter_3d': self._create_scatter_3d,
            'parallel_coordinates': self._create_parallel_coordinates,
            'heatmap': self._create_heatmap
        }
    
    def create_visualization(
        self, 
        data: pd.DataFrame, 
        config: VisualizationConfig
    ) -> Dict[str, Any]:
        """可視化を作成"""
        if config.chart_type not in self.chart_types:
            raise ValueError(f"Unknown chart type: {config.chart_type}")
        
        chart_func = self.chart_types[config.chart_type]
        return chart_func(data, config)
    
    def _create_scatter_2d(
        self, 
        data: pd.DataFrame, 
        config: VisualizationConfig
    ) -> Dict[str, Any]:
        # 2D散布図の実装
        pass
    
    def _create_scatter_3d(
        self, 
        data: pd.DataFrame, 
        config: VisualizationConfig
    ) -> Dict[str, Any]:
        # 3D散布図の実装
        pass
```

#### Analysis Engine
```python
from sklearn.cluster import KMeans, DBSCAN
from sklearn.metrics import silhouette_score
import pandas as pd

class AnalysisEngine:
    def __init__(self):
        self.clustering_algorithms = {
            'kmeans': KMeans,
            'dbscan': DBSCAN
        }
    
    def perform_clustering(
        self, 
        data: np.ndarray, 
        algorithm: str, 
        **kwargs
    ) -> Dict[str, Any]:
        """クラスタリング分析を実行"""
        if algorithm not in self.clustering_algorithms:
            raise ValueError(f"Unknown algorithm: {algorithm}")
        
        clusterer = self.clustering_algorithms[algorithm](**kwargs)
        labels = clusterer.fit_predict(data)
        
        # 評価メトリクスを計算
        silhouette = silhouette_score(data, labels) if len(set(labels)) > 1 else 0
        
        return {
            'labels': labels.tolist(),
            'n_clusters': len(set(labels)),
            'silhouette_score': silhouette,
            'algorithm': algorithm,
            'parameters': kwargs
        }
    
    def detect_outliers(self, data: np.ndarray) -> Dict[str, Any]:
        """外れ値検出"""
        from sklearn.ensemble import IsolationForest
        
        detector = IsolationForest(contamination=0.1)
        outliers = detector.fit_predict(data)
        
        return {
            'outliers': (outliers == -1).tolist(),
            'outlier_count': sum(outliers == -1),
            'outlier_ratio': sum(outliers == -1) / len(outliers)
        }
```

### Data Layer

#### File Storage
```python
import os
import pandas as pd
from pathlib import Path
from typing import Union, Dict, Any

class FileStorageManager:
    def __init__(self, base_path: str = "./data"):
        self.base_path = Path(base_path)
        self.base_path.mkdir(exist_ok=True)
    
    def save_dataset(self, data: pd.DataFrame, dataset_id: str) -> str:
        """データセットを保存"""
        file_path = self.base_path / f"{dataset_id}.parquet"
        data.to_parquet(file_path)
        return str(file_path)
    
    def load_dataset(self, dataset_id: str) -> pd.DataFrame:
        """データセットを読み込み"""
        file_path = self.base_path / f"{dataset_id}.parquet"
        return pd.read_parquet(file_path)
    
    def save_visualization_config(
        self, 
        config: Dict[str, Any], 
        viz_id: str
    ) -> str:
        """可視化設定を保存"""
        import json
        file_path = self.base_path / f"{viz_id}_config.json"
        with open(file_path, 'w') as f:
            json.dump(config, f, indent=2)
        return str(file_path)
```

#### Database Access
```python
from sqlalchemy import create_engine, Column, String, DateTime, Text, Integer
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from datetime import datetime

Base = declarative_base()

class Dataset(Base):
    __tablename__ = 'datasets'
    
    id = Column(String, primary_key=True)
    name = Column(String, nullable=False)
    description = Column(Text)
    file_path = Column(String, nullable=False)
    upload_time = Column(DateTime, default=datetime.utcnow)
    rows = Column(Integer)
    columns = Column(Integer)

class Visualization(Base):
    __tablename__ = 'visualizations'
    
    id = Column(String, primary_key=True)
    dataset_id = Column(String, nullable=False)
    config = Column(Text)  # JSON string
    created_time = Column(DateTime, default=datetime.utcnow)

class DatabaseManager:
    def __init__(self, database_url: str):
        self.engine = create_engine(database_url)
        Base.metadata.create_all(self.engine)
        self.SessionLocal = sessionmaker(bind=self.engine)
    
    def save_dataset_metadata(self, dataset: Dataset) -> None:
        with self.SessionLocal() as session:
            session.add(dataset)
            session.commit()
    
    def get_dataset_metadata(self, dataset_id: str) -> Dataset:
        with self.SessionLocal() as session:
            return session.query(Dataset).filter(Dataset.id == dataset_id).first()
```

#### Cache Manager
```python
import redis
import json
from typing import Optional, Any

class CacheManager:
    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis_client = redis.from_url(redis_url)
        self.default_ttl = 3600  # 1時間
    
    def set(self, key: str, value: Any, ttl: Optional[int] = None) -> None:
        """キャッシュに値を設定"""
        ttl = ttl or self.default_ttl
        serialized_value = json.dumps(value, default=str)
        self.redis_client.setex(key, ttl, serialized_value)
    
    def get(self, key: str) -> Optional[Any]:
        """キャッシュから値を取得"""
        value = self.redis_client.get(key)
        if value:
            return json.loads(value)
        return None
    
    def delete(self, key: str) -> None:
        """キャッシュから値を削除"""
        self.redis_client.delete(key)
    
    def clear_pattern(self, pattern: str) -> None:
        """パターンに一致するキーを削除"""
        keys = self.redis_client.keys(pattern)
        if keys:
            self.redis_client.delete(*keys)
```

## データフロー

### 1. データアップロード
```
User → Frontend → API → Data Processing → File Storage → Database
                                     ↓
                               Cache Manager
```

### 2. 可視化作成
```
User → Frontend → API → Dimension Reduction → Visualization Engine
         ↑                     ↓                      ↓
      Result ← Cache ← Business Logic ← Analysis Engine
```

### 3. リアルタイム更新
```
User Interaction → WebSocket → Business Logic → Cache → WebSocket → Frontend
```

## セキュリティ考慮事項

### 1. 認証・認可
```python
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer

security = HTTPBearer()

async def verify_token(token: str = Depends(security)):
    # JWTトークンの検証
    if not is_valid_token(token.credentials):
        raise HTTPException(status_code=401, detail="Invalid token")
    return decode_token(token.credentials)
```

### 2. データ検証
```python
from pydantic import BaseModel, validator

class DataUploadRequest(BaseModel):
    filename: str
    file_size: int
    
    @validator('file_size')
    def validate_file_size(cls, v):
        if v > 100 * 1024 * 1024:  # 100MB制限
            raise ValueError('File size too large')
        return v
```

### 3. レート制限
```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.post("/api/data/upload")
@limiter.limit("5/minute")
async def upload_data(request: Request, file: UploadFile):
    # 実装
```

## パフォーマンス最適化

### 1. GPU加速
```python
import cupy as cp  # NVIDIA GPU
# または
import numpy as np

def gpu_accelerated_pca(data: np.ndarray) -> np.ndarray:
    if cp.cuda.is_available():
        # GPU実装
        gpu_data = cp.asarray(data)
        # cuPy/CuMLを使用した処理
        result = cp.asnumpy(gpu_result)
    else:
        # CPU実装
        result = cpu_pca(data)
    return result
```

### 2. 並列処理
```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

async def parallel_dimension_reduction(
    data: np.ndarray, 
    methods: List[str]
) -> Dict[str, np.ndarray]:
    """複数の次元削減手法を並列実行"""
    with ProcessPoolExecutor() as executor:
        tasks = [
            asyncio.get_event_loop().run_in_executor(
                executor, 
                reduce_dimensions, 
                data, 
                method
            )
            for method in methods
        ]
        results = await asyncio.gather(*tasks)
    
    return dict(zip(methods, results))
```

### 3. メモリ効率
```python
import dask.dataframe as dd

class MemoryEfficientProcessor:
    def __init__(self, chunk_size: int = 10000):
        self.chunk_size = chunk_size
    
    def process_large_dataset(self, file_path: str) -> None:
        """大きなデータセットを効率的に処理"""
        df = dd.read_csv(file_path)
        
        # チャンクごとに処理
        for chunk in df.to_delayed():
            processed_chunk = self._process_chunk(chunk.compute())
            self._save_chunk(processed_chunk)
```

## 監視・ログ

### 1. ログ設定
```python
import logging
from pythonjsonlogger import jsonlogger

# JSON形式でのログ出力
logHandler = logging.StreamHandler()
formatter = jsonlogger.JsonFormatter()
logHandler.setFormatter(formatter)
logger = logging.getLogger()
logger.addHandler(logHandler)

@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = time.time()
    response = await call_next(request)
    process_time = time.time() - start_time
    
    logger.info("Request processed", extra={
        "method": request.method,
        "url": str(request.url),
        "status_code": response.status_code,
        "process_time": process_time
    })
    
    return response
```

### 2. メトリクス収集
```python
from prometheus_client import Counter, Histogram, generate_latest

REQUEST_COUNT = Counter('requests_total', 'Total requests', ['method', 'endpoint'])
REQUEST_LATENCY = Histogram('request_duration_seconds', 'Request latency')

@app.get("/metrics")
async def metrics():
    return Response(generate_latest(), media_type="text/plain")
```

## テスト戦略

### 1. ユニットテスト
```python
import pytest
import numpy as np
from unittest.mock import Mock

class TestDimensionReduction:
    def test_pca_basic(self):
        # テストデータ
        data = np.random.rand(100, 5)
        reducer = PCAReducer(n_components=2)
        
        # 実行
        result = reducer.fit_transform(data)
        
        # 検証
        assert result.shape == (100, 2)
        assert reducer.get_explained_variance().sum() <= 1.0
```

### 2. 統合テスト
```python
from fastapi.testclient import TestClient

def test_upload_and_visualize():
    client = TestClient(app)
    
    # データアップロード
    with open("test_data.csv", "rb") as f:
        response = client.post("/api/data/upload", files={"file": f})
    assert response.status_code == 200
    data_id = response.json()["data_id"]
    
    # 可視化作成
    response = client.post(f"/api/visualization/pca", json={
        "data_id": data_id,
        "n_components": 2
    })
    assert response.status_code == 200
```

### 3. パフォーマンステスト
```python
import time
import psutil

def test_memory_usage():
    process = psutil.Process()
    initial_memory = process.memory_info().rss
    
    # 大きなデータセットでの処理
    large_data = np.random.rand(10000, 100)
    result = apply_pca(large_data)
    
    final_memory = process.memory_info().rss
    memory_increase = final_memory - initial_memory
    
    # メモリ使用量が一定以下であることを確認
    assert memory_increase < 500 * 1024 * 1024  # 500MB
```

## 今後の拡張計画

### 1. 機械学習統合
- モデル訓練結果の可視化
- 特徴量重要度の表示
- 予測結果の比較

### 2. リアルタイム処理
- ストリーミングデータ対応
- リアルタイム次元削減
- 動的可視化更新

### 3. クラウド対応
- AWS/GCP/Azure での展開
- スケーラブルな計算リソース
- 分散処理対応

---

このアーキテクチャは実装の進行に応じて継続的に更新されます。

最終更新日: 2025年7月15日