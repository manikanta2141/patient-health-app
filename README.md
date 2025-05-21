# patient-health-app
!pip install streamlit fastapi uvicorn cryptography pydantic nest_asyncio pyngrok
# utils.py

from cryptography.fernet import Fernet
import datetime

key = Fernet.generate_key()
cipher = Fernet(key)

def encrypt_data(data):
    return cipher.encrypt(data)

def decrypt_data(encrypted):
    return cipher.decrypt(encrypted)

def extract_metadata(file):
    return {
        "filename": file.name,
        "upload_date": str(datetime.date.today()),
        "file_type": file.type
    }
# api.py

from fastapi import FastAPI
from pydantic import BaseModel
from typing import Dict, List

app = FastAPI()
database = {}

class Record(BaseModel):
    patient_id: str
    metadata: Dict
    encrypted_file: str

@app.post("/upload")
def upload_record(record: Record):
    if record.patient_id not in database:
        database[record.patient_id] = []
    database[record.patient_id].append({
        "metadata": record.metadata,
        "file": record.encrypted_file
    })
    return {"message": "Uploaded successfully"}

@app.get("/patient/{patient_id}")
def get_patient_data(patient_id: str):
    return database.get(patient_id, [])
# Launch FastAPI using uvicorn inside notebook
import nest_asyncio
from pyngrok import ngrok
import uvicorn

nest_asyncio.apply()

public_url = ngrok.connect(8000)
print("Your FastAPI endpoint:", public_url)

uvicorn.run(app, port=8000)
# app.py (Streamlit UI)
import streamlit as st
from utils import encrypt_data, extract_metadata
import requests
import base64

st.title("🏥 Health Records Management System")

role = st.sidebar.selectbox("Login as", ["Patient", "Doctor"])

if role == "Patient":
    st.header(" Upload Health Record")
    patient_id = st.text_input("Patient ID")
    uploaded_file = st.file_uploader("Upload File")
    if uploaded_file and patient_id:
        metadata = extract_metadata(uploaded_file)
        encrypted = encrypt_data(uploaded_file.read())
        encoded_file = base64.b64encode(encrypted).decode("utf-8")
        record = {
            "patient_id": patient_id,
            "metadata": metadata,
            "encrypted_file": encoded_file
        }
        response = requests.post("https://your-fastapi-url/upload", json=record)
        st.success("File uploaded successfully!")

elif role == "Doctor":
    st.header(" Search Patient Records")
    patient_id = st.text_input("Enter Patient ID")
    if st.button("Fetch Records"):
        response = requests.get(f"https://your-fastapi-url/patient/{patient_id}")
        st.json(response.json())
