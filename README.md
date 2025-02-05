# invoice-backend
git clone https://github.com/Dom9711/invoice-backend.git
from fastapi import FastAPI, Depends, HTTPException, File, UploadFile
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel
from typing import Dict
import shutil
import os
import psycopg2
import jwt
import datetime

app = FastAPI()
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")
SECRET_KEY = "mysecretkey"
UPLOAD_FOLDER = "uploads"
os.makedirs(UPLOAD_FOLDER, exist_ok=True)

# Подключение к базе данных PostgreSQL
DATABASE_URL = "postgresql://user:password@localhost:5432/invoice_db"
conn = psycopg2.connect(DATABASE_URL)
cursor = conn.cursor()
cursor.execute("""
    CREATE TABLE IF NOT EXISTS users (
        email TEXT PRIMARY KEY,
        first_name TEXT,
        last_name TEXT,
        phone TEXT,
        password TEXT
    )
""")
cursor.execute("""
    CREATE TABLE IF NOT EXISTS invoices (
        id SERIAL PRIMARY KEY,
        user_email TEXT REFERENCES users(email),
        date TEXT,
        place TEXT,
        hours INTEGER,
        price REAL
    )
""")
conn.commit()

class User(BaseModel):
    first_name: str
    last_name: str
    email: str
    phone: str
    password: str

class Invoice(BaseModel):
    date: str
    place: str
    hours: int
    price: float

def create_token(data: dict):
    to_encode = data.copy()
    expire = datetime.datetime.utcnow() + datetime.timedelta(hours=1)
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm="HS256")

def decode_token(token: str):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        return payload["sub"]
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.post("/register")
def register_user(user: User):
    cursor.execute("INSERT INTO users VALUES (%s, %s, %s, %s, %s)", (user.email, user.first_name, user.last_name, user.phone, user.password))
    conn.commit()
    return {"message": "User registered"}

@app.post("/token")
def login(form_data: OAuth2PasswordRequestForm = Depends()):
    cursor.execute("SELECT * FROM users WHERE email = %s AND password = %s", (form_data.username, form_data.password))
    user = cursor.fetchone()
    if not user:
        raise HTTPException(status_code=400, detail="Invalid credentials")
    token = create_token({"sub": form_data.username})
    return {"access_token": token, "token_type": "bearer"}

@app.get("/user_info")
def get_user_info(token: str = Depends(oauth2_scheme)):
    email = decode_token(token)
    cursor.execute("SELECT first_name, last_name, email, phone FROM users WHERE email = %s", (email,))
    user = cursor.fetchone()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return {"first_name": user[0], "last_name": user[1], "email": user[2], "phone": user[3]}

@app.put("/update_user")
def update_user(user: User, token: str = Depends(oauth2_scheme)):
    email = decode_token(token)
    cursor.execute("UPDATE users SET first_name = %s, last_name = %s, phone = %s WHERE email = %s", (user.first_name, user.last_name, user.phone, email))
    conn.commit()
    return {"message": "User updated"}

@app.post("/submit_invoice")
def submit_invoice(invoice: Invoice, token: str = Depends(oauth2_scheme)):
    email = decode_token(token)
    cursor.execute("INSERT INTO invoices (user_email, date, place, hours, price) VALUES (%s, %s, %s, %s, %s)", (email, invoice.date, invoice.place, invoice.hours, invoice.price))
    conn.commit()
    return {"message": "Invoice submitted"}

@app.post("/upload")
def upload_file(file: UploadFile = File(...), token: str = Depends(oauth2_scheme)):
    file_path = os.path.join(UPLOAD_FOLDER, file.filename)
    with open(file_path, "wb") as buffer:
        shutil.copyfileobj(file.file, buffer)
    return {"filename": file.filename}
