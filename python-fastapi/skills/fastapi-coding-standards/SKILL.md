---
name: fastapi-coding-standards
description: Enforce FastAPI coding standards for this stock LINE bot project. Use when writing new code, refactoring, or reviewing code to ensure compliance with project conventions (type hints, async patterns, Chinese comments, dependency injection, error handling).
allowed-tools: Read, Edit, Grep, Glob
---

# FastAPI 程式碼規範 Skill

專為本專案（股票 LINE Bot）設計的 FastAPI 程式碼規範檢查與生成工具。

## 專案架構

本專案採用**三層架構**：

```
API Layer (app/api/v1/endpoints/)
    ↓ 呼叫
Service Layer (app/services/)
    ↓ 呼叫
Database Layer (database/)
```

## 核心規範

### 1. Python 基礎規範

#### 版本與風格
- Python >= 3.11
- 遵循 PEP 8
- 使用型別提示（Type Hints）
- **所有註解和 docstring 使用繁體中文**

#### 命名規範
```python
# ✅ 正確
class StockService:           # 類別：PascalCase
def get_stock_list():         # 函數：snake_case
stock_code: str               # 變數：snake_case
API_VERSION = "v1"            # 常數：UPPER_CASE

# ❌ 錯誤
class stock_service:          # 類別不應使用 snake_case
def getStockList():           # 函數不應使用 camelCase
```

#### Import 順序
```python
# 1. 標準庫
from datetime import datetime
from typing import List, Optional

# 2. 第三方套件
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session

# 3. 本地模組
from database.session import get_db
from app.schemas.stock import StockResponse
```

---

### 2. API Layer 規範 (`app/api/v1/endpoints/`)

#### 職責
- ✅ 處理 HTTP 請求/回應
- ✅ 參數驗證（Query, Path, Body）
- ✅ 呼叫 Service Layer
- ❌ 不應包含業務邏輯
- ❌ 不應直接操作資料庫

#### 標準模板
```python
from fastapi import APIRouter, Depends, HTTPException, Query
from sqlalchemy.orm import Session
from typing import List, Optional

from database.session import get_db
from app.schemas.stock import StockResponse, StockCreate
from app.services.stock_service import StockService

router = APIRouter()

@router.get("/stocks", response_model=List[StockResponse])
async def get_stocks(
    skip: int = Query(0, ge=0, description="跳過筆數"),
    limit: int = Query(100, ge=1, le=1000, description="限制筆數"),
    stock_code: Optional[str] = Query(None, description="股票代碼"),
    stock_service: StockService = Depends(get_stock_service),
    db: Session = Depends(get_db)
):
    """
    取得股票列表

    - **skip**: 跳過筆數
    - **limit**: 限制筆數
    - **stock_code**: 股票代碼（可選）
    """
    stocks = await stock_service.get_stock_list(
        db=db,
        skip=skip,
        limit=limit,
        stock_code=stock_code
    )
    return stocks
```

#### 必須檢查項目
- [ ] 使用 `async def` 定義端點
- [ ] 所有參數都有型別提示
- [ ] 使用 `Query`/`Path`/`Body` 並提供繁體中文 `description`
- [ ] 使用 `Depends()` 注入依賴（資料庫、服務）
- [ ] 有繁體中文 docstring
- [ ] 指定 `response_model`
- [ ] 錯誤處理使用 `HTTPException`

#### Anti-patterns（禁止）
```python
# ❌ 錯誤 1：端點包含業務邏輯
@router.get("/stocks")
async def get_stocks(db: Session = Depends(get_db)):
    # ❌ 不應該在端點直接查詢資料庫
    stocks = db.query(Stock).filter(...).all()
    # ❌ 不應該在端點處理業務邏輯
    if stock.is_disposal:
        stock.status = "處置中"
    return stocks

# ❌ 錯誤 2：直接實例化服務
@router.get("/stocks")
async def get_stocks():
    service = StockService()  # ❌ 應使用 Depends()
    return service.get_stocks()

# ❌ 錯誤 3：缺少型別提示
@router.get("/stocks")
async def get_stocks(skip, limit, db):  # ❌ 缺少型別
    pass

# ❌ 錯誤 4：沒有使用 async
@router.get("/stocks")  # ❌ 應使用 async def
def get_stocks(db: Session = Depends(get_db)):
    pass
```

---

### 3. Service Layer 規範 (`app/services/`)

#### 職責
- ✅ 封裝業務邏輯
- ✅ 呼叫外部 API（爬蟲、LINE Bot）
- ✅ 資料轉換與驗證
- ✅ 整合多個資料來源
- ❌ 應透過 Repository 操作資料庫（不直接使用 SessionLocal）

#### 標準模板
```python
from typing import Optional, List, Dict
from datetime import date
from sqlalchemy.orm import Session

from app.repositories.stock_repository import StockRepository
from database.models import Stock
from app.exceptions import DataNotFoundException

class StockService:
    """股票服務層"""

    def __init__(self, repository: StockRepository):
        self.repository = repository

    async def get_stock_by_code(
        self,
        db: Session,
        stock_code: str
    ) -> Optional[Dict]:
        """
        根據股票代碼取得股票資訊

        Args:
            db: 資料庫 Session
            stock_code: 股票代碼

        Returns:
            股票資訊字典，如果不存在則拋出異常

        Raises:
            DataNotFoundException: 找不到股票時
        """
        stock = await self.repository.get_by_code(db, stock_code)
        if not stock:
            raise DataNotFoundException(f"找不到股票代碼：{stock_code}")

        return self._convert_to_dict(stock)

    def _convert_to_dict(self, stock: Stock) -> Dict:
        """將 ORM 模型轉換為字典"""
        return {
            "stock_code": stock.stock_code,
            "stock_name": stock.stock_name,
            "is_disposal": stock.is_disposal,
            "is_attention": stock.is_attention
        }

# 單例模式
stock_service = StockService(repository=StockRepository())
```

#### 必須檢查項目
- [ ] 類別有繁體中文 docstring
- [ ] 所有方法都有型別提示
- [ ] 使用 Repository 操作資料庫（不直接用 SessionLocal）
- [ ] 錯誤處理清晰（拋出自定義異常）
- [ ] 複雜邏輯有註解說明
- [ ] 提供單例實例

#### Anti-patterns（禁止）
```python
# ❌ 錯誤 1：直接使用 SessionLocal
from database.session import SessionLocal

class StockService:
    def get_stocks(self):
        db = SessionLocal()  # ❌ 應透過依賴注入
        try:
            stocks = db.query(Stock).all()
        finally:
            db.close()
        return stocks

# ❌ 錯誤 2：包含 HTTP 處理邏輯
class StockService:
    def get_stocks(self, request: Request):  # ❌ Service 不應處理 HTTP
        headers = request.headers
        return response.json()

# ❌ 錯誤 3：缺少錯誤處理
class StockService:
    def get_stock(self, db: Session, stock_id: int):
        stock = db.query(Stock).filter(Stock.id == stock_id).first()
        return stock.stock_code  # ❌ 如果 stock 是 None 會拋出 AttributeError
```

---

### 4. Repository Layer 規範 (`app/repositories/`)

#### 職責
- ✅ 封裝所有資料庫操作
- ✅ 提供查詢、新增、更新、刪除方法
- ✅ 使用 SQLAlchemy ORM
- ❌ 不應包含業務邏輯

#### 標準模板
```python
from typing import Optional, List
from sqlalchemy.orm import Session
from sqlalchemy import select

from database.models import Stock

class StockRepository:
    """股票資料存取層"""

    def get_by_id(self, db: Session, stock_id: int) -> Optional[Stock]:
        """根據 ID 取得股票"""
        return db.query(Stock).filter(Stock.id == stock_id).first()

    def get_by_code(self, db: Session, stock_code: str) -> Optional[Stock]:
        """根據股票代碼取得股票"""
        return db.query(Stock).filter(Stock.stock_code == stock_code).first()

    def get_all(
        self,
        db: Session,
        skip: int = 0,
        limit: int = 100
    ) -> List[Stock]:
        """取得股票列表"""
        return db.query(Stock).offset(skip).limit(limit).all()

    def create(self, db: Session, stock: Stock) -> Stock:
        """新增股票"""
        db.add(stock)
        db.commit()
        db.refresh(stock)
        return stock

    def update(self, db: Session, stock: Stock) -> Stock:
        """更新股票"""
        db.commit()
        db.refresh(stock)
        return stock

    def delete(self, db: Session, stock_id: int) -> bool:
        """刪除股票"""
        stock = self.get_by_id(db, stock_id)
        if stock:
            db.delete(stock)
            db.commit()
            return True
        return False
```

#### 必須檢查項目
- [ ] 所有方法都有型別提示
- [ ] 使用 ORM 查詢（不使用原生 SQL）
- [ ] 查詢後處理 None 的情況
- [ ] CRUD 操作包含 commit 和 refresh

---

### 5. Schema Layer 規範 (`app/schemas/`)

#### 職責
- ✅ 定義 Pydantic 資料驗證模型
- ✅ 區分 Create、Update、Response schemas

#### 標準模板
```python
from pydantic import BaseModel, Field, ConfigDict
from typing import Optional
from datetime import datetime

class StockBase(BaseModel):
    """股票基礎 Schema"""
    stock_code: str = Field(..., min_length=4, max_length=10, description="股票代碼")
    stock_name: str = Field(..., max_length=100, description="股票名稱")

class StockCreate(StockBase):
    """建立股票 Schema"""
    pass

class StockUpdate(BaseModel):
    """更新股票 Schema"""
    stock_name: Optional[str] = Field(None, max_length=100, description="股票名稱")
    is_disposal: Optional[bool] = Field(None, description="是否為處置股")
    is_attention: Optional[bool] = Field(None, description="是否為注意股")

class StockResponse(StockBase):
    """股票回應 Schema"""
    model_config = ConfigDict(from_attributes=True)

    id: int
    is_disposal: bool
    is_attention: bool
    created_at: datetime
    updated_at: datetime
```

#### 必須檢查項目
- [ ] 使用 `ConfigDict(from_attributes=True)` 而非舊版 `orm_mode`
- [ ] 所有欄位都有型別提示
- [ ] 使用 `Field()` 提供驗證和繁體中文描述
- [ ] 分離 Create、Update、Response schemas

---

### 6. Database Layer 規範 (`database/`)

#### Model 定義
```python
from sqlalchemy import Column, Integer, String, Boolean, DateTime
from sqlalchemy.sql import func
from database.base import Base

class Stock(Base):
    """股票模型"""
    __tablename__ = "stocks"

    id = Column(Integer, primary_key=True, autoincrement=True, comment="主鍵")
    stock_code = Column(String(10), unique=True, index=True, nullable=False, comment="股票代碼")
    stock_name = Column(String(100), nullable=False, comment="股票名稱")
    is_disposal = Column(Boolean, default=False, comment="是否為處置股")
    is_attention = Column(Boolean, default=False, comment="是否為注意股")
    created_at = Column(DateTime(timezone=True), server_default=func.now(), comment="建立時間")
    updated_at = Column(DateTime(timezone=True), onupdate=func.now(), comment="更新時間")
```

#### 必須檢查項目
- [ ] 所有欄位都有 `comment`
- [ ] 時間欄位使用 `func.now()`
- [ ] 適當設置 `index`、`unique`、`nullable`
- [ ] 外鍵使用 `ForeignKey` 定義

---

### 7. 錯誤處理規範

#### 自定義異常
```python
# app/exceptions.py
class BusinessException(Exception):
    """業務邏輯異常基類"""
    def __init__(self, message: str, code: int = 400):
        self.message = message
        self.code = code
        super().__init__(self.message)

class DataNotFoundException(BusinessException):
    """資料未找到異常"""
    def __init__(self, message: str = "資料未找到"):
        super().__init__(message, code=404)

class ExternalAPIException(BusinessException):
    """外部 API 呼叫異常"""
    def __init__(self, message: str = "外部服務異常"):
        super().__init__(message, code=503)

class ValidationException(BusinessException):
    """資料驗證異常"""
    def __init__(self, message: str = "資料驗證失敗"):
        super().__init__(message, code=422)
```

#### 使用方式
```python
# Service Layer
from app.exceptions import DataNotFoundException

class StockService:
    async def get_stock(self, db: Session, stock_id: int):
        stock = await self.repository.get_by_id(db, stock_id)
        if not stock:
            raise DataNotFoundException(f"找不到股票 ID：{stock_id}")
        return stock

# API Layer
from fastapi import HTTPException
from app.exceptions import BusinessException

@app.exception_handler(BusinessException)
async def business_exception_handler(request: Request, exc: BusinessException):
    return JSONResponse(
        status_code=exc.code,
        content={"message": exc.message, "success": False}
    )
```

---

### 8. 依賴注入規範

#### 資料庫 Session
```python
# database/session.py
from sqlalchemy.orm import Session

def get_db() -> Generator[Session, None, None]:
    """取得資料庫 Session"""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

#### 服務注入
```python
# app/dependencies.py
from app.services.stock_service import stock_service
from app.repositories.stock_repository import StockRepository

def get_stock_service() -> StockService:
    """取得股票服務"""
    return stock_service

def get_stock_repository() -> StockRepository:
    """取得股票 Repository"""
    return StockRepository()
```

#### 使用方式
```python
@router.get("/stocks")
async def get_stocks(
    db: Session = Depends(get_db),
    stock_service: StockService = Depends(get_stock_service)
):
    return await stock_service.get_stocks(db)
```

---

### 9. 安全規範

#### SQL Injection 防護
```python
# ✅ 正確：使用 ORM
stock = db.query(Stock).filter(Stock.stock_code == code).first()

# ❌ 錯誤：原生 SQL 拼接
stock = db.execute(f"SELECT * FROM stocks WHERE stock_code = '{code}'")
```

#### 敏感資料處理
```python
# ✅ 正確：使用環境變數
from core.config import settings

line_channel_secret = settings.LINE_CHANNEL_SECRET

# ❌ 錯誤：硬編碼
line_channel_secret = "your_secret_here"
```

#### 輸入驗證
```python
# ✅ 正確：使用 Pydantic 驗證
class StockCreate(BaseModel):
    stock_code: str = Field(..., min_length=4, max_length=10, pattern=r'^\d{4,6}$')

# ❌ 錯誤：不驗證直接使用
@router.post("/stocks")
async def create_stock(stock_code: str):  # 沒有驗證
    pass
```

---

### 10. 測試規範

#### 測試檔案命名
```
test/
├── test_api/
│   └── test_stocks.py
├── test_services/
│   └── test_stock_service.py
└── test_repositories/
    └── test_stock_repository.py
```

#### 測試範例
```python
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

@pytest.fixture
def test_db():
    """測試資料庫"""
    engine = create_engine("sqlite:///:memory:")
    TestingSessionLocal = sessionmaker(bind=engine)
    Base.metadata.create_all(bind=engine)

    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()

def test_get_stock(test_db):
    """測試取得股票"""
    # Arrange
    stock = Stock(stock_code="2330", stock_name="台積電")
    test_db.add(stock)
    test_db.commit()

    # Act
    result = stock_repository.get_by_code(test_db, "2330")

    # Assert
    assert result is not None
    assert result.stock_name == "台積電"
```

---

## 程式碼檢查清單

### 新增功能前
- [ ] 閱讀相關現有程式碼，理解現有架構
- [ ] 確認需要修改的層（API/Service/Repository）
- [ ] 設計 Schema（如果需要）
- [ ] 考慮錯誤處理

### 撰寫程式碼時
- [ ] 所有函數都有型別提示
- [ ] 所有函數都有繁體中文 docstring
- [ ] 使用 `async def` 定義異步函數
- [ ] 使用依賴注入（不直接實例化）
- [ ] 適當的錯誤處理（try-except 或拋出自定義異常）
- [ ] 遵循單一職責原則
- [ ] 不硬編碼敏感資訊

### 程式碼完成後
- [ ] 檢查是否有安全漏洞（SQL Injection、XSS 等）
- [ ] 確認符合專案架構（三層分離）
- [ ] 添加必要的註解
- [ ] 撰寫單元測試（如果是核心功能）
- [ ] 執行 linter 檢查

---

## 常見錯誤修正

### 錯誤 1：端點包含業務邏輯
```python
# ❌ 錯誤
@router.get("/stocks")
async def get_stocks(db: Session = Depends(get_db)):
    stocks = db.query(Stock).all()
    for stock in stocks:
        if stock.is_disposal:
            stock.status = "處置中"
    return stocks

# ✅ 正確
@router.get("/stocks", response_model=List[StockResponse])
async def get_stocks(
    stock_service: StockService = Depends(get_stock_service),
    db: Session = Depends(get_db)
):
    """取得股票列表"""
    return await stock_service.get_stock_list(db)
```

### 錯誤 2：服務層直接使用 SessionLocal
```python
# ❌ 錯誤
class StockService:
    def get_stocks(self):
        db = SessionLocal()
        try:
            return db.query(Stock).all()
        finally:
            db.close()

# ✅ 正確
class StockService:
    def __init__(self, repository: StockRepository):
        self.repository = repository

    async def get_stocks(self, db: Session) -> List[Stock]:
        """取得股票列表"""
        return await self.repository.get_all(db)
```

### 錯誤 3：缺少型別提示
```python
# ❌ 錯誤
def get_stock(db, stock_id):
    return db.query(Stock).filter(Stock.id == stock_id).first()

# ✅ 正確
def get_stock(db: Session, stock_id: int) -> Optional[Stock]:
    """根據 ID 取得股票"""
    return db.query(Stock).filter(Stock.id == stock_id).first()
```

---

## 參考資料

- [FastAPI 官方文檔](https://fastapi.tiangolo.com/)
- [Pydantic 文檔](https://docs.pydantic.dev/)
- [SQLAlchemy 文檔](https://docs.sqlalchemy.org/)
- [PEP 8 風格指南](https://peps.python.org/pep-0008/)

---

## 使用此 Skill

當你需要：
- 撰寫新的 API 端點
- 新增 Service 或 Repository
- 重構現有程式碼
- 檢查程式碼是否符合規範

Claude 會自動應用此 Skill 的規範，確保生成的程式碼符合專案標準。
