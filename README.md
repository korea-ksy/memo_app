# memo_app
- 인증(로그인) → 해싱 기법으로 비밀번호 저장
- 사용자(user)에 따라 메모 테이블 저장
- CRUD 적용

# 🐇 코드 Review (API)

### 1. 라이브러리 임포트

```python
from fastapi import FastAPI, Request, Depends, HTTPException
from fastapi.templating import Jinja2Templates
from sqlalchemy.orm import Session
from sqlalchemy import create_engine, MetaData, Table, Column, Integer, String, ForeignKey
from sqlalchemy.ext.declarative import declarative_base
from pydantic import BaseModel
from typing import Optional
from passlib.context import CryptContext
from starlette.middleware.sessions import SessionMiddleware
```

**코드 설명**

- **Request**: HTTP 요청을 나타내는 객체로, 요청의 메타데이터와 본문에 접근 가능
- **Depends**: 의존성 주입을 위한 기능으로, 특정 함수나 클래스의 인스턴스를 자동으로 주입 가능
- **HTTPException**: HTTP 오류를 발생시키기 위한 예외 클래스
- **Jinja2Templates**: Jinja2 템플릿 엔진을 사용하여 HTML 파일을 렌더링하는 데 사용
- **Session**: SQLAlchemy의 세션 객체로, 데이터베이스와의 상호작용을 관리
- **create_engine**: SQLAlchemy의 데이터베이스 엔진을 생성하는 함수
- **Column, Integer, String, ForeignKey**: SQLAlchemy에서 데이터베이스 테이블의 열을 정의하는 데 사용되는 클래스
- **declarative_base**: SQLAlchemy ORM에서 사용할 기본 클래스를 생성
- **BaseModel**: Pydantic의 기본 모델 클래스로, 데이터 검증 및 직렬화를 지원
- **Optional**: 타입 힌팅에서 선택적 필드를 정의할 때 사용
- **CryptContext**: 비밀번호 해싱 및 검증을 위한 설정을 관리
- **SessionMiddleware**: 세션 관리를 위한 미들웨어로, 사용자 세션을 유지하는 데 사용

### 2. 비밀번호 해싱 설정

```python
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def get_password_hash(password):
    return pwd_context.hash(password)

def verify_password(plain_password, hashed_password):
    return pwd_context.verify(plain_password, hashed_password)
```

- **`CryptContext`: 비밀번호 해싱을 위한 다양한 알고리즘을 설정 
여기서는 bcrypt를 사용함**
- **`get_password_hash`**: 
사용자가 입력한 평문 비밀번호를 해시하여 안전하게 저장할 수 있는 형태로 변환
해시된 비밀번호는 데이터베이스에 저장
- **`verify_password**:` 사용자가 로그인할 때 입력한 평문 비밀번호와 데이터베이스에 저장된 해시 비밀번호를 비교하여 일치 여부를 확인

### 3. FastAPI 애플리케이션 생성

```python
app = FastAPI()
app.add_middleware(SessionMiddleware, secret_key="your-secret-key")
templates = Jinja2Templates(directory="templates")
```

- **`SessionMiddleware`**: 세션 관리를 위한 미들웨어를 추가
 secret_key는 세션 데이터의 암호화를 위한 비밀 키입니다.
- **`Jinja2Templates`**: HTML 템플릿을 렌더링하기 위한 설정
 directory는 템플릿 파일이 위치한 디렉토리

### 4. 데이터베이스 설정

```python
DATABASE_URL = "mysql+pymysql://yun:0000@localhost/my_memo_app"
engine = create_engine(DATABASE_URL)
Base = declarative_base()
```

- **`DATABASE_URL`: 데이터베이스에 연결하기 위한 URL
 여기서는 MySQL 데이터베이스를 사용하고 있음. 
yun은 사용자 이름, 0000은 비밀번호, localhost는 데이터베이스 서버의 주소, my_memo_app은 데이터베이스 이름**
- **`create_engine`**: 데이터베이스와의 연결을 관리하는 엔진을 생성
 이 엔진을 통해 SQLAlchemy가 데이터베이스와 상호작용함
→ sqlalchemy는 파이썬에서 sql을 사용하게 할 수 있는 ORM 라이브러리임
- **`declarative_base`**: SQLAlchemy ORM에서 사용할 기본 클래스를 생성
이 클래스를 상속받아 데이터베이스 모델을 정의

### 5. 데이터베이스 모델 정의

```python
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True, index=True)
    username = Column(String(100), unique=True, index=True)
    email = Column(String(200))
    hashed_password = Column(String(512))
```

- **User**: 사용자 정보를 저장하기 위한 데이터베이스 모델
→ `Base`를 상속받아 SQLAlchemy ORM의 기능을 사용할 수 있음
- **tablename**: 데이터베이스에서 사용할 테이블 이름을 정의
- **Column**: 테이블의 각 열을 정의

### 6. 데이터 검증 모델 정의

```python
class UserCreate(BaseModel):
    username: str
    email: str
    password: str

class UserLogin(BaseModel):
    username: str
    password: str
```

- **`UserCreate`**: 회원가입 시 입력받는 데이터 모델
→ Pydantic의 BaseModel을 상속받아 데이터 검증을 수행
→ 사용자가 입력한 데이터가 올바른 형식인지 확인
- **`UserLogin`**: 로그인 시 입력받는 데이터 모델 
마찬가지로 Pydantic의 BaseModel을 상속받음

### 7. 데이터베이스 세션 관리

```python
def get_db():
    db = Session(bind=engine)
    try:
        yield db
    finally:
        db.close()
```

- **`get_db`**: 데이터베이스 세션을 생성하고, 요청이 끝난 후 세션을 닫음
    
     이 함수는 FastAPI의 의존성 주입 시스템을 통해 사용
    
- **`Session(bind=engine)`**: 데이터베이스와 연결된 세션을 생성
- **`yield db`**: 생성된 세션을 반환 
이 시점에서 클라이언트의 요청을 처리 가능
- **`finally`**: 요청이 끝난 후 세션을 닫음
이는 데이터베이스 연결을 안전하게 종료하는 데 중요함

### 8. 테이블 생성

```python
Base.metadata.create_all(bind=engine)
```

**`create_all`** : 정의된 모든 모델에 대해 데이터베이스에 테이블을 생성
User모델에 해당하는 users 테이블이 데이터베이스에 생성됨
만약 테이블이 이미 존재한다면, 아무런 변화 X

### 9. API 엔드포인트 정의

- 회원가입: /signup 엔드포인트에서 사용자 정보를 받아 회원가입을 처리함

```python
  @app.post("/signup")
  async def signup(signup_data: UserCreate, db: Session = Depends(get_db)):
      existing_user = db.query(User).filter(User.username == signup_data.username).first()
      if existing_user:
          raise HTTPException(status_code=400, detail="이미 동일 사용자 이름이 가입되어 있습니다.")
      hashed_password = get_password_hash(signup_data.password)
      new_user = User(username=signup_data.username, email=signup_data.email, hashed_password=hashed_password)
      db.add(new_user)
      
      try:
          db.commit()
      except Exception as e:
          print(e)
          raise HTTPException(status_code=500, detail="회원가입이 실패했습니다. 기입한 내용을 확인해보세요.")
      
      db.refresh(new_user)
      return {"message": "회원가입이 성공했습니다."}
```

- 사용자가 입력한 사용자 이름이 이미 존재하는지 확인합니다. 존재한다면 400 오류를 발생시킵니다.
- 비밀번호를 해시하여 새로운 사용자 객체를 생성하고, 데이터베이스에 추가합니다.
- 데이터베이스에 커밋하여 변경 사항을 저장합니다. 오류가 발생하면 500 오류를 발생시킵니다.

- **로그인**: /login 엔드포인트에서 사용자 인증을 처리

```python
  @app.post("/login")
  async def login(request: Request, signin_data: UserLogin, db: Session = Depends(get_db)):
      user = db.query(User).filter(User.username == signin_data.username).first()
      if user and verify_password(signin_data.password, user.hashed_password):
          request.session["username"] = user.username
          return {"message": "로그인이 성공했습니다."}
      else:
          raise HTTPException(status_code=401, detail="로그인이 실패했습니다.")
```

- 사용자가 입력한 사용자 이름으로 데이터베이스에서 사용자를 조회합니다.
- 비밀번호가 일치하면 세션에 사용자 이름을 저장하고 성공 메시지를 반환합니다. 일치하지 않으면 401 오류를 발생시킵니다.

- **로그아웃**: /logout 엔드포인트에서 세션을 종료

```python
  @app.post("/logout")
  async def logout(request: Request):
      request.session.pop("username", None)
      return {"message": "로그아웃이 성공했습니다."}
```

- 메모 생성: /memos/ 엔드포인트에서 메모 생성

```python
 @app.post("/memos/")
  async def create_memo(request: Request, memo: MemoCreate, db: Session = Depends(get_db)):
      username = request.session.get("username")
      if username is None:
          raise HTTPException(status_code=401, detail="Not authorized")
      user = db.query(User).filter(User.username == username).first()
      if user is None:
          raise HTTPException(status_code=404, detail="User not found")
      new_memo = Memo(user_id=user.id, title=memo.title, content=memo.content)
      db.add(new_memo)
      db.commit()
      db.refresh(new_memo)
      return new_memo
```

- 세션에서 사용자 이름을 가져와 사용자가 로그인했는지 확인 
로그인하지 않았다면 401 오류를 발생
- 사용자가 존재하는지 확인한 후, 새로운 메모를 생성하고 데이터베이스에 추가
- **메모 조회**: /memos/ 엔드포인트에서 사용자의 메모를 조회

```python
@app.get("/memos/")
  async def list_memos(request: Request, db: Session = Depends(get_db)):
      username = request.session.get("username")
      if username is None:
          raise HTTPException(status_code=401, detail="Not authorized")
      user = db.query(User).filter(User.username == username).first()
      if user is None:
          raise HTTPException(status_code=404, detail="User not found")    
      
      memos = db.query(Memo).filter(Memo.user_id == user.id).all()
      return templates.TemplateResponse("memos.html", {"request": request, "memos": memos, "username": username})
```

- 로그인된 사용자의 메모를 조회하여 HTML 템플릿으로 렌더링함

- **메모 수정**: /memos/{memo_id} 엔드포인트에서 특정 메모를 수정

```python
  @app.put("/memos/{memo_id}")
  async def update_memo(request: Request, memo_id: int, memo: MemoUpdate, db: Session = Depends(get_db)):
      username = request.session.get("username")
      if username is None:
          raise HTTPException(status_code=401, detail="Not authorized")
      user = db.query(User).filter(User.username == username).first()
      if user is None:
          raise HTTPException(status_code=404, detail="User not found")     
      db_memo = db.query(Memo).filter(Memo.user_id == user.id, Memo.id == memo_id).first()
      if db_memo is None:
          return ({"error": "Memo not found"})

      if memo.title is not None:
          db_memo.title = memo.title
      if memo.content is not None:
          db_memo.content = memo.content
          
      db.commit()
      db.refresh(db_memo)
      return db_memo
```

 사용자가 로그인했는지 확인한 후, 수정할 메모를 데이터베이스에서 조회
 메모가 존재하면 제목과 내용을 수정하고, 변경 사항을 커밋

- 메모 삭제: /memos/{memo_id} 엔드포인트에서 특정 메모를 삭제

```python
  @app.delete("/memos/{memo_id}")
  async def delete_memo(request: Request, memo_id: int, db: Session = Depends(get_db)):
      username = request.session.get("username")
      if username is None:
          raise HTTPException(status_code=401, detail="Not authorized")
      user = db.query(User).filter(User.username == username).first()
      if user is None:
          raise HTTPException(status_code=404, detail="User not found")     
      db_memo = db.query(Memo).filter(Memo.user_id == user.id, Memo.id == memo_id).first()
      if db_memo is None:
          return ({"error": "Memo not found"})
          
      db.delete(db_memo)
      db.commit()
      return ({"message": "Memo deleted"})
```

사용자가 로그인했는지 확인한 후, 삭제할 메모를 데이터베이스에서 조회
메모가 존재하면 삭제하고, 변경 사항을 커밋

### 루트 및 소개 페이지

```python
@app.get('/')
async def read_root(request: Request):
    return templates.TemplateResponse('home.html', {"request": request})

@app.get("/about")
async def about():
    return {"message": "이것은 마이 메모 앱의 소개 페이지입니다."}
```

- **루트 엔드포인트**: 기본 페이지를 렌더링함
 home.html 템플릿을 사용하여 사용자에게 보여줄 내용을 생성
 `{"request": request}`는 템플릿에서 요청 정보를 사용할 수 있도록 전달
- **소개 페이지**: 애플리케이션에 대한 간단한 소개 메시지를 반환
 JSON 형식으로 메시지를 반환
