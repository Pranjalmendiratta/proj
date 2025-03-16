from fastapi import FastAPI, HTTPException, Depends
from pymongo import MongoClient
from pydantic import BaseModel
from passlib.context import CryptContext
from datetime import datetime, timedelta
from jose import JWTError, jwt
from typing import List, Optional

# FastAPI instance
app = FastAPI()

# Database Connection
client = MongoClient("mongodb://localhost:27017")
db = client["recipe_nest"]
users_collection = db["users"]
recipes_collection = db["recipes"]

# Password Hashing
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# JWT Config
SECRET_KEY = "secret_key"  
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

# User Model
class User(BaseModel):
    username: str
    password: str

# Recipe Model
class Recipe(BaseModel):
    title: str
    ingredients: List[str]
    instructions: str
    user_id: Optional[str] = None

# Token Model
class TokenData(BaseModel):
    username: Optional[str] = None

# Helper functions
def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

def create_access_token(data: dict, expires_delta: timedelta):
    to_encode = data.copy()
    expire = datetime.utcnow() + expires_delta
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

# API Endpoints

@app.post("/register/")
def register(user: User):
    if users_collection.find_one({"username": user.username}):
        raise HTTPException(status_code=400, detail="Username already exists")
    
    hashed_password = hash_password(user.password)
    users_collection.insert_one({"username": user.username, "password": hashed_password})
    return {"message": "User registered successfully"}

@app.post("/login/")
def login(user: User):
    db_user = users_collection.find_one({"username": user.username})
    if not db_user or not verify_password(user.password, db_user["password"]):
        raise HTTPException(status_code=400, detail="Invalid credentials")
    
    access_token = create_access_token({"sub": user.username}, timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES))
    return {"access_token": access_token, "token_type": "bearer"}

@app.post("/recipes/")
def add_recipe(recipe: Recipe):
    recipe_data = recipe.dict()
    recipes_collection.insert_one(recipe_data)
    return {"message": "Recipe added successfully"}

@app.get("/recipes/", response_model=List[Recipe])
def get_recipes():
    return list(recipes_collection.find({}, {"_id": 0}))

# Placeholder AI Recommendation Route
@app.get("/recommendations/")
def get_recommendations():
    return {"message": "AI recommendations coming soon"}
