# Flashcard
main.py = from fastapi import FastAPI
from pydantic import BaseModel
from fastapi.middleware.cors import CORSMiddleware
from openai import OpenAI
import json
from dotenv import load_dotenv
import os 

# Initialize FastAPI
app = FastAPI()
@app.get("/")
async def root():
    return {"message": "AI Flashcards API is running 🚀"}

# Allow CORS (for React frontend)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # change to ["http://localhost:5173"] for security
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize OpenAI client

load_dotenv()
OPENAI_API_KEY  = os.getenv("OPENAI_API_KEY")

if not OPENAI_API_KEY:
    raise ValueError("OpenAI API key not found!")
client = OpenAI(api_key=OPENAI_API_KEY)

# Request body model
class TextInput(BaseModel):
    text: str

@app.post("/generate")
async def generate_flashcards(input_data: TextInput):
    text = input_data.text.strip()

    if not text:
        return {"flashcards": []}

    prompt = f"""
    Create 5 flashcards from this study material.
    Return valid JSON in the format:
    [
        {{"question":"...", "answer":"..."}},
        {{"question":"...", "answer":"..."}}
    ]

    Text: {text}
    """

    response = client.chat.completions.create(
        model="gpt-4o-mini",  # lightweight, cheap, and fast
        messages=[{"role": "user", "content": prompt}],
        temperature=0.7
    )

    raw_output = response.choices[0].message.content.strip()

    # Try parsing JSON safely
    try:
        flashcards = json.loads(raw_output)
    except:
        # Fallback: wrap in list if JSON isn't perfect
        flashcards = [{"question": "Error parsing AI output", "answer": raw_output}]

    return {"flashcards": flashcards}
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="localhost", port=8000)
React.py = import React, { useState } from "react";
import { motion } from "framer-motion";

export default function FlashcardsApp() {
  const [text, setText] = useState("");
  const [flashcards, setFlashcards] = useState([]);
  const [loading, setLoading] = useState(false);

  const generateFlashcards = async () => {
    if (!text.trim()) return alert("Please enter study material");
    setLoading(true);

    try {
      const response = await fetch("http://localhost:5000/generate", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ text }),
      });

      const data = await response.json();
      setFlashcards(data.flashcards); // Expecting [{question:"", answer:""}, ...]
    } catch (error) {
      console.error(error);
      alert("Error generating flashcards");
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="min-h-screen bg-gray-100 p-6">
      <h1 className="text-3xl font-bold text-center mb-6">
        📘 AI-Powered Flashcards
      </h1>

      {/* Input Section */}
      <textarea
        className="w-full p-3 rounded-lg border border-gray-300 mb-4"
        rows="6"
        placeholder="Paste your study notes here..."
        value={text}
        onChange={(e) => setText(e.target.value)}
      />

      <button
        onClick={generateFlashcards}
        className="bg-blue-600 text-white px-4 py-2 rounded-lg hover:bg-blue-700 transition"
      >
        {loading ? "Generating..." : "Generate Flashcards"}
      </button>

      {/* Flashcards Section */}
      <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4 mt-6">
        {flashcards.map((card, idx) => (
          <motion.div
            key={idx}
            className="bg-white shadow-md rounded-2xl p-6 cursor-pointer"
            whileHover={{ scale: 1.05 }}
            onClick={() => {
              setFlashcards((prev) =>
                prev.map((c, i) =>
                  i === idx ? { ...c, flipped: !c.flipped } : c
                )
              );
            }}
          >
            <p className="text-lg font-semibold">
              {card.flipped ? `A: ${card.answer}` : `Q: ${card.question}`}
            </p>
          </motion.div>
        ))}
      </div>
    </div>
  );
}

    
