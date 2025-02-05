# invoice-backend
git clone https://github.com/Dom9711/invoice-backend.git
from fastapi import FastAPI, Depends, HTTPException, File, UploadFile
from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session
from passlib.context import CryptContext
import jwt
import datetime

# Настройки
DATABASE_URL = "postgresql://user:password@localhost/invoice_db"
SECRET_KEY = "your_secret_key"
ALGORITHM = "HS256"

# Подключение к БД
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(bind=engine, autocommit=False, autoflush=False)
Base = declarative_base()

# Хеширование паролей
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Модель пользователя
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, unique=True, index=True)
    password_hash = Column(String)
    first_name = Column(String)
    last_name = Column(String)
    street = Column(String)
    phone = Column(String)
    email = Column(String, unique=True)
    bank_account = Column(String)

# Модель информации о транзакции
class Invoice(Base):
    __tablename__ = "invoices"
    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer)
    date = Column(String)
    place = Column(String)
    hours = Column(Integer)
    price = Column(Integer)

Base.metadata.create_all(bind=engine)

app = FastAPI()

# Регистрация пользователя
@app.post("/register")
def register(username: str, password: str, first_name: str, last_name: str, street: str, phone: str, email: str, bank_account: str, db: Session = Depends(get_db)):
    hashed_password = pwd_context.hash(password)
    user = User(username=username, password_hash=hashed_password, first_name=first_name, last_name=last_name, street=street, phone=phone, email=email, bank_account=bank_account)
    db.add(user)
    db.commit()
    return {"message": "User registered successfully"}

# Авторизация
@app.post("/login")
def login(username: str, password: str, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.username == username).first()
    if not user or not pwd_context.verify(password, user.password_hash):
        raise HTTPException(status_code=400, detail="Invalid credentials")
    token = jwt.encode({"sub": username, "exp": datetime.datetime.utcnow() + datetime.timedelta(hours=1)}, SECRET_KEY, algorithm=ALGORITHM)
    return {"access_token": token}

# Отправка информации о транзакции
@app.post("/submit_invoice")
def submit_invoice(user_id: int, date: str, place: str, hours: int, price: int, db: Session = Depends(get_db)):
    invoice = Invoice(user_id=user_id, date=date, place=place, hours=hours, price=price)
    db.add(invoice)
    db.commit()
    return {"message": "Invoice submitted successfully"}

# Загрузка файла
@app.post("/upload")
def upload_file(file: UploadFile = File(...)):
    return {"filename": file.filename}
