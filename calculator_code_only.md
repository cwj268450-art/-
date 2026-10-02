# 计算器系统完整代码

> 使用前请修改代码中的 GitHub 用户名、仓库地址等 GitHub
> 信息，然后即可上传到自己的 GitHub 使用。

## `calculator-backend/src/main.py`

``` python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from .database import init_db
from .routes.calculate import router as calculate_router
from .routes.history import router as history_router

app = FastAPI(title="Calculator Backend", version="1.0.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=False,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(calculate_router)
app.include_router(history_router)


@app.on_event("startup")
def startup():
    init_db()


@app.get("/api/health")
def health():
    return {"status": "ok", "service": "calculator-backend"}
```

## `calculator-backend/src/calculator.py`

``` python
"""安全数学表达式解析器，不使用 eval/exec。"""
import math
import re


class CalculationError(Exception):
    """可预期的计算错误。"""


class Parser:
    NUMBER = re.compile(r"(?:\d+(?:\.\d*)?|\.\d+)(?:[eE][+-]?\d+)?")

    def __init__(self, expression: str):
        self.text = expression.replace("×", "*").replace("÷", "/")
        self.pos = 0

    def parse(self):
        if not self.text.strip():
            raise CalculationError("表达式不能为空")
        if len(self.text) > 200:
            raise CalculationError("表达式过长")
        value = self.expression()
        self.skip()
        if self.pos != len(self.text):
            raise CalculationError("表达式格式错误或存在非法字符")
        if not math.isfinite(value):
            raise CalculationError("计算结果超出范围")
        return int(value) if value == int(value) else round(value, 12)

    def skip(self):
        while self.pos < len(self.text) and self.text[self.pos].isspace():
            self.pos += 1

    def match(self, ch):
        self.skip()
        if self.text.startswith(ch, self.pos):
            self.pos += len(ch)
            return True
        return False

    def expression(self):
        value = self.term()
        while True:
            if self.match("+"):
                value += self.term()
            elif self.match("-"):
                value -= self.term()
            else:
                return value

    def term(self):
        value = self.unary()
        while True:
            if self.match("*"):
                value *= self.unary()
            elif self.match("/"):
                divisor = self.unary()
                if divisor == 0:
                    raise CalculationError("除数不能为 0")
                value /= divisor
            else:
                return value

    def unary(self):
        if self.match("+"):
            return +self.unary()
        if self.match("-"):
            return -self.unary()
        return self.primary()

    def primary(self):
        self.skip()
        if self.match("("):
            value = self.expression()
            if not self.match(")"):
                raise CalculationError("缺少右括号")
            return value

        m = self.NUMBER.match(self.text, self.pos)
        if not m:
            raise CalculationError("需要数字或左括号")
        self.pos = m.end()
        return float(m.group())


def calculate_expression(expression: str):
    if not isinstance(expression, str):
        raise CalculationError("expression 必须是字符串")
    return Parser(expression).parse()
```

## `calculator-backend/src/database.py`

``` python
from pathlib import Path
import sqlite3
from datetime import datetime, timezone

DB_PATH = Path(__file__).resolve().parent.parent / "calculator.db"


def connection():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    return conn


def init_db():
    with connection() as conn:
        conn.execute("""
            CREATE TABLE IF NOT EXISTS calculation_history (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                expression TEXT NOT NULL,
                result TEXT NOT NULL,
                created_at TEXT NOT NULL
            )
        """)
        conn.commit()


def add_history(expression, result):
    now = datetime.now(timezone.utc).astimezone().strftime("%Y-%m-%d %H:%M:%S")
    with connection() as conn:
        cur = conn.execute(
            "INSERT INTO calculation_history(expression,result,created_at) VALUES(?,?,?)",
            (expression, str(result), now),
        )
        conn.commit()
        return cur.lastrowid


def list_history():
    with connection() as conn:
        rows = conn.execute(
            "SELECT id,expression,result,created_at FROM calculation_history "
            "ORDER BY id DESC LIMIT 100"
        ).fetchall()
        return [dict(row) for row in rows]


def delete_history(record_id):
    with connection() as conn:
        cur = conn.execute("DELETE FROM calculation_history WHERE id=?", (record_id,))
        conn.commit()
        return cur.rowcount > 0


def clear_history():
    with connection() as conn:
        cur = conn.execute("DELETE FROM calculation_history")
        conn.commit()
        return cur.rowcount
```

## `calculator-backend/src/schemas.py`

``` python
from pydantic import BaseModel, Field


class CalculateRequest(BaseModel):
    expression: str = Field(..., min_length=1, max_length=200)


class CalculateResponse(BaseModel):
    success: bool
    expression: str
    result: int | float
    history_id: int


class HistoryItem(BaseModel):
    id: int
    expression: str
    result: str
    created_at: str
```

## `calculator-backend/src/routes/calculate.py`

``` python
from fastapi import APIRouter, HTTPException
from ..calculator import CalculationError, calculate_expression
from ..database import add_history
from ..schemas import CalculateRequest, CalculateResponse

router = APIRouter(prefix="/api", tags=["calculate"])


@router.post("/calculate", response_model=CalculateResponse)
def calculate(request: CalculateRequest):
    try:
        result = calculate_expression(request.expression)
        history_id = add_history(request.expression, result)
        return {
            "success": True,
            "expression": request.expression,
            "result": result,
            "history_id": history_id,
        }
    except CalculationError as exc:
        raise HTTPException(
            status_code=400,
            detail={"success": False, "message": str(exc)},
        ) from exc
```

## `calculator-backend/src/routes/history.py`

``` python
from fastapi import APIRouter, HTTPException
from ..database import clear_history, delete_history, list_history
from ..schemas import HistoryItem

router = APIRouter(prefix="/api/history", tags=["history"])


@router.get("", response_model=list[HistoryItem])
def get_history():
    return list_history()


@router.delete("/{record_id}")
def remove_history(record_id: int):
    if not delete_history(record_id):
        raise HTTPException(
            status_code=404,
            detail={"success": False, "message": "历史记录不存在"},
        )
    return {"success": True, "message": "删除成功"}


@router.delete("")
def remove_all_history():
    return {"success": True, "deleted": clear_history()}
```

## `calculator-backend/requirements.txt`

``` text
fastapi==0.117.1
uvicorn[standard]==0.36.0
pydantic==2.11.10
```

## `calculator-backend/.gitignore`

``` text
.venv/
__pycache__/
*.pyc
calculator.db
```

## `calculator-frontend/src/index.html`

``` html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>前后端分离计算器</title>
<link rel="stylesheet" href="style.css">
</head>
<body>
<main class="page">
<section class="card">
<header><div><small>FRONTEND / BACKEND</small><h1>智能计算器</h1>
<p>表达式由 FastAPI 后端安全解析与计算</p></div><span id="status">检查服务中</span></header>
<div class="calc">
<input id="expression" placeholder="例如：(1+2)*3" autocomplete="off">
<div id="result">0</div>
<div class="keys">
<button data-action="clear">AC</button><button data-value="(">(</button>
<button data-value=")">)</button><button class="op" data-value="/">÷</button>
<button data-value="7">7</button><button data-value="8">8</button>
<button data-value="9">9</button><button class="op" data-value="*">×</button>
<button data-value="4">4</button><button data-value="5">5</button>
<button data-value="6">6</button><button class="op" data-value="-">−</button>
<button data-value="1">1</button><button data-value="2">2</button>
<button data-value="3">3</button><button class="op" data-value="+">+</button>
<button data-value="0">0</button><button data-value=".">.</button>
<button data-action="backspace">⌫</button><button class="equal" data-action="calculate">=</button>
</div>
<div class="quick">
<button data-expression="1+2*3">优先级</button>
<button data-expression="(1+2)*3">括号</button>
<button data-expression="-5+8">负数</button>
<button data-expression="1.5*2">小数</button>
</div>
</div>
</section>
<section class="card history"><header><div><small>DATABASE</small><h2>计算历史</h2></div>
<button id="clear-history">清空全部</button></header>
<div id="history">正在加载...</div></section>
</main>
<script src="app.js"></script>
</body>
</html>
```

## `calculator-frontend/src/style.css`

``` css
*{box-sizing:border-box}body{margin:0;min-height:100vh;font-family:"Segoe UI","Microsoft YaHei",sans-serif;background:#f1f5f9;color:#1e293b}.page{width:min(1050px,92vw);margin:auto;padding:40px 0;display:grid;grid-template-columns:1fr 1fr;gap:22px}.card{background:white;border:1px solid #e2e8f0;border-radius:22px;padding:26px;box-shadow:0 16px 40px #0f172a14}header{display:flex;justify-content:space-between;gap:15px;align-items:flex-start}small{color:#4f46e5;font-weight:700;letter-spacing:.12em}h1,h2{margin:6px 0}p{color:#64748b;margin-top:5px}#status{background:#dcfce7;color:#15803d;padding:7px 10px;border-radius:999px;font-size:12px}.calc{margin-top:20px;background:#111827;padding:18px;border-radius:18px}#expression{width:100%;background:none;border:0;outline:0;color:white;text-align:right;font-size:26px;padding:10px}#expression::placeholder{color:#64748b}#result{min-height:62px;text-align:right;color:#a5b4fc;font-size:36px;padding:8px 4px 16px;overflow-wrap:anywhere}.keys{display:grid;grid-template-columns:repeat(4,1fr);gap:9px}.keys button,.quick button,#clear-history{border:0;border-radius:12px;padding:15px;cursor:pointer}.keys button{background:#1f2937;color:white;font-size:20px}.keys .op{background:#3730a3}.keys .equal{background:#4f46e5}.quick{display:grid;grid-template-columns:repeat(4,1fr);gap:7px;margin-top:12px}.quick button{padding:8px;color:#4338ca;background:#eef2ff;font-size:12px}.history{min-height:500px}.history-list,.item{display:grid;gap:10px}.item{grid-template-columns:1fr auto;align-items:center;padding:14px;border:1px solid #e2e8f0;border-radius:12px;margin-top:10px}.expr{font-weight:700}.res{color:#4f46e5;font-weight:700}.time{font-size:12px;color:#94a3b8}.del{border:0;background:#fee2e2;color:#b91c1c;border-radius:8px;padding:7px;cursor:pointer}.empty{text-align:center;color:#94a3b8;padding:40px}#clear-history{background:#fff1f2;color:#be123c}@media(max-width:800px){.page{grid-template-columns:1fr;padding:20px 0}}
```

## `calculator-frontend/src/app.js`

``` javascript
const API_BASE = "http://127.0.0.1:8000/api";
const input = document.getElementById("expression");
const result = document.getElementById("result");
const history = document.getElementById("history");
const status = document.getElementById("status");

function safe(v) {
    return String(v).replaceAll("&","&amp;").replaceAll("<","&lt;")
        .replaceAll(">","&gt;").replaceAll('"',"&quot;").replaceAll("'","&#039;");
}

async function calculate() {
    const expression = input.value.trim();
    if (!expression) { result.textContent = "请输入表达式"; return; }
    result.textContent = "计算中...";
    try {
        const response = await fetch(`${API_BASE}/calculate`, {
            method:"POST",
            headers:{"Content-Type":"application/json"},
            body:JSON.stringify({expression})
        });
        const data = await response.json();
        if (!response.ok) {
            result.textContent = data.detail?.message || "表达式无效";
            return;
        }
        result.textContent = data.result;
        loadHistory();
    } catch (e) {
        result.textContent = "无法连接后端";
    }
}

async function loadHistory() {
    try {
        const response = await fetch(`${API_BASE}/history`);
        const items = await response.json();
        if (!items.length) {
            history.innerHTML = '<div class="empty">暂无计算历史</div>';
            return;
        }
        history.innerHTML = items.map(item => `
            <div class="item">
                <div><div class="expr">${safe(item.expression)}</div>
                <div class="res">= ${safe(item.result)}</div>
                <div class="time">${safe(item.created_at)}</div></div>
                <button class="del" data-id="${item.id}">删除</button>
            </div>`).join("");
    } catch (e) {
        history.innerHTML = '<div class="empty">无法读取历史，请检查后端</div>';
    }
}

async function deleteItem(id) {
    await fetch(`${API_BASE}/history/${id}`, {method:"DELETE"});
    loadHistory();
}

document.querySelectorAll(".keys button").forEach(button => {
    button.onclick = () => {
        const value = button.dataset.value;
        const action = button.dataset.action;
        if (value !== undefined) input.value += value;
        if (action === "clear") { input.value = ""; result.textContent = "0"; }
        if (action === "backspace") input.value = input.value.slice(0,-1);
        if (action === "calculate") calculate();
        input.focus();
    };
});

document.querySelectorAll(".quick button").forEach(button => {
    button.onclick = () => { input.value = button.dataset.expression; calculate(); };
});
history.onclick = e => { if (e.target.classList.contains("del")) deleteItem(e.target.dataset.id); };
document.getElementById("clear-history").onclick = async () => {
    if (confirm("确定清空全部历史吗？")) {
        await fetch(`${API_BASE}/history`, {method:"DELETE"});
        loadHistory();
    }
};
input.onkeydown = e => { if (e.key === "Enter") calculate(); };

async function health() {
    try {
        const response = await fetch(`${API_BASE}/health`);
        status.textContent = response.ok ? "后端在线" : "后端离线";
    } catch (e) { status.textContent = "后端离线"; }
}
health();
loadHistory();
setInterval(health,10000);
```

## `calculator-frontend/.gitignore`

``` text
.DS_Store
Thumbs.db
```

## GitHub 信息修改位置

上传到 GitHub 前，请检查以下位置：

``` text
1. GitHub 用户名：修改为你自己的 GitHub 用户名
2. GitHub 仓库地址：修改为你创建的仓库地址
3. README / 文档中的 GitHub 链接：替换为自己的仓库链接
4. 如果代码中出现作者、项目地址等信息：替换为自己的信息
```
