Step 1: Set Up the Environment
Install the required packages:
FastAPI: To create the API.
Uvicorn: ASGI server to run the FastAPI app.
SQLAlchemy: ORM to interact with the SQLite database.
Pydantic: For request and response validation (used by FastAPI).
HTTPX: For making asynchronous HTTP requests to JokeAPI


1. Database Models (SQLAlchemy)

from sqlalchemy import create_engine, Column, Integer, String, Boolean, JSON
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

# Create a base class for SQLAlchemy models
Base = declarative_base()

# Define the Joke model
class Joke(Base):
    __tablename__ = 'jokes'

    id = Column(Integer, primary_key=True, index=True)
    category = Column(String, index=True)
    type = Column(String)
    joke = Column(String, nullable=True)
    setup = Column(String, nullable=True)
    delivery = Column(String, nullable=True)
    nsfw = Column(Boolean)
    political = Column(Boolean)
    sexist = Column(Boolean)
    safe = Column(Boolean)
    lang = Column(String)

# Database URL (SQLite)
DATABASE_URL = "sqlite:///./test.db"

# Set up the database engine and session
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# Create the tables
Base.metadata.create_all(bind=engine)

2. FastAPI Application (API)

from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
import httpx
from typing import List
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from pydantic import BaseModel

# Import database session and model
from .models import SessionLocal, Joke

# Initialize the FastAPI application
app = FastAPI()

# Define the Pydantic model for the Joke response
class JokeResponse(BaseModel):
    category: str
    type: str
    joke: str = None
    setup: str = None
    delivery: str = None
    nsfw: bool
    political: bool
    sexist: bool
    safe: bool
    lang: str

    class Config:
        orm_mode = True

# Function to get the database session
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Function to fetch jokes from JokeAPI
async def fetch_jokes():
    url = "https://v2.jokeapi.dev/joke/Any?amount=100"
    params = {"lang": "en", "safe-mode": "true"}  # Safe mode for jokes
    async with httpx.AsyncClient() as client:
        response = await client.get(url, params=params)
        response.raise_for_status()
        return response.json()

# Function to store jokes in the database
def store_jokes_in_db(jokes_data: List[dict], db: Session):
    for joke_data in jokes_data:
        joke = Joke(
            category=joke_data.get("category"),
            type=joke_data.get("type"),
            joke=joke_data.get("joke") if joke_data["type"] == "single" else None,
            setup=joke_data.get("setup") if joke_data["type"] == "twopart" else None,
            delivery=joke_data.get("delivery") if joke_data["type"] == "twopart" else None,
            nsfw=joke_data["flags"].get("nsfw", False),
            political=joke_data["flags"].get("political", False),
            sexist=joke_data["flags"].get("sexist", False),
            safe=joke_data["safe"],
            lang=joke_data["lang"]
        )
        db.add(joke)
    db.commit()

# Endpoint to fetch jokes and store them in the database
@app.get("/fetch_and_store_jokes", response_model=List[JokeResponse])
async def fetch_and_store_jokes(db: Session = Depends(get_db)):
    try:
        jokes_data = await fetch_jokes()  # Fetch jokes from the external API

        # Process the jokes and store them in the database
        store_jokes_in_db(jokes_data['jokes'], db)
        
        # Return the processed jokes as the response
        return jokes_data['jokes']
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Error fetching jokes: {str(e)}")

3. Run the Fast API Application
Run the FastAPI application using Uvicorn:

uvicorn app:app --reload

Step 4: Testing the API
http://127.0.0.1:8000/fetch_and_store_jokes


Step 5: Database Query 
SELECT * FROM jokes;
