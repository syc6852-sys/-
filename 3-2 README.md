# 1. 최종 프로젝트 구성

프로젝트 이름은 `mydata-ai`입니다.

- 데이터 주제: 일별 매출 데이터
- 샘플 데이터: 180개
- 백엔드: FastAPI
- 데이터베이스: Firebase Firestore
- AI: OpenAI API
- 프론트엔드: HTML, CSS, JavaScript
- 백엔드 배포: Render
- 프론트엔드 배포: Vercel

## 최종 폴더 구조

아래 구조와 파일명을 정확하게 사용하세요.

text

```text
mydata-ai/
│
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── firebase.py
│   │   │
│   │   ├── models/
│   │   │   ├── __init__.py
│   │   │   └── schemas.py
│   │   │
│   │   ├── routers/
│   │   │   ├── __init__.py
│   │   │   ├── data.py
│   │   │   ├── chat.py
│   │   │   └── conversations.py
│   │   │
│   │   └── services/
│   │       ├── __init__.py
│   │       ├── data_service.py
│   │       ├── analysis_service.py
│   │       ├── ai_service.py
│   │       └── conversation_service.py
│   │
│   ├── requirements.txt
│   └── .env.example
│
├── frontend/
│   ├── index.html
│   ├── style.css
│   ├── script.js
│   ├── build.js
│   ├── package.json
│   └── vercel.json
│
├── .gitignore
└── README.md
```

# 2. 백엔드 전체 코드

## 2-1. `backend/requirements.txt`

text

```text

fastapi==0.115.12
uvicorn[standard]==0.34.0
firebase-admin==6.7.0
openai==1.68.2
python-dotenv==1.0.1
pydantic==2.10.6
```

## 2-2. `backend/.env.example`

dotenv

```dotenv

OPENAI_API_KEY=
OPENAI_MODEL=gpt-4o-mini

FIREBASE_SERVICE_ACCOUNT_JSON=
FIREBASE_SERVICE_ACCOUNT_PATH=

ALLOWED_ORIGINS=http://localhost:5500,http://127.0.0.1:5500
```

이 파일에는 실제 비밀키를 넣지 않습니다.

나중에 실제 환경에서는 `.env` 파일을 만들고 본인의 인증 정보를 입력합니다.

## 2-3. `backend/app/__init__.py`

python

```python

# MyData AI backend package
```

## 2-4. `backend/app/config.py`

python

```python

import os
from pathlib import Path
from dotenv import load_dotenv

BACKEND_DIR = Path(__file__).resolve().parents[1]

load_dotenv(BACKEND_DIR / ".env")

OPENAI_API_KEY = os.getenv("OPENAI_API_KEY", "").strip()

OPENAI_MODEL = os.getenv(
    "OPENAI_MODEL",
    "gpt-4o-mini"
)

FIREBASE_SERVICE_ACCOUNT_JSON = os.getenv(
    "FIREBASE_SERVICE_ACCOUNT_JSON",
    ""
)

FIREBASE_SERVICE_ACCOUNT_PATH = os.getenv(
    "FIREBASE_SERVICE_ACCOUNT_PATH",
    ""
)

ALLOWED_ORIGINS = [
    origin.strip().rstrip("/")
    for origin in os.getenv(
        "ALLOWED_ORIGINS",
        "http://localhost:5500,http://127.0.0.1:5500"
    ).split(",")
    if origin.strip()
]
```

## 2-5. `backend/app/firebase.py`

Firebase 인증 정보를 환경변수 또는 로컬 서비스 계정 파일로 읽습니다.

Firebase가 아직 준비되지 않았더라도 서버 프로세스가 바로 종료되지 않도록 지연 초기화합니다. Firestore가 필요한 요청은 인증 설정이 없으면 명확한 오류를 반환합니다.

python

```python

import json
from pathlib import Path

import firebase_admin
from firebase_admin import credentials, firestore
from fastapi import HTTPException

from app.config import (
    FIREBASE_SERVICE_ACCOUNT_JSON,
    FIREBASE_SERVICE_ACCOUNT_PATH
)


_db = None


def get_db():
    global _db

    if _db is not None:
        return _db

    try:
        if firebase_admin._apps:
            app = firebase_admin.get_app()
        else:
            if FIREBASE_SERVICE_ACCOUNT_JSON:
                service_data = json.loads(
                    FIREBASE_SERVICE_ACCOUNT_JSON
                )
                cred = credentials.Certificate(service_data)

            elif FIREBASE_SERVICE_ACCOUNT_PATH:
                path = Path(FIREBASE_SERVICE_ACCOUNT_PATH)

                if not path.exists():
                    raise RuntimeError(
                        "Firebase 서비스 계정 파일을 찾을 수 없습니다."
                    )

                cred = credentials.Certificate(str(path))

            else:
                raise RuntimeError(
                    "Firebase 인증 환경변수가 설정되지 않았습니다."
                )

            app = firebase_admin.initialize_app(cred)

        _db = firestore.client(app=app)
        return _db

    except Exception as exc:
        raise HTTPException(
            status_code=503,
            detail=f"Firestore 연결 실패: {str(exc)}"
        )
```

## 2-6. `backend/app/models/__init__.py`

python

```python

# Models package
```

## 2-7. `backend/app/models/schemas.py`

날짜, 매출 금액, 메모, 채팅 메시지 등을 검증합니다.

python

```python

from datetime import date
from typing import Literal, Optional

from pydantic import BaseModel, Field, ConfigDict, field_validator


class DataCreate(BaseModel):
    date: date
    value: float = Field(ge=0, le=1000000000000)
    memo: str = Field(default="", max_length=500)

    @field_validator("memo")
    @classmethod
    def clean_memo(cls, value):
        return value.strip()


class DataUpdate(BaseModel):
    date: Optional[date] = None
    value: Optional[float] = Field(
        default=None,
        ge=0,
        le=1000000000000
    )
    memo: Optional[str] = Field(default=None, max_length=500)

    @field_validator("memo")
    @classmethod
    def clean_memo(cls, value):
        return value.strip() if value is not None else value


class ChatMessage(BaseModel):
    role: Literal["user", "assistant"]
    content: str = Field(min_length=1, max_length=4000)


class ChatRequest(BaseModel):
    message: str = Field(min_length=1, max_length=2000)
    conversation_id: Optional[str] = None


class ConversationCreate(BaseModel):
    title: str = Field(default="새 대화", max_length=100)
    messages: list[ChatMessage] = Field(min_length=1, max_length=100)

    model_config = ConfigDict(extra="forbid")
```

## 2-8. `backend/app/services/__init__.py`

python

```python

# Services package
```

## 2-9. `backend/app/services/data_service.py`

python

```python

from datetime import datetime, timezone

from app.firebase import get_db


COLLECTION = "data"


def serialize_document(doc):
    result = doc.to_dict()
    result["id"] = doc.id
    return result


def create_data(data):
    db = get_db()

    document = {
        "date": data.date.isoformat(),
        "value": data.value,
        "memo": data.memo,
        "created_at": datetime.now(timezone.utc).isoformat()
    }

    ref = db.collection(COLLECTION).add(document)

    return {
        "id": ref[1].id,
        **document
    }


def get_all_data():
    db = get_db()

    docs = db.collection(COLLECTION).stream()

    records = [
        serialize_document(doc)
        for doc in docs
    ]

    return sorted(
        records,
        key=lambda item: item["date"]
    )


def update_data(data_id, data):
    db = get_db()

    ref = db.collection(COLLECTION).document(data_id)

    if not ref.get().exists:
        return None

    updates = data.model_dump(exclude_unset=True)

    if "date" in updates and updates["date"] is not None:
        updates["date"] = updates["date"].isoformat()

    if not updates:
        return serialize_document(ref.get())

    updates["updated_at"] = datetime.now(
        timezone.utc
    ).isoformat()

    ref.update(updates)

    return serialize_document(ref.get())


def delete_data(data_id):
    db = get_db()

    ref = db.collection(COLLECTION).document(data_id)

    if not ref.get().exists:
        return False

    ref.delete()

    return True
```

## 2-10. `backend/app/services/analysis_service.py`

최근 7개와 이전 7개 데이터의 평균을 비교합니다.

python

```python

from statistics import mean

from app.services.data_service import get_all_data


def get_summary():
    records = get_all_data()

    if not records:
        return {
            "period": "데이터 없음",
            "count": 0,
            "metrics": {
                "total": 0,
                "average": 0,
                "max": 0,
                "min": 0
            },
            "trend": "분석할 데이터가 없습니다."
        }

    values = [
        float(item["value"])
        for item in records
    ]

    dates = [
        item["date"]
        for item in records
    ]

    trend = "유지"

    if len(values) >= 14:
        recent = mean(values[-7:])
        previous = mean(values[-14:-7])

        if recent > previous * 1.02:
            trend = "상승"

        elif recent < previous * 0.98:
            trend = "하락"

    elif len(values) < 14:
        trend = "데이터 부족 (14개 미만)"

    return {
        "period": f"{dates[0]} ~ {dates[-1]}",
        "count": len(values),
        "metrics": {
            "total": round(sum(values), 2),
            "average": round(mean(values), 2),
            "max": round(max(values), 2),
            "min": round(min(values), 2)
        },
        "trend": trend
    }
```

## 2-11. `backend/app/services/conversation_service.py`

python

```python

from datetime import datetime, timezone

from app.firebase import get_db


COLLECTION = "conversations"


def save_conversation(title, messages, conversation_id=None):
    db = get_db()

    now = datetime.now(timezone.utc).isoformat()

    document = {
        "title": title[:100],
        "messages": messages,
        "updated_at": now
    }

    if conversation_id:
        ref = db.collection(COLLECTION).document(
            conversation_id
        )

        if ref.get().exists:
            ref.update(document)

            return {
                "id": conversation_id,
                **document
            }

    document["created_at"] = now

    ref = db.collection(COLLECTION).add(document)

    return {
        "id": ref[1].id,
        **document
    }


def get_all_conversations():
    db = get_db()

    docs = db.collection(COLLECTION).stream()

    result = []

    for doc in docs:
        data = doc.to_dict()

        result.append({
            "id": doc.id,
            "title": data.get("title", "새 대화"),
            "created_at": data.get("created_at"),
            "updated_at": data.get("updated_at")
        })

    return sorted(
        result,
        key=lambda item: item.get("updated_at") or "",
        reverse=True
    )


def get_conversation(conversation_id):
    db = get_db()

    doc = db.collection(COLLECTION).document(
        conversation_id
    ).get()

    if not doc.exists:
        return None

    return {
        "id": doc.id,
        **doc.to_dict()
    }


def delete_conversation(conversation_id):
    db = get_db()

    ref = db.collection(COLLECTION).document(
        conversation_id
    )

    if not ref.get().exists:
        return False

    ref.delete()

    return True
```

## 2-12. `backend/app/services/ai_service.py`

실제 데이터 요약을 시스템 프롬프트에 삽입하는 핵심 코드입니다.

python

```python

from openai import OpenAI

from app.config import OPENAI_API_KEY, OPENAI_MODEL


def generate_answer(summary, messages):
    if not OPENAI_API_KEY:
        raise RuntimeError(
            "OPENAI_API_KEY 환경변수가 설정되지 않았습니다."
        )

    client = OpenAI(api_key=OPENAI_API_KEY)

    metrics = summary["metrics"]

    system_prompt = f"""
당신은 사용자의 개인 매출 데이터 분석 비서입니다.

다음은 Firestore에 실제 저장된 데이터 분석 결과입니다.

[데이터 요약]
기간: {summary["period"]}
데이터 개수: {summary["count"]}
총 매출: {metrics["total"]}원
평균 매출: {metrics["average"]}원
최대 매출: {metrics["max"]}원
최소 매출: {metrics["min"]}원
최근 추세: {summary["trend"]}

[답변 규칙]
1. 반드시 제공된 데이터를 근거로 답변하세요.
2. 제공되지 않은 월별 실적이나 날짜별 실적을 추측하지 마세요.
3. 정확한 월별 데이터를 모르면 월별 분석이 불가능하다고 알려주세요.
4. 데이터가 없는 경우 데이터가 없다고 알려주세요.
5. 한국어로 친절하게 답변하세요.
6. 필요하다면 매출 개선 방향을 제안하세요.
"""

    api_messages = [
        {
            "role": "system",
            "content": system_prompt
        }
    ]

    for message in messages[-10:]:
        api_messages.append({
            "role": message["role"],
            "content": message["content"]
        })

    response = client.chat.completions.create(
        model=OPENAI_MODEL,
        messages=api_messages,
        max_tokens=700,
        temperature=0.4
    )

    return response.choices[0].message.content
```

## 2-13. `backend/app/routers/__init__.py`

python

```python

# Routers package
```

## 2-14. `backend/app/routers/data.py`

python

```python

from fastapi import APIRouter, HTTPException

from app.models.schemas import DataCreate, DataUpdate
from app.services import data_service
from app.services.analysis_service import get_summary


router = APIRouter(
    prefix="/api/data",
    tags=["Data"]
)


@router.post("")
def create_data(request: DataCreate):
    return data_service.create_data(request)


@router.get("")
def list_data():
    return data_service.get_all_data()


@router.put("/{data_id}")
def update_data(data_id: str, request: DataUpdate):
    result = data_service.update_data(data_id, request)

    if result is None:
        raise HTTPException(
            status_code=404,
            detail="데이터를 찾을 수 없습니다."
        )

    return result


@router.delete("/{data_id}")
def delete_data(data_id: str):
    result = data_service.delete_data(data_id)

    if not result:
        raise HTTPException(
            status_code=404,
            detail="데이터를 찾을 수 없습니다."
        )

    return {
        "message": "데이터가 삭제되었습니다."
    }


@router.get("/summary")
def summary():
    return get_summary()
```

## 2-15. `backend/app/routers/conversations.py`

python

```python

from fastapi import APIRouter, HTTPException

from app.models.schemas import ConversationCreate
from app.services import conversation_service


router = APIRouter(
    prefix="/api/conversations",
    tags=["Conversations"]
)


@router.post("")
def create_conversation(request: ConversationCreate):
    messages = [
        item.model_dump()
        for item in request.messages
    ]

    return conversation_service.save_conversation(
        request.title,
        messages
    )


@router.get("")
def list_conversations():
    return conversation_service.get_all_conversations()


@router.get("/{conversation_id}")
def read_conversation(conversation_id: str):
    result = conversation_service.get_conversation(
        conversation_id
    )

    if result is None:
        raise HTTPException(
            status_code=404,
            detail="대화를 찾을 수 없습니다."
        )

    return result


@router.delete("/{conversation_id}")
def remove_conversation(conversation_id: str):
    result = conversation_service.delete_conversation(
        conversation_id
    )

    if not result:
        raise HTTPException(
            status_code=404,
            detail="대화를 찾을 수 없습니다."
        )

    return {
        "message": "대화가 삭제되었습니다."
    }
```

## 2-16. `backend/app/routers/chat.py`

AI 질문, 응답, 자동 저장을 처리합니다.

python

```python

from fastapi import APIRouter, HTTPException

from app.models.schemas import ChatRequest
from app.services.analysis_service import get_summary
from app.services.ai_service import generate_answer
from app.services import conversation_service


router = APIRouter(
    prefix="/api/chat",
    tags=["AI Chat"]
)


@router.post("")
def chat(request: ChatRequest):
    conversation = None

    if request.conversation_id:
        conversation = conversation_service.get_conversation(
            request.conversation_id
        )

        if conversation is None:
            raise HTTPException(
                status_code=404,
                detail="대화를 찾을 수 없습니다."
            )

        messages = conversation.get("messages", [])

    else:
        messages = []

    messages = messages[-10:]

    messages.append({
        "role": "user",
        "content": request.message
    })

    try:
        summary = get_summary()

        answer = generate_answer(
            summary,
            messages
        )

        messages.append({
            "role": "assistant",
            "content": answer
        })

        title = (
            conversation.get("title")
            if conversation
            else request.message[:40]
        )

        saved = conversation_service.save_conversation(
            title,
            messages,
            request.conversation_id
        )

        return {
            "answer": answer,
            "conversation_id": saved["id"],
            "summary": summary,
            "messages": messages
        }

    except RuntimeError as exc:
        raise HTTPException(
            status_code=503,
            detail=str(exc)
        )

    except Exception:
        raise HTTPException(
            status_code=502,
            detail="AI 응답 생성 또는 대화 저장에 실패했습니다."
        )
```

## 2-17. `backend/app/main.py`

백엔드의 시작 파일입니다.

python

```python

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.config import ALLOWED_ORIGINS
from app.routers import data, chat, conversations


app = FastAPI(
    title="MyData AI",
    description="개인 시계열 데이터 분석 AI 서비스",
    version="1.0.0"
)


app.add_middleware(
    CORSMiddleware,
    allow_origins=ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Content-Type", "Authorization"]
)


app.include_router(data.router)
app.include_router(chat.router)
app.include_router(conversations.router)


@app.get("/")
def home():
    return {
        "service": "MyData AI",
        "status": "running",
        "docs": "/docs"
    }


@app.get("/health")
def health():
    return {
        "status": "ok"
    }
```

## 2-18. `backend/app/seed.py`

180개 샘플 데이터를 생성합니다.

같은 날짜를 문서 ID로 사용하므로 여러 번 실행해도 날짜별 문서가 중복 생성되지 않습니다.

python

```python

from datetime import date, timedelta
import random

from app.firebase import get_db


def seed_data():
    db = get_db()

    start_date = date(2026, 1, 1)

    batch = db.batch()

    for i in range(180):
        current_date = start_date + timedelta(days=i)

        random.seed(i)

        value = (
            1000000
            + i * 5000
            + random.randint(-150000, 200000)
        )

        document_id = current_date.isoformat()

        ref = db.collection("data").document(document_id)

        batch.set(ref, {
            "date": document_id,
            "value": max(value, 0),
            "memo": "샘플 일별 매출 데이터"
        })

        if (i + 1) % 450 == 0:
            batch.commit()
            batch = db.batch()

    batch.commit()

    print("180개 샘플 데이터 저장 완료")


if __name__ == "__main__":
    seed_data()
```

# 3. 프론트엔드 전체 코드

## 3-1. `frontend/index.html`

HTML줄 바꿈코드 복사미리보기

```html

<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>MyData AI</title>
    <link rel="stylesheet" href="style.css">
</head>

<body>
<div class="app">

    <aside class="sidebar">
        <h2>📊 MyData AI</h2>

        <button id="newChat">+ 새 대화</button>

        <h3>이전 대화</h3>
        <div id="conversationList"></div>
    </aside>

    <main class="main">

        <header>
            <div>
                <h1>나만의 데이터 분석 비서</h1>
                <p>내 데이터를 이해하는 AI와 대화하세요.</p>
            </div>
            <span id="status">연결 확인 중</span>
        </header>

        <section class="stats">
            <div class="stat">
                <span>데이터 개수</span>
                <strong id="count">0</strong>
            </div>

            <div class="stat">
                <span>총 매출</span>
                <strong id="total">0원</strong>
            </div>

            <div class="stat">
                <span>평균 매출</span>
                <strong id="average">0원</strong>
            </div>

            <div class="stat">
                <span>최근 추세</span>
                <strong id="trend">-</strong>
            </div>
        </section>

        <section class="summary panel">
            <h2>📈 데이터 요약</h2>
            <p id="period">기간: -</p>
            <p id="max">최대: -</p>
            <p id="min">최소: -</p>
        </section>

        <section class="panel">
            <h2>데이터 관리</h2>

            <form id="dataForm">
                <input type="hidden" id="editId">

                <input type="date" id="date" required>

                <input
                    type="number"
                    id="value"
                    placeholder="매출 금액"
                    min="0"
                    step="0.01"
                    required
                >

                <input
                    type="text"
                    id="memo"
                    placeholder="메모"
                    maxlength="500"
                >

                <button type="submit" id="saveButton">
                    데이터 추가
                </button>

                <button type="button" id="cancelEdit" hidden>
                    수정 취소
                </button>
            </form>

            <div class="table-wrap">
                <table>
                    <thead>
                        <tr>
                            <th>날짜</th>
                            <th>매출</th>
                            <th>메모</th>
                            <th>관리</th>
                        </tr>
                    </thead>
                    <tbody id="dataList"></tbody>
                </table>
            </div>
        </section>

        <section class="panel">
            <h2>💬 AI 데이터 채팅</h2>

            <p class="notice">
                무료 Render 서버는 첫 접속 시 응답이 늦어질 수 있습니다.
            </p>

            <div id="chatMessages" class="messages">
                <div class="message assistant">
                    안녕하세요! 저장된 매출 데이터를 분석해 드릴게요.
                </div>
            </div>

            <p id="loading" hidden>AI가 데이터를 분석하고 있습니다...</p>

            <form id="chatForm">
                <input
                    id="messageInput"
                    type="text"
                    placeholder="이번 달 매출은 어때?"
                    maxlength="2000"
                    required
                >
                <button type="submit" id="sendButton">전송</button>
            </form>
        </section>

    </main>
</div>

<script src="config.js"></script>
<script src="script.js"></script>
</body>
</html>
```

## 3-2. `frontend/style.css`

css

```css

* {
    box-sizing: border-box;
}

body {
    margin: 0;
    background: #f4f6fb;
    color: #20243a;
    font-family: Arial, "Noto Sans KR", sans-serif;
}

button {
    background: #6258e8;
    color: white;
    border: 0;
    border-radius: 8px;
    padding: 11px 15px;
    cursor: pointer;
}

button:hover {
    opacity: .85;
}

input {
    border: 1px solid #dfe2eb;
    border-radius: 8px;
    padding: 11px;
    min-width: 0;
}

.app {
    display: flex;
    min-height: 100vh;
}

.sidebar {
    width: 240px;
    flex-shrink: 0;
    background: #20243a;
    color: white;
    padding: 24px 16px;
}

.sidebar h2 {
    margin-bottom: 30px;
}

.sidebar h3 {
    margin-top: 35px;
    font-size: 15px;
}

.sidebar > button {
    width: 100%;
}

.conversation-item {
    width: 100%;
    display: block;
    text-align: left;
    background: #343951;
    margin-bottom: 8px;
    overflow-wrap: anywhere;
}

.main {
    flex: 1;
    min-width: 0;
    padding: 28px;
}

header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 15px;
    margin-bottom: 24px;
}

header h1 {
    margin-bottom: 5px;
}

header p {
    color: #82879b;
}

#status {
    font-size: 13px;
    color: #168b61;
}

.panel, .stat {
    background: white;
    border-radius: 13px;
    padding: 20px;
    margin-bottom: 18px;
    box-shadow: 0 3px 12px #20243a08;
}

.stats {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 14px;
    margin-bottom: 18px;
}

.stat span {
    display: block;
    color: #858ba0;
    font-size: 13px;
}

.stat strong {
    display: block;
    margin-top: 12px;
    font-size: 21px;
    overflow-wrap: anywhere;
}

.summary p {
    color: #666;
}

#dataForm {
    display: grid;
    grid-template-columns: repeat(4, minmax(0, 1fr));
    gap: 8px;
    margin: 18px 0;
}

.table-wrap {
    max-height: 400px;
    overflow: auto;
}

table {
    width: 100%;
    border-collapse: collapse;
    text-align: left;
}

th, td {
    padding: 12px 8px;
    border-bottom: 1px solid #eceef4;
    font-size: 13px;
}

th {
    background: white;
    position: sticky;
    top: 0;
}

.actions {
    display: flex;
    gap: 5px;
}

.actions button {
    padding: 7px 9px;
    font-size: 12px;
}

.delete-btn {
    background: #e35465;
}

.messages {
    height: 350px;
    overflow-y: auto;
    background: #f7f8fc;
    padding: 15px;
    border-radius: 10px;
}

.message {
    max-width: 85%;
    padding: 12px;
    margin-bottom: 12px;
    border-radius: 10px;
    white-space: pre-wrap;
    overflow-wrap: anywhere;
    line-height: 1.6;
}

.message.user {
    background: #6258e8;
    color: white;
    margin-left: auto;
}

.message.assistant {
    background: white;
    border: 1px solid #e9eaf0;
}

#chatForm {
    display: flex;
    gap: 10px;
    margin-top: 15px;
}

#messageInput {
    flex: 1;
}

.notice {
    background: #fff7df;
    color: #876519;
    padding: 10px;
    border-radius: 8px;
    font-size: 13px;
}

#loading {
    color: #6258e8;
    font-size: 13px;
}

[hidden] {
    display: none !important;
}

@media(max-width: 900px) {
    .stats {
        grid-template-columns: repeat(2, 1fr);
    }

    #dataForm {
        grid-template-columns: repeat(2, 1fr);
    }
}

@media(max-width: 600px) {
    .app {
        display: block;
    }

    .sidebar {
        width: 100%;
    }

    .main {
        padding: 14px;
    }

    header {
        align-items: flex-start;
        flex-direction: column;
    }

    #dataForm {
        grid-template-columns: 1fr;
    }

    .stats {
        gap: 8px;
    }

    .stat {
        padding: 13px;
    }
}
```

## 3-3. `frontend/script.js`

이 파일은 데이터 관리, 채팅, 대화 불러오기를 모두 연결합니다.

javascript

```javascript

const API = (
    window.API_BASE_URL || "http://127.0.0.1:8000"
).replace(/\/$/, "");

let records = [];
let messages = [];
let currentConversationId = null;

const $ = id => document.getElementById(id);

function money(value) {
    return Number(value || 0).toLocaleString("ko-KR") + "원";
}

async function api(path, options = {}) {
    const response = await fetch(API + path, {
        ...options,
        headers: {
            "Content-Type": "application/json",
            ...options.headers
        }
    });

    const result = await response.json().catch(() => ({}));

    if (!response.ok) {
        throw new Error(
            result.detail || "서버 요청에 실패했습니다."
        );
    }

    return result;
}

function addMessage(role, content) {
    const element = document.createElement("div");

    element.className = "message " + role;
    element.textContent = content;

    $("chatMessages").appendChild(element);

    $("chatMessages").scrollTop =
        $("chatMessages").scrollHeight;
}

function renderMessages() {
    $("chatMessages").replaceChildren();

    if (messages.length === 0) {
        addMessage(
            "assistant",
            "안녕하세요! 어떤 데이터를 분석해 드릴까요?"
        );
        return;
    }

    messages.forEach(message => {
        addMessage(message.role, message.content);
    });
}


// -----------------------
// Data
// -----------------------

async function loadData() {
    records = await api("/api/data");

    $("dataList").replaceChildren();

    records.slice().reverse().forEach(item => {
        const tr = document.createElement("tr");

        const date = document.createElement("td");
        date.textContent = item.date;

        const value = document.createElement("td");
        value.textContent = money(item.value);

        const memo = document.createElement("td");
        memo.textContent = item.memo || "";

        const actions = document.createElement("td");
        actions.className = "actions";

        const edit = document.createElement("button");
        edit.textContent = "수정";
        edit.addEventListener("click", () => editData(item.id));

        const remove = document.createElement("button");
        remove.textContent = "삭제";
        remove.className = "delete-btn";
        remove.addEventListener("click", () => deleteData(item.id));

        actions.append(edit, remove);
        tr.append(date, value, memo, actions);

        $("dataList").appendChild(tr);
    });
}

async function loadSummary() {
    const data = await api("/api/data/summary");

    $("count").textContent =
        data.count.toLocaleString("ko-KR") + "개";

    $("total").textContent = money(data.metrics.total);

    $("average").textContent = money(data.metrics.average);

    $("trend").textContent = data.trend;

    $("period").textContent = "기간: " + data.period;

    $("max").textContent =
        "최대 매출: " + money(data.metrics.max);

    $("min").textContent =
        "최소 매출: " + money(data.metrics.min);
}

async function refresh() {
    try {
        await Promise.all([
            loadData(),
            loadSummary()
        ]);

        $("status").textContent = "● 서버 연결 완료";

    } catch (error) {
        $("status").textContent = "● 연결 실패";
        throw error;
    }
}

function resetDataForm() {
    $("dataForm").reset();
    $("editId").value = "";
    $("saveButton").textContent = "데이터 추가";
    $("cancelEdit").hidden = true;
}

$("dataForm").addEventListener("submit", async event => {
    event.preventDefault();

    const id = $("editId").value;

    const payload = {
        date: $("date").value,
        value: Number($("value").value),
        memo: $("memo").value
    };

    try {
        if (id) {
            await api("/api/data/" + encodeURIComponent(id), {
                method: "PUT",
                body: JSON.stringify(payload)
            });

            alert("데이터가 수정되었습니다.");

        } else {
            await api("/api/data", {
                method: "POST",
                body: JSON.stringify(payload)
            });

            alert("데이터가 추가되었습니다.");
        }

        resetDataForm();
        await refresh();

    } catch (error) {
        alert(error.message);
    }
});

function editData(id) {
    const item = records.find(record => record.id === id);

    if (!item) return;

    $("editId").value = item.id;
    $("date").value = item.date;
    $("value").value = item.value;
    $("memo").value = item.memo || "";

    $("saveButton").textContent = "수정 저장";
    $("cancelEdit").hidden = false;

    window.scrollTo({
        top: 0,
        behavior: "smooth"
    });
}

async function deleteData(id) {
    if (!confirm("정말 삭제하시겠습니까?")) return;

    try {
        await api("/api/data/" + encodeURIComponent(id), {
            method: "DELETE"
        });

        resetDataForm();
        await refresh();

        alert("삭제되었습니다.");

    } catch (error) {
        alert(error.message);
    }
}

$("cancelEdit").addEventListener("click", resetDataForm);


// -----------------------
// Chat
// -----------------------

$("chatForm").addEventListener("submit", async event => {
    event.preventDefault();

    const input = $("messageInput");
    const message = input.value.trim();

    if (!message) return;

    input.disabled = true;
    $("sendButton").disabled = true;

    $("loading").hidden = false;

    try {
        const result = await api("/api/chat", {
            method: "POST",
            body: JSON.stringify({
                message,
                conversation_id: currentConversationId
            })
        });

        currentConversationId = result.conversation_id;

        messages = result.messages;

        renderMessages();

        await loadConversations();

    } catch (error) {
        alert(error.message);

    } finally {
        input.value = "";
        input.disabled = false;
        $("sendButton").disabled = false;
        $("loading").hidden = true;
        input.focus();
    }
});


// -----------------------
// Conversations
// -----------------------

async function loadConversations() {
    const conversations = await api("/api/conversations");

    $("conversationList").replaceChildren();

    conversations.forEach(item => {
        const button = document.createElement("button");

        button.className = "conversation-item";
        button.textContent = item.title;

        button.addEventListener("click", () => {
            openConversation(item.id);
        });

        $("conversationList").appendChild(button);
    });
}

async function openConversation(id) {
    try {
        const result = await api(
            "/api/conversations/" + encodeURIComponent(id)
        );

        currentConversationId = result.id;
        messages = result.messages || [];

        renderMessages();

    } catch (error) {
        alert(error.message);
    }
}

$("newChat").addEventListener("click", () => {
    currentConversationId = null;
    messages = [];
    renderMessages();
});


// -----------------------
// Start
// -----------------------

async function initialize() {
    $("date").value = new Date().toLocaleDateString("sv-SE");

    try {
        await refresh();
        await loadConversations();

    } catch (error) {
        console.error(error);
        $("status").textContent = "● 서버 연결 실패";
    }
}

initialize();
```

## 3-4. `frontend/build.js`

Vercel 환경변수를 JavaScript 설정 파일로 만들어 줍니다.

javascript

```javascript

const fs = require("fs");

const apiUrl = process.env.API_BASE_URL;

if (!apiUrl) {
    throw new Error(
        "Vercel 환경변수 API_BASE_URL이 설정되지 않았습니다."
    );
}

if (!/^https?:\/\//.test(apiUrl)) {
    throw new Error("API_BASE_URL 주소가 올바르지 않습니다.");
}

fs.writeFileSync(
    "config.js",
    `window.API_BASE_URL = ${JSON.stringify(apiUrl)};`
);

console.log("API 설정 파일 생성 완료");
```

## 3-5. `frontend/package.json`

json

```json

{
  "name": "mydata-ai-frontend",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "build": "node build.js"
  }
}
```

## 3-6. `frontend/vercel.json`

json

```json

{
  "cleanUrls": true,
  "trailingSlash": false
}
```

# 4. GitHub 보안 설정

## `.gitignore`

반드시 프로젝트 최상위 폴더에 생성합니다.

text

```text

# Python
__pycache__/
*.py[cod]
venv/
.venv/

# Secrets
.env
.env.*
!.env.example
*.json.key
serviceAccountKey.json

# Frontend generated config
frontend/config.js

# Node
node_modules/

# Editor
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db
```

이 설정으로 다음 파일들은 GitHub에 올라가지 않습니다.

- `backend/.env`
- Firebase 서비스 계정 JSON 파일
- OpenAI API 키
- 생성된 프론트엔드 `config.js`

주의: `.gitignore`는 이미 GitHub에 올라간 비밀키를 자동으로 삭제하지 않습니다. 실수로 키를 올렸다면 해당 키를 폐기하고 새로 발급해야 합니다.

# 5. Firebase 처음부터 설정하기

Firebase 프로젝트가 아직 없으므로 아래 과정을 진행해야 합니다.

Learn the Basics and Get Started with Firebase — an App Development Platform Backed by Google | by Lynn G. Kwong | CodeX | Medium

PlatformIO Registry

Configuring your app | About the service

1. [Firebase Console](<https://console.firebase.google.com/?utm_source=chatgpt.com>)에 접속합니다.
2. Google 계정으로 로그인합니다.
3. 프로젝트 추가를 누릅니다.
4. 프로젝트 이름을 `mydata-ai`로 입력합니다.
5. Google Analytics는 이번 과제에서 사용하지 않으므로 비활성화해도 됩니다.
6. 프로젝트 생성 후 Firestore Database 메뉴로 이동합니다.
7. 데이터베이스 만들기를 누릅니다.
8. 테스트 데이터베이스가 아닌 운영 모드를 선택합니다.
9. 데이터베이스 위치를 선택하고 생성합니다.

### Firebase 서비스 계정 키 만들기

1. 프로젝트 설정으로 들어갑니다.
2. 서비스 계정 탭을 선택합니다.
3. Python을 선택합니다.
4. 새 비공개 키 생성 버튼을 누릅니다.
5. JSON 파일을 다운로드합니다.

비밀키 주의다운로드한 JSON 파일은 GitHub에 절대 업로드하지 마세요.

Windows에서는 파일을 다음과 같이 보관할 수 있습니다.

text

```text
C:\mydata-ai-secrets\firebase-key.json
```

이 파일의 경로를 로컬 `.env`에 설정합니다.

## 6. Windows에서 실제 실행하기

### 6-1. Python 설치

Python 3.10 이상을 설치합니다.

[Python 다운로드](<https://www.python.org/downloads/?utm_source=chatgpt.com>)

설치 시 `Add Python to PATH`를 체크하세요.

### 6-2. GitHub 프로젝트 다운로드

[GitHub](<https://github.com/?utm_source=chatgpt.com>)에서 저장소를 만든 후 PC에 다운로드합니다.

프로젝트 폴더에서 터미널을 열고 실행합니다.

powershell

```powershell
cd backend

py -3 -m venv venv

.\venv\Scripts\Activate.ps1
```

PowerShell에서 실행 정책 오류가 발생하면 현재 사용자에 대해 다음 명령을 실행할 수 있습니다.

powershell

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

이후 패키지를 설치합니다.

powershell

```powershell
python -m pip install --upgrade pip

pip install -r requirements.txt
```

### 6-3. 실제 환경변수 설정

`backend` 폴더에 `.env` 파일을 새로 만드세요.

dotenv

```dotenv
OPENAI_API_KEY=여기에_본인의_OpenAI_API_키
OPENAI_MODEL=gpt-4o-mini

FIREBASE_SERVICE_ACCOUNT_PATH=C:/mydata-ai-secrets/firebase-key.json

ALLOWED_ORIGINS=http://localhost:5500,http://127.0.0.1:5500
```

OpenAI API 키는 다음에서 발급받습니다.

[OpenAI API Keys](<https://platform.openai.com/api-keys?utm_source=chatgpt.com>)

API 사용에는 별도 과금이 발생할 수 있으므로 사용량과 결제 한도를 확인하세요.

### 6-4. 샘플 데이터 생성

프로젝트 최상위 폴더가 아닌 `backend` 폴더에서 실행합니다.

powershell

```powershell
python -m app.seed
```

앞서 작성한 `seed.py`의 날짜 부분은 다음처럼 수정하면 현재 기준 최근 180일 데이터가 만들어집니다.

python

```python
start_date = date.today() - timedelta(days=179)
```

이렇게 하면 최근 180일의 매출 데이터가 Firestore에 저장됩니다.

### 6-5. 백엔드 실행

powershell

```powershell
uvicorn app.main:app --reload
```

실행 확인 주소

- API: [http://127.0.0.1:8000](<http://127.0.0.1:8000?utm_source=chatgpt.com>)
- Swagger: [http://127.0.0.1:8000/docs](<http://127.0.0.1:8000/docs?utm_source=chatgpt.com>)
- Health: [http://127.0.0.1:8000/health](<http://127.0.0.1:8000/health?utm_source=chatgpt.com>)
  서버 실행 확인

### 6-6. 프론트엔드 실행

VS Code에서 `frontend/index.html`을 열고 Live Server로 실행합니다.

[VS Code Live Server](<https://marketplace.visualstudio.com/items?itemName=ritwickdey.LiveServer&utm_source=chatgpt.com>)

로컬에서는 `config.js`가 없어도 JavaScript의 기본 주소인 `http://127.0.0.1:8000`을 사용합니다.

# 7. Render 배포

[Render](<https://render.com/?utm_source=chatgpt.com>)에 GitHub 계정으로 로그인합니다.

1. New Web Service 선택
2. GitHub 저장소 연결
3. 다음 설정 적용

| 설정 | 값 |
| --- | --- |
| Root Directory | `backend` |
| Runtime | Python 3 |
| Build Command | `pip install -r requirements.txt` |
| Start Command | `uvicorn app.main:app --host 0.0.0.0 --port $PORT` |

환경변수는 Render의 Environment 메뉴에서 등록합니다.

| Key | Value |
| --- | --- |
| `OPENAI_API_KEY` | 본인 OpenAI 키 |
| `OPENAI_MODEL` | `gpt-4o-mini` |
| `FIREBASE_SERVICE_ACCOUNT_JSON` | Firebase JSON 전체 내용 |
| `ALLOWED_ORIGINS` | Vercel 주소 |

Firebase JSON은 파일 경로가 아니라 JSON 전체 내용을 환경변수로 넣습니다. JSON은 유효한 형식이어야 합니다.

예를 들어 실제 JSON 파일의 내용을 그대로 복사해 넣습니다. 키 값은 누구와도 공유하지 마세요.

배포 후 다음 주소가 열리는지 확인합니다.

text

```text
https://본인서비스.onrender.com
https://본인서비스.onrender.com/docs
```

# 8. Vercel 배포

[Vercel](<https://vercel.com/?utm_source=chatgpt.com>)에 로그인합니다.

1. Add New Project 선택
2. GitHub 저장소 연결
3. Root Directory를 `frontend`로 설정
4. Build Command: `npm run build`
5. Output Directory: `.`
6. 환경변수 추가

env

```env
API_BASE_URL=https://본인서비스.onrender.com
```

Deploy를 누릅니다.

배포가 끝나면 Vercel 도메인을 복사합니다.

Render의 `ALLOWED_ORIGINS`에 해당 도메인을 넣고 백엔드를 재배포합니다.

# 9. 최종 README.md

프로젝트 최상위 폴더에 `README.md`를 만들고 아래 내용을 붙여넣으세요.

markdown

````markdown

# MyData AI 📊

나만의 시계열 데이터를 분석하고 AI와 대화하는 개인 데이터 분석 비서입니다.

## 1. 프로젝트 소개

일반적인 AI 챗봇은 사용자의 개인 데이터를 알지 못합니다.

MyData AI는 사용자의 매출 데이터를 Firestore에 저장하고,
데이터의 평균, 최대, 최소, 총합 및 최근 추세를 분석합니다.

분석 결과를 GPT 시스템 프롬프트에 주입하여
사용자의 실제 데이터를 기반으로 맞춤형 답변을 제공합니다.

## 2. 주요 기능

- 시계열 데이터 추가, 조회, 수정, 삭제
- 데이터 기간 및 통계 분석
- 최근 매출 추세 분석
- GPT 기반 AI 데이터 채팅
- 대화 자동 저장
- 이전 대화 목록 조회
- 특정 대화 불러오기 및 삭제
- Swagger API 문서

## 3. 기술 스택

### Backend
- Python
- FastAPI
- Pydantic
- Firebase Admin SDK
- Cloud Firestore
- OpenAI API

### Frontend
- HTML
- CSS
- Vanilla JavaScript

### Deployment
- GitHub
- Render
- Vercel

## 4. 폴더 구조

```text
mydata-ai/
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── firebase.py
│   │   ├── models/
│   │   ├── routers/
│   │   └── services/
│   ├── requirements.txt
│   └── .env.example
├── frontend/
│   ├── index.html
│   ├── style.css
│   ├── script.js
│   ├── build.js
│   ├── package.json
│   └── vercel.json
├── .gitignore
└── README.md
```

## 5. Firestore 구조

### data

- date: 데이터 날짜
- value: 매출 금액
- memo: 메모
- created_at: 생성 시간

### conversations

- title: 대화 제목
- messages: 대화 메시지
- created_at: 생성 시간
- updated_at: 수정 시간

## 6. API 목록

| Method | Endpoint | 기능 |
|---|---|---|
| POST | /api/data | 데이터 추가 |
| GET | /api/data | 데이터 조회 |
| PUT | /api/data/{data_id} | 데이터 수정 |
| DELETE | /api/data/{data_id} | 데이터 삭제 |
| GET | /api/data/summary | 데이터 요약 |
| POST | /api/chat | AI 채팅 |
| POST | /api/conversations | 대화 저장 |
| GET | /api/conversations | 대화 목록 |
| GET | /api/conversations/{id} | 대화 불러오기 |
| DELETE | /api/conversations/{id} | 대화 삭제 |

## 7. 로컬 실행

### Python 가상환경

```powershell
cd backend

py -3 -m venv venv

.\venv\Scripts\Activate.ps1

pip install -r requirements.txt
```

### 환경변수

backend 폴더에 .env 파일을 생성합니다.

```env
OPENAI_API_KEY=
OPENAI_MODEL=gpt-4o-mini
FIREBASE_SERVICE_ACCOUNT_PATH=C:/mydata-ai-secrets/firebase-key.json
ALLOWED_ORIGINS=http://localhost:5500,http://127.0.0.1:5500
```

### 샘플 데이터 등록

```powershell
python -m app.seed
```

### 서버 실행

```powershell
uvicorn app.main:app --reload
```

Swagger UI:

http://127.0.0.1:8000/docs

### 프론트엔드 실행

VS Code Live Server로 frontend/index.html을 실행합니다.

## 8. 배포 주소

- Frontend: 배포 후 입력
- Backend: 배포 후 입력
- Swagger UI: 배포 후 입력

## 9. AI 컨텍스트 주입 원리

1. 사용자가 자연어 질문을 입력합니다.
2. 백엔드가 Firestore 데이터를 조회합니다.
3. 데이터 분석 서비스를 통해 요약을 계산합니다.
4. 분석 결과를 GPT 시스템 프롬프트에 삽입합니다.
5. GPT API가 데이터를 참고하여 답변합니다.
6. 질문과 답변을 Firestore에 자동 저장합니다.

## 10. 보안

- API 키는 환경변수로 관리합니다.
- Firebase 서비스 계정 키는 저장소에 업로드하지 않습니다.
- CORS 허용 도메인을 제한합니다.
- Pydantic으로 사용자 입력을 검증합니다.
- AI 요청 토큰 수를 제한합니다.

## 11. 제출 스크린샷

- AI 채팅 및 데이터 요약 화면
- 데이터 CRUD 화면
- 대화 기록 및 불러오기 화면
- Swagger UI 화면
````

# 10. 최종 테스트 순서

아래 순서대로 확인하세요.

| 순서 | 테스트 | 정상 결과 |
| --- | --- | --- |
| 1 | `/health` | status: ok |
| 2 | `/docs` | Swagger 표시 |
| 3 | `POST /api/data` | 데이터 저장 |
| 4 | `GET /api/data` | 데이터 목록 반환 |
| 5 | `GET /api/data/summary` | 통계 반환 |
| 6 | `PUT /api/data/{id}` | 수정 완료 |
| 7 | `DELETE /api/data/{id}` | 삭제 완료 |
| 8 | `POST /api/chat` | AI 답변 및 자동 저장 |
| 9 | `GET /api/conversations` | 대화 목록 반환 |
| 10 | `GET /api/conversations/{id}` | 메시지 불러오기 |
이제 GitHub에 올릴 파일은 모두 준비됐습니다. 다만 아직 외부 서비스에 연결해 배포 성공까지 검증한 것은 아닙니다.

가장 좋은 진행 순서는 먼저 Windows에서 로컬 실행을 성공시키고, Swagger에서 CRUD와 AI 채팅을 테스트한 다음 배포하는 것입니다.
