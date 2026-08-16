---
title: SQLAlchemy基本功能
# tags:
#   - nodejs
date: '2023-11-04'
summary: Python 世界里的一个现代、高性能 Web API 开发框架，
---

SQLAlchemy 是 Python 生态里最主流的数据库工具之一。它不只是一个 ORM，更准确地说，它由 **SQL Expression Language + ORM + Engine/Connection + Transaction** 等部分组成。

如果你熟悉 Java 的 **Spring Boot + JPA/Hibernate + MyBatis**，可以这样理解：

| SQLAlchemy              | Java 中类似                     |
| ----------------------- | ---------------------------- |
| Engine                  | DataSource / Connection Pool |
| Connection              | JDBC Connection              |
| SQL Expression Language | MyBatis / JDBC SQL           |
| ORM                     | JPA / Hibernate              |
| Session                 | EntityManager                |
| Model                   | Entity                       |
| relationship            | `@OneToMany` / `@ManyToOne`  |
| Alembic                 | Flyway / Liquibase           |

## 1. 数据库连接

SQLAlchemy 可以管理数据库连接：

```python
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql+psycopg2://user:password@localhost/mydb"
)
```

支持很多数据库，例如：

* PostgreSQL
* MySQL
* SQLite
* Oracle
* SQL Server

而且自带 **Connection Pooling**。

例如：

```python
engine = create_engine(
    "mysql+pymysql://user:password@localhost/test",
    pool_size=10,
    max_overflow=20
)
```

这相当于 Java 中配置 HikariCP。

---

# 2. 执行原生 SQL

SQLAlchemy 可以直接执行 SQL。

```python
from sqlalchemy import text

with engine.connect() as conn:
    result = conn.execute(
        text("SELECT * FROM users WHERE age > :age"),
        {"age": 18}
    )

    for row in result:
        print(row)
```

这一点比较像：

```java
jdbcTemplate.query(...)
```

或者 MyBatis：

```xml
<select id="findUsers">
    SELECT * FROM users WHERE age > #{age}
</select>
```

---

# 3. SQL Expression Language

这是 SQLAlchemy 非常重要的功能。

你可以不用 ORM，直接用 Python 构造 SQL。

例如：

```python
from sqlalchemy import select

stmt = select(user_table).where(
    user_table.c.age > 18
)

with engine.connect() as conn:
    result = conn.execute(stmt)
```

SQLAlchemy 会生成类似：

```sql
SELECT *
FROM users
WHERE age > 18
```

复杂查询也可以组合：

```python
stmt = (
    select(users)
    .where(users.c.age > 18)
    .order_by(users.c.name)
    .limit(10)
)
```

所以 SQLAlchemy 并不是简单的 ORM。

---

# 4. ORM

这是大家最熟悉的功能。

定义 Model：

```python
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    age: Mapped[int]
```

这相当于 Java：

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    private Long id;

    private String name;

    private Integer age;
}
```

---

# 5. CRUD

SQLAlchemy ORM 可以完成完整 CRUD。

### Create

```python
user = User(
    name="Vincent",
    age=30
)

session.add(user)
session.commit()
```

### Read

```python
users = session.query(User).all()
```

现代 SQLAlchemy 更推荐：

```python
stmt = select(User)

users = session.scalars(stmt).all()
```

### Update

```python
user = session.get(User, 1)

user.age = 35

session.commit()
```

### Delete

```python
user = session.get(User, 1)

session.delete(user)

session.commit()
```

---

# 6. 条件查询

例如：

```python
stmt = (
    select(User)
    .where(User.age >= 18)
    .where(User.name == "Vincent")
)

users = session.scalars(stmt).all()
```

对应：

```sql
SELECT *
FROM users
WHERE age >= 18
AND name = 'Vincent';
```

还可以：

```python
from sqlalchemy import or_

stmt = select(User).where(
    or_(
        User.age < 18,
        User.age > 60
    )
)
```

---

# 7. JOIN

SQLAlchemy 对 JOIN 支持非常完整。

例如：

```python
stmt = (
    select(User, Order)
    .join(Order, User.id == Order.user_id)
)
```

相当于：

```sql
SELECT *
FROM users u
JOIN orders o
    ON u.id = o.user_id;
```

也支持：

* INNER JOIN
* LEFT JOIN
* RIGHT JOIN
* OUTER JOIN
* 多表 JOIN
* 子查询
* CTE

---

# 8. 一对多 / 多对一关系

例如：

```text
User
 │
 ├── Order
 ├── Order
 └── Order
```

定义：

```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)

    orders: Mapped[list["Order"]] = relationship(
        back_populates="user"
    )


class Order(Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True)

    user_id: Mapped[int] = mapped_column(
        ForeignKey("users.id")
    )

    user: Mapped["User"] = relationship(
        back_populates="orders"
    )
```

然后：

```python
user.orders
```

就可以获取用户订单。

这非常类似 Hibernate：

```java
@OneToMany
private List<Order> orders;
```

---

# 9. Lazy Loading / Eager Loading

SQLAlchemy 支持不同的数据加载策略。

例如：

```python
select(User).options(
    selectinload(User.orders)
)
```

常见策略：

```text
lazy loading
selectin loading
joined loading
subquery loading
```

这对于解决 **N+1 Query** 问题非常重要。

例如：

```text
查询 100 个 User
       ↓
每个 User 再查询一次 Order
       ↓
101 次 SQL
```

这就是典型的 N+1。

SQLAlchemy 可以通过：

```python
selectinload(User.orders)
```

把它优化成少量 SQL。

---

# 10. Transaction 事务

SQLAlchemy 支持事务。

```python
with Session(engine) as session:

    with session.begin():

        user = User(name="Tom", age=20)

        session.add(user)
```

如果发生异常：

```text
Exception
   ↓
Rollback
```

正常完成：

```text
commit
```

这和 Spring：

```java
@Transactional
```

的思想非常接近。

---

# 11. Transaction Isolation

可以配置：

```text
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

例如：

```python
engine = create_engine(
    url,
    isolation_level="REPEATABLE READ"
)
```

对于你之前学习的数据库事务、并发控制，这部分非常值得掌握。

---

# 12. Connection Pool

SQLAlchemy 内置连接池。

例如：

```python
engine = create_engine(
    url,
    pool_size=10,
    max_overflow=20,
    pool_timeout=30,
    pool_recycle=1800
)
```

主要参数：

```text
pool_size
max_overflow
pool_timeout
pool_recycle
pool_pre_ping
```

这部分可以直接和你熟悉的 **HikariCP** 对比学习。

---

# 13. Schema / Table 定义

SQLAlchemy 可以定义数据库表：

```python
from sqlalchemy import Table, Column
from sqlalchemy import Integer, String

users = Table(
    "users",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(100)),
    Column("age", Integer)
)
```

所以它也可以作为一个 **SQL Schema Definition 工具**。

---

# 14. Index

可以定义索引：

```python
from sqlalchemy import Index

Index(
    "idx_user_name",
    users.c.name
)
```

也可以联合索引：

```python
Index(
    "idx_user_name_age",
    users.c.name,
    users.c.age
)
```

对应：

```sql
CREATE INDEX idx_user_name_age
ON users(name, age);
```

---

# 15. Constraint

支持：

```text
Primary Key
Foreign Key
Unique
Check
Not Null
```

例如：

```python
age = mapped_column(
    Integer,
    CheckConstraint("age >= 0")
)
```

---

# 16. 数据库迁移

SQLAlchemy 本身主要负责数据库访问，数据库 Migration 通常搭配：

**Alembic**

例如：

```bash
alembic revision --autogenerate
```

然后：

```bash
alembic upgrade head
```

如果你熟悉 Java：

```text
SQLAlchemy + Alembic
        ≈
Hibernate/JPA + Flyway/Liquibase
```

---

# 17. Async 异步数据库

现代 Python 后端非常重要。

SQLAlchemy 支持 Async API：

```python
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(
    "postgresql+asyncpg://user:password@localhost/test"
)
```

然后：

```python
async with AsyncSession(engine) as session:

    result = await session.execute(
        select(User)
    )

    users = result.scalars().all()
```

这在：

```text
FastAPI
+
SQLAlchemy
+
PostgreSQL
```

架构中非常常见。

---

# 18. SQLAlchemy 的整体架构

你可以把它理解成：

```text
                    SQLAlchemy
                        │
          ┌─────────────┼─────────────┐
          │             │             │
        Engine      SQL Expression    ORM
          │             │             │
     Connection       select()       Session
          │             │             │
    Connection Pool    JOIN          Model
          │             │             │
          └─────────────┼─────────────┘
                        │
                   DB Driver
                        │
          ┌─────────────┼─────────────┐
          │             │             │
       PostgreSQL      MySQL        SQLite
```

---
