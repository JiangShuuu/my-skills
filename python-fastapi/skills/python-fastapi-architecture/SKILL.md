---
name: python-fastapi-architecture
description: Enforce FastAPI best architecture patterns for enterprise-level Python web applications. Use when writing APIs, services, CRUD operations, or reviewing code to ensure compliance with pseudo 3-tier architecture patterns.
allowed-tools: Read, Edit, Grep, Glob
---

# Python FastAPI Best Architecture Skill

A comprehensive guide for building enterprise-level FastAPI applications following the pseudo 3-tier architecture pattern from [fastapi_best_architecture](https://github.com/fastapi-practices/fastapi_best_architecture).

## Architecture Overview

This architecture adapts traditional layered patterns to FastAPI:

| Layer | Traditional | FastAPI Implementation |
|-------|-------------|------------------------|
| Presentation | Controller | API endpoints (`api/`) |
| Data Transfer | DTO | Pydantic schemas (`schema/`) |
| Business Logic | Service | Service layer (`service/`) |
| Data Access | DAO/Mapper | CRUD operations (`crud/`) |
| Entities | Model/Entity | SQLAlchemy models (`model/`) |

---

## Project Structure

```
backend/
├── app/
│   ├── admin/                    # Feature module
│   │   ├── api/
│   │   │   └── v1/              # Versioned API endpoints
│   │   │       └── sys/
│   │   │           └── user.py
│   │   ├── crud/                # Data access layer
│   │   │   └── crud_user.py
│   │   ├── model/               # SQLAlchemy models
│   │   │   └── user.py
│   │   ├── schema/              # Pydantic schemas
│   │   │   └── user.py
│   │   └── service/             # Business logic
│   │       └── user_service.py
│   └── router.py                # Route aggregation
├── common/
│   ├── exception/               # Custom exceptions
│   ├── response/                # Response schemas
│   ├── security/                # Security utilities
│   ├── enums.py                 # Enumerations
│   ├── log.py                   # Logging config
│   ├── pagination.py            # Pagination utilities
│   └── schema.py                # Base schemas
└── database/
    └── db.py                    # Database configuration
```

---

## 1. API Layer (`api/`)

### Responsibilities
- ✅ Handle HTTP requests and responses
- ✅ Parameter validation via Pydantic and Path/Query
- ✅ Dependency injection for auth, pagination, services
- ✅ Return standardized response schemas
- ❌ No business logic
- ❌ No direct database access

### Standard Pattern

```python
from typing import Annotated

from fastapi import APIRouter, Depends, Path, Query
from sqlalchemy.ext.asyncio import AsyncSession

from backend.common.response.response_schema import ResponseModel, ResponseSchemaModel
from backend.common.security.jwt import DependsJwtAuth
from backend.common.security.permission import RequestPermission
from backend.common.security.rbac import DependsRBAC
from backend.database.db import CurrentSession
from backend.app.admin.schema.user import (
    GetUserInfoDetail,
    AddUserParam,
    UpdateUserParam,
)
from backend.app.admin.service.user_service import user_service

router = APIRouter()


@router.get(
    '/{pk}',
    summary='Get user details',
    dependencies=[DependsJwtAuth],
)
async def get_user(
    pk: Annotated[int, Path(description='User ID')],
) -> ResponseSchemaModel[GetUserInfoDetail]:
    """
    Get user information by ID.

    - **pk**: User primary key
    """
    user = await user_service.get(pk=pk)
    return ResponseSchemaModel[GetUserInfoDetail](data=user)


@router.get(
    '',
    summary='Get user list (paginated)',
    dependencies=[
        DependsJwtAuth,
        DependsRBAC,
    ],
)
async def get_users(
    db: CurrentSession,
    username: Annotated[str | None, Query(description='Username filter')] = None,
    status: Annotated[int | None, Query(description='Status filter')] = None,
) -> ResponseSchemaModel[list[GetUserInfoDetail]]:
    """Get paginated user list with optional filters."""
    users = await user_service.get_list(
        db=db,
        username=username,
        status=status,
    )
    return ResponseSchemaModel[list[GetUserInfoDetail]](data=users)


@router.post(
    '',
    summary='Create user',
    dependencies=[
        DependsJwtAuth,
        DependsRBAC,
        Depends(RequestPermission('sys:user:add')),
    ],
)
async def create_user(
    obj: AddUserParam,
) -> ResponseModel:
    """Create a new user."""
    await user_service.create(obj=obj)
    return ResponseModel(msg='User created successfully')


@router.put(
    '/{pk}',
    summary='Update user',
    dependencies=[
        DependsJwtAuth,
        DependsRBAC,
        Depends(RequestPermission('sys:user:edit')),
    ],
)
async def update_user(
    pk: Annotated[int, Path(description='User ID')],
    obj: UpdateUserParam,
) -> ResponseModel:
    """Update user information."""
    await user_service.update(pk=pk, obj=obj)
    return ResponseModel(msg='User updated successfully')


@router.delete(
    '/{pk}',
    summary='Delete user',
    dependencies=[
        DependsJwtAuth,
        DependsRBAC,
        Depends(RequestPermission('sys:user:del')),
    ],
)
async def delete_user(
    pk: Annotated[int, Path(description='User ID')],
) -> ResponseModel:
    """Delete a user by ID."""
    await user_service.delete(pk=pk)
    return ResponseModel(msg='User deleted successfully')
```

### Checklist

- [ ] Use `async def` for all endpoints
- [ ] Type annotate all parameters with `Annotated`
- [ ] Use `Path()`, `Query()`, `Body()` with descriptions
- [ ] Return `ResponseModel` or `ResponseSchemaModel[T]`
- [ ] Apply authentication via `DependsJwtAuth`
- [ ] Apply authorization via `DependsRBAC` or `RequestPermission()`
- [ ] Add `summary` to all routes
- [ ] Include docstrings for complex endpoints

### Anti-Patterns

```python
# ❌ Wrong: Business logic in endpoint
@router.get('/users')
async def get_users(db: CurrentSession):
    users = await db.execute(select(User))  # Direct DB access
    for user in users:
        if user.is_admin:  # Business logic
            user.permissions = ['all']
    return users

# ❌ Wrong: Missing type annotations
@router.get('/users/{pk}')
async def get_user(pk):  # No type hint
    return await user_service.get(pk)

# ❌ Wrong: Not using ResponseModel
@router.get('/users')
async def get_users():
    return {'data': users}  # Raw dict response
```

---

## 2. Service Layer (`service/`)

### Responsibilities
- ✅ Implement business logic
- ✅ Validate business rules
- ✅ Orchestrate CRUD operations
- ✅ Handle cache invalidation
- ✅ Call external APIs
- ❌ No HTTP handling
- ❌ No direct session creation

### Standard Pattern

```python
from typing import Any

from sqlalchemy.ext.asyncio import AsyncSession

from backend.common.exception.errors import (
    NotFoundError,
    ConflictError,
    ForbiddenError,
)
from backend.app.admin.crud.crud_user import user_dao
from backend.app.admin.schema.user import AddUserParam, UpdateUserParam


class UserService:
    """User business logic service."""

    @staticmethod
    async def get(*, pk: int) -> dict[str, Any]:
        """
        Get user by primary key.

        Args:
            pk: User primary key

        Returns:
            User information dictionary

        Raises:
            NotFoundError: User not found
        """
        user = await user_dao.get(pk)
        if not user:
            raise NotFoundError(msg=f'User {pk} not found')
        return user

    @staticmethod
    async def get_list(
        *,
        db: AsyncSession,
        username: str | None = None,
        status: int | None = None,
    ) -> list[dict[str, Any]]:
        """Get filtered user list."""
        return await user_dao.get_list(
            db,
            username=username,
            status=status,
        )

    @staticmethod
    async def create(*, obj: AddUserParam) -> None:
        """
        Create new user.

        Args:
            obj: User creation parameters

        Raises:
            ConflictError: Username already exists
        """
        # Validate uniqueness
        existing = await user_dao.get_by_username(obj.username)
        if existing:
            raise ConflictError(msg=f'Username {obj.username} already exists')

        # Validate department exists
        if obj.dept_id:
            dept = await dept_dao.get(obj.dept_id)
            if not dept:
                raise NotFoundError(msg=f'Department {obj.dept_id} not found')

        # Create user
        await user_dao.add(obj)

    @staticmethod
    async def update(*, pk: int, obj: UpdateUserParam) -> None:
        """
        Update existing user.

        Args:
            pk: User primary key
            obj: Update parameters

        Raises:
            NotFoundError: User not found
            ConflictError: Username conflict
        """
        user = await user_dao.get(pk)
        if not user:
            raise NotFoundError(msg=f'User {pk} not found')

        # Check username conflict
        if obj.username and obj.username != user.username:
            existing = await user_dao.get_by_username(obj.username)
            if existing:
                raise ConflictError(msg=f'Username {obj.username} already exists')

        await user_dao.update(pk, obj)

        # Invalidate cache
        await redis_client.delete(f'user:{pk}')

    @staticmethod
    async def delete(*, pk: int) -> None:
        """
        Delete user.

        Args:
            pk: User primary key

        Raises:
            NotFoundError: User not found
            ForbiddenError: Cannot delete super admin
        """
        user = await user_dao.get(pk)
        if not user:
            raise NotFoundError(msg=f'User {pk} not found')

        if user.is_superuser:
            raise ForbiddenError(msg='Cannot delete super admin')

        await user_dao.delete(pk)

        # Invalidate cache
        await redis_client.delete(f'user:{pk}')
        await redis_client.delete(f'token:{user.uuid}')

    @staticmethod
    async def update_password(
        *,
        pk: int,
        old_password: str,
        new_password: str,
    ) -> None:
        """Update user password with validation."""
        user = await user_dao.get_with_password(pk)
        if not user:
            raise NotFoundError(msg=f'User {pk} not found')

        # Verify old password
        if not password_security.verify(old_password, user.password):
            raise ForbiddenError(msg='Invalid current password')

        # Update password
        hashed = password_security.hash(new_password)
        await user_dao.reset_password(pk, hashed)


# Singleton instance
user_service = UserService()
```

### Checklist

- [ ] Use `@staticmethod` for stateless operations
- [ ] Validate before performing operations
- [ ] Raise appropriate exceptions (`NotFoundError`, `ConflictError`, `ForbiddenError`)
- [ ] Handle cache invalidation after mutations
- [ ] Document with docstrings including Args, Returns, Raises
- [ ] Use keyword-only arguments (`*`) for clarity
- [ ] Provide singleton instance

### Anti-Patterns

```python
# ❌ Wrong: Direct session creation
class UserService:
    def get_user(self, pk: int):
        db = SessionLocal()  # Don't create sessions
        try:
            return db.query(User).get(pk)
        finally:
            db.close()

# ❌ Wrong: No validation before operation
class UserService:
    async def delete(self, pk: int):
        await user_dao.delete(pk)  # No existence check

# ❌ Wrong: HTTP handling in service
class UserService:
    def get_user(self, request: Request):  # No HTTP objects
        auth = request.headers.get('Authorization')
        return self.repo.get(auth.user_id)
```

---

## 3. CRUD Layer (`crud/`)

### Responsibilities
- ✅ Encapsulate all database operations
- ✅ Provide type-safe query methods
- ✅ Handle joins and relationships
- ✅ Manage transactions
- ❌ No business logic
- ❌ No validation beyond data integrity

### Standard Pattern

```python
from typing import Sequence

from sqlalchemy import select, Select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload

from sqlalchemy_crud_plus import CRUDPlus

from backend.app.admin.model.user import User
from backend.app.admin.model.dept import Dept
from backend.app.admin.schema.user import AddUserParam, UpdateUserParam
from backend.common.security.password import password_security


class CRUDUser(CRUDPlus[User]):
    """User CRUD operations."""

    async def get(
        self,
        db: AsyncSession,
        pk: int,
    ) -> User | None:
        """Get user by primary key."""
        return await self.select_model(db, pk)

    async def get_by_username(
        self,
        db: AsyncSession,
        username: str,
    ) -> User | None:
        """Get user by username."""
        return await self.select_model_by_column(
            db,
            username=username,
        )

    async def get_with_relations(
        self,
        db: AsyncSession,
        pk: int,
    ) -> User | None:
        """Get user with department and roles loaded."""
        stmt = (
            select(User)
            .where(User.id == pk)
            .options(
                selectinload(User.dept),
                selectinload(User.roles),
            )
        )
        result = await db.execute(stmt)
        return result.scalar_one_or_none()

    def get_select(
        self,
        *,
        username: str | None = None,
        status: int | None = None,
        dept_id: int | None = None,
    ) -> Select:
        """Build filtered select statement."""
        stmt = (
            select(self.model)
            .join(Dept, Dept.id == self.model.dept_id, isouter=True)
        )

        filters = []
        if username:
            filters.append(self.model.username.contains(username))
        if status is not None:
            filters.append(self.model.status == status)
        if dept_id:
            filters.append(self.model.dept_id == dept_id)

        if filters:
            stmt = stmt.where(*filters)

        return stmt.order_by(self.model.created_time.desc())

    async def get_list(
        self,
        db: AsyncSession,
        *,
        username: str | None = None,
        status: int | None = None,
    ) -> Sequence[User]:
        """Get filtered user list."""
        stmt = self.get_select(username=username, status=status)
        result = await db.execute(stmt)
        return result.scalars().all()

    async def add(
        self,
        db: AsyncSession,
        obj: AddUserParam,
    ) -> User:
        """Create new user."""
        # Hash password
        hashed_password = password_security.hash(obj.password)

        user = User(
            username=obj.username,
            password=hashed_password,
            nickname=obj.nickname,
            email=obj.email,
            dept_id=obj.dept_id,
        )

        db.add(user)
        await db.flush()

        # Add roles
        if obj.roles:
            await self._add_roles(db, user.id, obj.roles)

        return user

    async def update(
        self,
        db: AsyncSession,
        pk: int,
        obj: UpdateUserParam,
    ) -> User:
        """Update existing user."""
        user = await self.get(db, pk)

        # Update fields
        update_data = obj.model_dump(exclude_unset=True)
        for field, value in update_data.items():
            if field != 'roles':
                setattr(user, field, value)

        # Update roles if provided
        if obj.roles is not None:
            await self._sync_roles(db, pk, obj.roles)

        await db.flush()
        return user

    async def delete(
        self,
        db: AsyncSession,
        pk: int,
    ) -> None:
        """Delete user and related data."""
        # Delete related OAuth records
        await self._delete_oauth_records(db, pk)

        # Delete user
        await self.delete_model(db, pk)

    async def set_status(
        self,
        db: AsyncSession,
        pk: int,
        status: int,
    ) -> None:
        """Update user status."""
        await self.update_model(db, pk, {'status': status})

    async def reset_password(
        self,
        db: AsyncSession,
        pk: int,
        hashed_password: str,
    ) -> None:
        """Reset user password."""
        await self.update_model(
            db,
            pk,
            {
                'password': hashed_password,
                'password_changed_time': datetime.now(timezone.utc),
            },
        )

    async def _add_roles(
        self,
        db: AsyncSession,
        user_id: int,
        role_ids: list[int],
    ) -> None:
        """Add roles to user."""
        for role_id in role_ids:
            stmt = user_role.insert().values(user_id=user_id, role_id=role_id)
            await db.execute(stmt)

    async def _sync_roles(
        self,
        db: AsyncSession,
        user_id: int,
        role_ids: list[int],
    ) -> None:
        """Sync user roles (replace all)."""
        # Remove existing
        await db.execute(
            user_role.delete().where(user_role.c.user_id == user_id)
        )
        # Add new
        await self._add_roles(db, user_id, role_ids)


# Singleton instance
user_dao = CRUDUser(User)
```

### Checklist

- [ ] Extend `CRUDPlus[Model]` for type safety
- [ ] Use async/await for all database operations
- [ ] Provide `get_select()` for reusable query building
- [ ] Handle relationships with `selectinload` or joins
- [ ] Use `flush()` instead of `commit()` (let caller manage transactions)
- [ ] Provide singleton DAO instance

### Anti-Patterns

```python
# ❌ Wrong: Business logic in CRUD
class CRUDUser(CRUDPlus[User]):
    async def get(self, db, pk):
        user = await self.select_model(db, pk)
        if user and user.is_disabled:  # Business logic
            raise ForbiddenError('User is disabled')
        return user

# ❌ Wrong: Committing in CRUD (should use flush)
class CRUDUser(CRUDPlus[User]):
    async def add(self, db, obj):
        user = User(**obj.model_dump())
        db.add(user)
        await db.commit()  # Don't commit, use flush

# ❌ Wrong: Raw SQL without ORM
class CRUDUser:
    async def get_users(self, db):
        result = await db.execute(
            "SELECT * FROM users WHERE status = 1"  # Use ORM
        )
```

---

## 4. Schema Layer (`schema/`)

### Responsibilities
- ✅ Define request/response data structures
- ✅ Validate input data
- ✅ Transform data between layers
- ✅ Document API contracts

### Standard Pattern

```python
from datetime import datetime
from typing import Annotated

from pydantic import BaseModel, ConfigDict, Field, EmailStr, model_validator
from pydantic.functional_serializers import PlainSerializer


class SchemaBase(BaseModel):
    """Base schema with common configuration."""

    model_config = ConfigDict(
        from_attributes=True,
        str_strip_whitespace=True,
    )


# --- Request Schemas ---

class AddUserParam(SchemaBase):
    """User creation request."""

    username: Annotated[
        str,
        Field(min_length=3, max_length=32, description='Username'),
    ]
    password: Annotated[
        str,
        Field(min_length=8, max_length=128, description='Password'),
    ]
    nickname: Annotated[
        str | None,
        Field(max_length=64, description='Display name'),
    ] = None
    email: Annotated[
        EmailStr | None,
        Field(description='Email address'),
    ] = None
    phone: Annotated[
        str | None,
        Field(max_length=20, pattern=r'^\+?[0-9]{10,15}$', description='Phone number'),
    ] = None
    dept_id: Annotated[
        int | None,
        Field(description='Department ID'),
    ] = None
    roles: Annotated[
        list[int],
        Field(description='Role IDs'),
    ] = []


class UpdateUserParam(SchemaBase):
    """User update request (partial)."""

    username: Annotated[
        str | None,
        Field(min_length=3, max_length=32, description='Username'),
    ] = None
    nickname: Annotated[
        str | None,
        Field(max_length=64, description='Display name'),
    ] = None
    email: Annotated[
        EmailStr | None,
        Field(description='Email address'),
    ] = None
    phone: Annotated[
        str | None,
        Field(max_length=20, description='Phone number'),
    ] = None
    dept_id: Annotated[
        int | None,
        Field(description='Department ID'),
    ] = None
    roles: Annotated[
        list[int] | None,
        Field(description='Role IDs'),
    ] = None
    status: Annotated[
        int | None,
        Field(ge=0, le=1, description='Status (0=disabled, 1=enabled)'),
    ] = None


class UpdatePasswordParam(SchemaBase):
    """Password update request."""

    old_password: Annotated[
        str,
        Field(min_length=8, description='Current password'),
    ]
    new_password: Annotated[
        str,
        Field(min_length=8, max_length=128, description='New password'),
    ]

    @model_validator(mode='after')
    def validate_passwords(self):
        if self.old_password == self.new_password:
            raise ValueError('New password must be different from current password')
        return self


# --- Response Schemas ---

class GetUserInfoDetail(SchemaBase):
    """User detail response."""

    id: int
    username: str
    nickname: str | None
    email: str | None
    phone: str | None
    avatar: Annotated[
        str | None,
        PlainSerializer(lambda x: str(x) if x else None),
    ]
    status: int
    is_superuser: bool
    is_staff: bool
    dept_id: int | None
    created_time: datetime
    updated_time: datetime | None


class GetUserInfoWithRelationDetail(GetUserInfoDetail):
    """User detail with relations response."""

    dept: 'GetDeptDetail | None' = None
    roles: list['GetRoleDetail'] = []


class GetCurrentUserInfoDetail(GetUserInfoDetail):
    """Current user detail with transformed relations."""

    dept_name: str | None = None
    role_names: list[str] = []

    @model_validator(mode='before')
    @classmethod
    def transform_relations(cls, data):
        """Transform related objects to simple values."""
        if hasattr(data, 'dept') and data.dept:
            data = dict(data.__dict__)
            data['dept_name'] = data.pop('dept').name
        if hasattr(data, 'roles'):
            data = dict(data.__dict__) if not isinstance(data, dict) else data
            data['role_names'] = [r.name for r in data.pop('roles', [])]
        return data


# --- List Response Schema ---

class GetUserListDetail(SchemaBase):
    """User list item (simplified)."""

    id: int
    username: str
    nickname: str | None
    status: int
    dept_name: str | None = None
```

### Checklist

- [ ] Inherit from `SchemaBase` with `from_attributes=True`
- [ ] Use `Annotated` with `Field()` for all fields
- [ ] Provide `description` for all fields
- [ ] Separate Create, Update, and Response schemas
- [ ] Use `model_validator` for cross-field validation
- [ ] Use `PlainSerializer` for custom serialization
- [ ] Make Update schema fields optional

### Anti-Patterns

```python
# ❌ Wrong: No field descriptions
class AddUserParam(BaseModel):
    username: str  # No description
    password: str

# ❌ Wrong: Using old orm_mode
class UserResponse(BaseModel):
    class Config:
        orm_mode = True  # Use ConfigDict(from_attributes=True)

# ❌ Wrong: Same schema for create and update
class UserParam(BaseModel):  # Should be separate schemas
    username: str
    password: str
```

---

## 5. Model Layer (`model/`)

### Responsibilities
- ✅ Define database table structures
- ✅ Define relationships between tables
- ✅ Provide column metadata
- ❌ No business logic

### Standard Pattern

```python
from datetime import datetime
from typing import TYPE_CHECKING

import sqlalchemy as sa
from sqlalchemy import ForeignKey, String, Text
from sqlalchemy.orm import Mapped, mapped_column, relationship

from backend.common.model import Base, id_key
from backend.database.db import uuid4_str

if TYPE_CHECKING:
    from backend.app.admin.model.dept import Dept
    from backend.app.admin.model.role import Role


class User(Base):
    """User model."""

    __tablename__ = 'sys_user'

    # Primary key (auto-generated)
    id: Mapped[id_key] = mapped_column(init=False)

    # Required fields
    username: Mapped[str] = mapped_column(
        String(32),
        unique=True,
        index=True,
        comment='Username',
    )
    password: Mapped[str] = mapped_column(
        String(255),
        comment='Hashed password',
    )

    # Optional fields
    uuid: Mapped[str] = mapped_column(
        String(50),
        unique=True,
        index=True,
        default_factory=uuid4_str,
        comment='Unique identifier',
    )
    nickname: Mapped[str | None] = mapped_column(
        String(64),
        default=None,
        comment='Display name',
    )
    email: Mapped[str | None] = mapped_column(
        String(255),
        default=None,
        comment='Email address',
    )
    phone: Mapped[str | None] = mapped_column(
        String(20),
        default=None,
        comment='Phone number',
    )
    avatar: Mapped[str | None] = mapped_column(
        Text,
        default=None,
        comment='Avatar URL',
    )

    # Status flags
    status: Mapped[int] = mapped_column(
        default=1,
        comment='Status: 0=disabled, 1=enabled',
    )
    is_superuser: Mapped[bool] = mapped_column(
        default=False,
        comment='Is super admin',
    )
    is_staff: Mapped[bool] = mapped_column(
        default=False,
        comment='Is staff member',
    )
    is_multi_login: Mapped[bool] = mapped_column(
        default=False,
        comment='Allow multiple logins',
    )

    # Foreign keys
    dept_id: Mapped[int | None] = mapped_column(
        sa.BigInteger,
        ForeignKey('sys_dept.id', ondelete='SET NULL'),
        default=None,
        comment='Department ID',
    )

    # Timestamps
    created_time: Mapped[datetime] = mapped_column(
        sa.DateTime(timezone=True),
        init=False,
        server_default=sa.func.now(),
        comment='Creation time',
    )
    updated_time: Mapped[datetime | None] = mapped_column(
        sa.DateTime(timezone=True),
        init=False,
        onupdate=sa.func.now(),
        comment='Last update time',
    )
    last_login_time: Mapped[datetime | None] = mapped_column(
        sa.DateTime(timezone=True),
        init=False,
        default=None,
        comment='Last login time',
    )
    password_changed_time: Mapped[datetime | None] = mapped_column(
        sa.DateTime(timezone=True),
        init=False,
        default=None,
        comment='Password change time',
    )

    # Relationships
    dept: Mapped['Dept | None'] = relationship(
        back_populates='users',
        init=False,
    )
    roles: Mapped[list['Role']] = relationship(
        secondary='sys_user_role',
        back_populates='users',
        init=False,
        default_factory=list,
    )


# Association table for many-to-many
user_role = sa.Table(
    'sys_user_role',
    Base.metadata,
    sa.Column('user_id', sa.BigInteger, ForeignKey('sys_user.id', ondelete='CASCADE')),
    sa.Column('role_id', sa.BigInteger, ForeignKey('sys_role.id', ondelete='CASCADE')),
)
```

### Checklist

- [ ] Use `Mapped[]` type annotations for all columns
- [ ] Provide `comment` for all columns
- [ ] Use `init=False` for auto-generated fields
- [ ] Use `default` or `default_factory` appropriately
- [ ] Define foreign keys with proper `ondelete` behavior
- [ ] Use `TYPE_CHECKING` for relationship type hints
- [ ] Use timezone-aware datetime

### Anti-Patterns

```python
# ❌ Wrong: No column comments
class User(Base):
    username = Column(String(32))  # No comment

# ❌ Wrong: No type annotations
class User(Base):
    username = mapped_column(String(32))  # Missing Mapped[]

# ❌ Wrong: Business logic in model
class User(Base):
    def is_admin(self):  # Put in service layer
        return self.role == 'admin'
```

---

## 6. Exception Handling

### Standard Exceptions

```python
from fastapi import HTTPException
from starlette.background import BackgroundTask

from backend.common.response.response_code import StandardResponseCode


class BaseExceptionError(Exception):
    """Base exception with standard fields."""

    def __init__(
        self,
        *,
        code: int = StandardResponseCode.HTTP_400,
        msg: str = 'Bad Request',
        data: dict | None = None,
        background: BackgroundTask | None = None,
    ):
        self.code = code
        self.msg = msg
        self.data = data
        self.background = background
        super().__init__(msg)


class RequestError(BaseExceptionError):
    """400 Bad Request."""

    def __init__(self, *, msg: str = 'Bad Request', data: dict | None = None):
        super().__init__(code=400, msg=msg, data=data)


class NotFoundError(BaseExceptionError):
    """404 Not Found."""

    def __init__(self, *, msg: str = 'Not Found', data: dict | None = None):
        super().__init__(code=404, msg=msg, data=data)


class ConflictError(BaseExceptionError):
    """409 Conflict."""

    def __init__(self, *, msg: str = 'Conflict', data: dict | None = None):
        super().__init__(code=409, msg=msg, data=data)


class ForbiddenError(BaseExceptionError):
    """403 Forbidden."""

    def __init__(self, *, msg: str = 'Forbidden', data: dict | None = None):
        super().__init__(code=403, msg=msg, data=data)


class AuthorizationError(BaseExceptionError):
    """403 Authorization Failed."""

    def __init__(self, *, msg: str = 'Permission Denied', data: dict | None = None):
        super().__init__(code=403, msg=msg, data=data)


class TokenError(HTTPException):
    """401 Token Invalid."""

    def __init__(self, *, msg: str = 'Token Invalid'):
        super().__init__(
            status_code=401,
            detail=msg,
            headers={'WWW-Authenticate': 'Bearer'},
        )


class ServerError(BaseExceptionError):
    """500 Internal Server Error."""

    def __init__(self, *, msg: str = 'Internal Server Error', data: dict | None = None):
        super().__init__(code=500, msg=msg, data=data)
```

### Exception Handler Registration

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

from backend.common.exception.errors import BaseExceptionError


def register_exception_handlers(app: FastAPI) -> None:
    """Register global exception handlers."""

    @app.exception_handler(BaseExceptionError)
    async def base_exception_handler(
        request: Request,
        exc: BaseExceptionError,
    ) -> JSONResponse:
        return JSONResponse(
            status_code=exc.code,
            content={
                'code': exc.code,
                'msg': exc.msg,
                'data': exc.data,
            },
            background=exc.background,
        )
```

---

## 7. Response Schema

### Standard Response Pattern

```python
from typing import Any, Generic, TypeVar

from pydantic import BaseModel

from backend.common.response.response_code import StandardResponseCode

SchemaT = TypeVar('SchemaT')


class ResponseModel(BaseModel):
    """Standard response without typed data."""

    code: int = StandardResponseCode.HTTP_200
    msg: str = 'Success'
    data: Any | None = None


class ResponseSchemaModel(ResponseModel, Generic[SchemaT]):
    """Typed response with generic data."""

    data: SchemaT | None = None


class ResponseBase:
    """Response factory methods."""

    @staticmethod
    def success(
        *,
        msg: str = 'Success',
        data: Any | None = None,
    ) -> ResponseModel:
        return ResponseModel(
            code=StandardResponseCode.HTTP_200,
            msg=msg,
            data=data,
        )

    @staticmethod
    def fail(
        *,
        code: int = StandardResponseCode.HTTP_400,
        msg: str = 'Failed',
        data: Any | None = None,
    ) -> ResponseModel:
        return ResponseModel(code=code, msg=msg, data=data)
```

---

## 8. Database Configuration

### Async Engine and Session

```python
from typing import Annotated, AsyncGenerator

from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from fastapi import Depends

from backend.core.config import settings


def create_database_url() -> str:
    """Create async database URL."""
    if settings.DB_TYPE == 'mysql':
        return (
            f'mysql+asyncmy://{settings.DB_USER}:{settings.DB_PASSWORD}'
            f'@{settings.DB_HOST}:{settings.DB_PORT}/{settings.DB_NAME}'
            f'?charset=utf8mb4'
        )
    elif settings.DB_TYPE == 'postgresql':
        return (
            f'postgresql+asyncpg://{settings.DB_USER}:{settings.DB_PASSWORD}'
            f'@{settings.DB_HOST}:{settings.DB_PORT}/{settings.DB_NAME}'
        )
    raise ValueError(f'Unsupported database type: {settings.DB_TYPE}')


# Create async engine
async_engine = create_async_engine(
    create_database_url(),
    echo=settings.DB_ECHO,
    pool_size=10,
    max_overflow=20,
    pool_timeout=30,
    pool_recycle=3600,
    pool_pre_ping=True,
)

# Session factory
async_session_factory = async_sessionmaker(
    bind=async_engine,
    class_=AsyncSession,
    autoflush=False,
    expire_on_commit=False,
)


async def get_db() -> AsyncGenerator[AsyncSession, None]:
    """Dependency for database session."""
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise


async def get_db_transaction() -> AsyncGenerator[AsyncSession, None]:
    """Dependency for explicit transaction."""
    async with async_session_factory() as session:
        async with session.begin():
            yield session


# Type aliases for dependency injection
CurrentSession = Annotated[AsyncSession, Depends(get_db)]
CurrentSessionTransaction = Annotated[AsyncSession, Depends(get_db_transaction)]
```

---

## 9. Security Patterns

### Password Hashing

```python
import bcrypt


class PasswordSecurity:
    """Password hashing utilities."""

    @staticmethod
    def hash(password: str) -> str:
        """Hash password with bcrypt."""
        salt = bcrypt.gensalt()
        return bcrypt.hashpw(password.encode(), salt).decode()

    @staticmethod
    def verify(plain_password: str, hashed_password: str) -> bool:
        """Verify password against hash."""
        return bcrypt.checkpw(
            plain_password.encode(),
            hashed_password.encode(),
        )


password_security = PasswordSecurity()
```

### JWT Dependencies

```python
from typing import Annotated

from fastapi import Depends, Security
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl='/api/v1/auth/login')


async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
) -> User:
    """Validate token and return current user."""
    payload = jwt.decode(token, settings.SECRET_KEY, algorithms=['HS256'])
    user = await user_dao.get(payload['sub'])
    if not user:
        raise TokenError(msg='User not found')
    return user


async def get_current_superuser(
    user: Annotated[User, Depends(get_current_user)],
) -> User:
    """Require superuser privileges."""
    if not user.is_superuser:
        raise ForbiddenError(msg='Superuser required')
    return user


# Dependency shortcuts
DependsJwtAuth = Depends(get_current_user)
DependsSuperUser = Depends(get_current_superuser)
```

---

## Code Quality Tools

### Ruff Configuration

```toml
# pyproject.toml
[tool.ruff]
line-length = 120
target-version = "py310"

[tool.ruff.lint]
select = [
    "E",      # pycodestyle errors
    "W",      # pycodestyle warnings
    "F",      # Pyflakes
    "I",      # isort
    "B",      # flake8-bugbear
    "C4",     # flake8-comprehensions
    "UP",     # pyupgrade
    "SIM",    # flake8-simplify
]
ignore = [
    "E501",   # line too long (handled by formatter)
    "B008",   # function call in default argument
]

[tool.ruff.lint.isort]
known-first-party = ["backend"]
```

---

## Checklist Summary

### API Layer
- [ ] Async endpoints with type hints
- [ ] Annotated parameters with descriptions
- [ ] ResponseModel/ResponseSchemaModel returns
- [ ] Proper auth/permission dependencies

### Service Layer
- [ ] Static methods for stateless operations
- [ ] Validation before operations
- [ ] Appropriate exception raising
- [ ] Cache invalidation

### CRUD Layer
- [ ] Extend CRUDPlus[Model]
- [ ] Async operations
- [ ] Flush instead of commit
- [ ] Singleton DAO instances

### Schema Layer
- [ ] from_attributes=True
- [ ] Field descriptions
- [ ] Separate CRUD schemas
- [ ] Model validators

### Model Layer
- [ ] Mapped[] type hints
- [ ] Column comments
- [ ] Proper relationships
- [ ] Timezone-aware datetimes

---

## References

- [FastAPI Best Architecture](https://github.com/fastapi-practices/fastapi_best_architecture)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [SQLAlchemy 2.0](https://docs.sqlalchemy.org/en/20/)
- [Pydantic v2](https://docs.pydantic.dev/latest/)
- [Ruff Linter](https://docs.astral.sh/ruff/)

---

## Using This Skill

Use this skill when you need to:

- Design new FastAPI modules
- Write API endpoints following best practices
- Implement service layer business logic
- Create CRUD operations for database access
- Define Pydantic schemas for validation
- Review code for architecture compliance

Claude will automatically apply these patterns to ensure generated code follows enterprise-level FastAPI architecture.
