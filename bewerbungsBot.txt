import streamlit as st
import pandas as pd
import openai
import requests
from bs4 import BeautifulSoup

# OpenAI API Key setzen
openai.api_key = "DEIN_OPENAI_API_KEY"

# --- WEB-APP LAYOUT ---
st.title("🔍 Intelligenter Bewerbungs-Bot")
st.sidebar.header("🔧 Einstellungen")

# Nutzerpräferenzen eingeben
standort = st.sidebar.text_input("📍 Bevorzugter Standort", "Frankfurt am Main")
branche = st.sidebar.text_input("🏢 Bevorzugte Branche", "Marketing/Kommunikation")
unternehmensgröße = st.sidebar.selectbox("🏢 Unternehmensgröße", ["Alle", "Startup", "Mittelständisch", "Konzern"])

st.sidebar.markdown("---")

# --- JOB-SCRAPING ---
def scrape_jobs(query, location):
    url = f"https://www.stepstone.de/jobs?q={query}&location={location}"
    headers = {"User-Agent": "Mozilla/5.0"}
    response = requests.get(url, headers=headers)
    soup = BeautifulSoup(response.text, "html.parser")
    jobs = []
    
    for job in soup.find_all("article", class_="job-element"):
        title = job.find("h2").text.strip()
        link = job.find("a")["href"]
        company = job.find("span", class_="company").text.strip() if job.find("span", class_="company") else "Unbekannt"
        jobs.append({"Jobtitel": title, "Unternehmen": company, "Link": link})
    
    return jobs

if st.button("🔍 Jobs suchen"):
    st.write(f"Suche nach Jobs in {standort} in der Branche {branche}...")
    jobs = scrape_jobs(branche, standort)
    df_jobs = pd.DataFrame(jobs)
    st.dataframe(df_jobs)
    st.session_state.jobs = df_jobs

# --- KI-gestütztes Matching ---
def match_job_to_cv(job_description, cv_text):
    prompt = f"""
    Vergleiche folgende Stellenbeschreibung mit meinem Lebenslauf und bewerte die Passung auf einer Skala von 1-10:
    
    Stellenbeschreibung:
    {job_description}
    
    Mein Lebenslauf:
    {cv_text}
    
    Antwort: (nur die Zahl 1-10 ausgeben)
    """
    
    response = openai.ChatCompletion.create(
        model="gpt-4",
        messages=[{"role": "system", "content": "Du bist ein Bewerbungs-Experte."},
                  {"role": "user", "content": prompt}]
    )
    
    return int(response["choices"][0]["message"]["content"].strip())

# --- Anschreiben-Erstellung ---
def generate_cover_letter(job_title, company_name, job_description, cv_text):
    prompt = f"""
    Schreibe ein professionelles Anschreiben für die folgende Stelle:
    
    Jobtitel: {job_title}
    Unternehmen: {company_name}
    Stellenbeschreibung:
    {job_description}
    
    Mein Lebenslauf:
    {cv_text}
    
    Das Anschreiben sollte freundlich, professionell und überzeugend sein.
    """
    
    response = openai.ChatCompletion.create(
        model="gpt-4",
        messages=[{"role": "system", "content": "Du bist ein Experte für Bewerbungsschreiben."},
                  {"role": "user", "content": prompt}]
    )
    
    return response["choices"][0]["message"]["content"]

# --- Feedback-Mechanismus ---
if "jobs" in st.session_state:
    st.subheader("🎯 Bewerte die Job-Vorschläge")
    df_jobs = st.session_state.jobs
    for index, row in df_jobs.iterrows():
        score = st.slider(f"Bewertung für {row['Jobtitel']} bei {row['Unternehmen']}", 1, 10, 5)
        st.write(f"Deine Bewertung: {score}/10")
    
    st.subheader("✍ Generiere ein Anschreiben")
    selected_job = st.selectbox("Wähle einen Job für das Anschreiben", df_jobs["Jobtitel"])
    if st.button("📝 Anschreiben generieren"):
        job_info = df_jobs[df_jobs["Jobtitel"] == selected_job].iloc[0]
        cover_letter = generate_cover_letter(job_info["Jobtitel"], job_info["Unternehmen"], "Keine ausführliche Beschreibung verfügbar", "DEIN LEBENSLAUF-TEXT")
        st.text_area("📄 Dein Anschreiben", cover_letter, height=300)
        st.session_state.cover_letter = cover_letter
