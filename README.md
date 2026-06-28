🤖 Autonomous Multi-Agent Academic Submission & Tracking Pipeline

Welcome to the Autonomous Multi-Agent Academic Submission Pipeline! This repository hosts a hybrid automation system that uses large language models (LLMs) and visual workflow orchestration to automate the tedious parts of academic publishing—from abstract analysis and journal shortlisting to real-time decision tracking and auto-generating cover letters for your next target journals upon rejection.

🌟 How the Hybrid System Works

The architecture consists of two interconnected parts:

The Python Compilation Pipeline (Google Colab or Local Script): Evaluates your manuscript abstract, identifies potential journal targets using modern academic databases, fetches Impact Factors and SCImago Quartiles via Gemini, filters them according to your preferences, ranks them by thematic scope match, and logs the shortlist directly into a Google Sheet. It can be run as a clean automation script or launched as an interactive local GUI Dashboard.

The n8n Active Monitoring Loop: Runs 24/7 in the cloud. It monitors your inbox for emails from journal portals. If a rejection occurs, n8n automatically marks the journal as REJECTED in your Google Sheet, identifies the next prioritized journal, calls Gemini to draft a new custom cover letter alongside publisher guidelines, and drops those assets straight into your inbox.

# 📋 Table of Contents

🌟 How the Hybrid System Works

- [Part 1: The Python Pipeline (Google Colab/Script)](#part-1-the-python-pipeline-google-colabscript)
  - [Prerequisites & Setup](#prerequisites--setup)
  - [Full Python Implementation](#full-python-implementation)
  - [Optional: Interactive Gradio GUI Dashboard](#optional-interactive-gradio-gui-dashboard)
- [Part 2: The n8n Real-time Loop (Autopilot)](#part-2-the-n8n-real-time-loop-autopilot)
  - [Workflow Overview](#workflow-overview)
  - [Importing the n8n JSON Template](#importing-the-n8n-json-template)
- [🛡️ Why We Chose n8n Over a Pure Python Tracking Loop](#️-why-we-chose-n8n-over-a-pure-python-tracking-loop)
  - [1. No Real-Time, 24/7 Screening (Without High Costs)](#1-no-real-time-247-screening-without-high-costs)
  - [2. Painful Gmail API Setup & Maintenance](#2-painful-gmail-api-setup--maintenance)
  - [3. Serious Security Vulnerabilities for Personal Mailboxes](#3-serious-security-vulnerabilities-for-personal-mailboxes)
  - [4. The n8n Solution: Secure, Instant, Visual](#4-the-n8n-solution-secure-instant-visual)
  

# Part 1: The Python Pipeline (Google Colab/Script)

This script automates journal matchmaking and database initialization. It screens potential journal targets against a target abstract and writes the final curated ranking to Google Drive.

## Prerequisites & Setup

Libraries: Install the necessary requirements:

pip install requests pandas gspread google-genai gradio


API Keys: Add GEMINI_API_KEY to your environment variables (or Google Colab Secrets).

Google Sheets Access:
Ensure you run the authentication step so your environment can read/write to your Google Drive.

## Full Python Implementation

Below is the consolidated, production-ready pipeline script. Replace the abstract and spreadsheet configurations with your own data.

```python
import os
import json
import pandas as pd
import gspread
from google import genai
# If using Google Colab, uncomment the next line to authenticate Sheets and retrieve secrets:
from google.colab import auth, userdata
```


⚙️ USER CONFIGURATIONS - REPLACE WITH YOURS
=
1. Your Research Abstract
USER_ABSTRACT = """
[Insert your manuscript abstract here. Make sure it contains enough technical context,
methodologies, and domain keywords so the LLM agent can match journals accurately.]
"""

2. Target Constraints
MIN_IMPACT_FACTOR = 2.0
MAX_IMPACT_FACTOR = 10.0
TARGET_QUARTILES = ["Q1", "Q2"]  # Options: Q1, Q2, Q3, Q4

3. Google Sheets Target
SPREADSHEET_NAME = "My Journal Submissions Tracker"
NEW_TAB_NAME = "Curated Shortlist"


🛠️ INITIALIZING GOOGLE GEMINI ENGINE
=
print("🧠 Connecting to the Gemini API Engine...")
try:
    # Retrieve key from Environment or Colab Userdata
    api_key = os.environ.get('GEMINI_API_KEY') or userdata.get('GEMINI_API_KEY')
    client = genai.Client(api_key=api_key)
    print("✅ Gemini API Engine connected successfully!")
except Exception as e:
    print(f"⚠️ Failed to connect to Gemini API: {e}")
    print("Ensure 'GEMINI_API_KEY' is properly set in your environment or Secrets.")
    client = None


🔍 STEP 1: KEYWORD EXTRACTION AGENT
=
if client:
    print("\n🔍 Extracting core technical keywords from abstract...")
    keyword_prompt = f"""
    Read the following research abstract and extract the 4 to 6 most important core technical keyphrases.
    Wrap each keyphrase in double quotes, and separate them with spaces so a database can search them perfectly.
    Do not include any introductory text, bullet points, or extra explanation. Only return the quoted phrases.

    Abstract:
    {USER_ABSTRACT}
    """
    try:
        response = client.models.generate_content(
            model='models/gemini-flash-latest',
            contents=keyword_prompt
        )
        my_keywords = response.text.strip()
        print(f"🎯 Extracted search query: {my_keywords}")
    except Exception as e:
        print(f"❌ Keyword extraction failed: {e}")
        my_keywords = ""


📊 STEP 2: METRIC ENRICHMENT AGENT
=
Mock search results for structural showcase. Replace this with your search scraper list if integrated.
final_journal_list = [
    {"Journal Name": "Nature Medicine", "Publisher": "Springer Nature", "Matched Article Title": "Integrated multi-omics markers"},
    {"Journal Name": "Cancers", "Publisher": "MDPI", "Matched Article Title": "DNA methylation profiling in tumors"},
    {"Journal Name": "Gastroenterology", "Publisher": "Elsevier", "Matched Article Title": "Colon cancer early detection markers"},
    {"Journal Name": "Bioinformatics", "Publisher": "Oxford Academic", "Matched Article Title": "Multiplex PageRank network analysis"}
]

if client and final_journal_list:
    print(f"\n🧠 Evaluating metrics for {len(final_journal_list)} candidate journals...")
    journal_names = [item["Journal Name"] for item in final_journal_list]
    
    metric_prompt = f"""
    You are an academic database metrics engine. Look up or accurately estimate the current real Impact Factor and the standard SCImago Journal Quartile (Q1, Q2, Q3, or Q4) for the scientific journals listed below.

    Journal List:
    {journal_names}

    Output your entire response strictly as a valid JSON list of objects. Do not include markdown formatting like ```json or any conversational text. Use this exact structure:
    [
        {{"journal_name": "Exact Journal Name", "impact_factor": 4.5, "quartile": "Q1"}}
    ]
    """
    
    try:
        response = client.models.generate_content(
            model='models/gemini-flash-latest',
            contents=metric_prompt
        )
        
        # Clean any accidental markdown code wrappers
        clean_json_str = response.text.replace("```json", "").replace("```", "").strip()
        ai_metrics_list = json.loads(clean_json_str)
        metrics_map = {item["journal_name"].lower().strip(): item for item in ai_metrics_list}
        
        enriched_journals = []
        for entry in final_journal_list:
            original_name = entry["Journal Name"]
            ai_data = metrics_map.get(original_name.lower().strip(), {})
            
            # Cast safety check
            try:
                impact = float(ai_data.get("impact_factor", 1.5))
            except ValueError:
                impact = 1.5
            quartile = ai_data.get("quartile", "Q3").upper().strip()
            
            # Filter matches inside constraints
            if MIN_IMPACT_FACTOR <= impact <= MAX_IMPACT_FACTOR and quartile in TARGET_QUARTILES:
                enriched_journals.append({
                    "Journal Name": original_name,
                    "Impact Factor": impact,
                    "Quartile": quartile,
                    "Publisher": entry["Publisher"],
                    "Matched Article Title": entry["Matched Article Title"]
                })
                
        df_final_metrics = pd.DataFrame(enriched_journals)
        print("🎯 Journal metrics enriched and filtered successfully.")
    except Exception as e:
        print(f"❌ Metrics lookup failed: {e}")
        df_final_metrics = pd.DataFrame(final_journal_list)


🎯 STEP 3: SCOPE FIT & EDITORIAL SCREENING
=
if client and not df_final_metrics.empty:
    print("\n🧠 Screening remaining targets against thematic abstract scope...")
    raw_journals_data = df_final_metrics.to_dict(orient="records")
    
    screen_prompt = f"""
    You are an expert senior academic editor ranking journals for a research paper submission.
    Review the list of candidate journals provided below and evaluate them against the author's target abstract.
    
    Author's Target Abstract:
    {USER_ABSTRACT}
    
    Candidate Journals Data:
    {json.dumps(raw_journals_data, indent=2)}
    
    Evaluation Rules:
    1. Evaluate how well the journal's typical scientific scope aligns with the technical themes in the abstract.
    2. Assign a "Scope Match %" (0% to 100%) and write a brief 1-sentence "Editorial Reason" explaining why it matches.
    
    Output your entire response strictly as a valid JSON list of objects. Do not include markdown formatting or conversational text. Use this exact structure:
    [
        {{
            "Journal Name": "Example Journal",
            "Impact Factor": 4.5,
            "Quartile": "Q1",
            "Publisher": "Example Publisher",
            "Scope Match %": 95,
            "Editorial Reason": "Strong fit because they frequently publish multi-omics network analysis papers."
        }}
    ]
    """
    try:
        response = client.models.generate_content(
            model='models/gemini-flash-latest',
            contents=screen_prompt
        )
        clean_json_str = response.text.replace("```json", "").replace("```", "").strip()
        shortlisted_data = json.loads(clean_json_str)
        
        df_shortlist = pd.DataFrame(shortlisted_data).sort_values(by="Scope Match %", ascending=False)
        print(f"🎉 Compiled Ranked Shortlist! Found {len(df_shortlist)} optimized targets.")
    except Exception as e:
        print(f"❌ Scope match screening failed: {e}")
        df_shortlist = df_final_metrics


💾 STEP 4: SHEET DEPLOYMENT & AUTO-SAVE
=
if not df_shortlist.empty:
    print(f"\n🔐 Connecting to Google Sheets and updating database table...")
    try:
        # If running locally, make sure you configure your Service Account json
        # For Google Colab:
        # auth.authenticate_user()
        # from google.auth import default
        # creds, _ = default()
        # gc = gspread.authorize(creds)
        
        # Fallback local connection
        gc = gspread.oauth() # Assumes credentials are saved in environment
        
        # Connect to Spreadsheet or create a new one
        try:
            sh = gc.open(SPREADSHEET_NAME)
        except gspread.exceptions.SpreadsheetNotFound:
            sh = gc.create(SPREADSHEET_NAME)
            
        # Write to the specific shortlisted sheet tab
        try:
            worksheet = sh.worksheet(NEW_TAB_NAME)
            worksheet.clear()
        except gspread.exceptions.WorksheetNotFound:
            worksheet = sh.add_worksheet(title=NEW_TAB_NAME, rows="100", cols="10")
            
        # Append headers & rows
        headers = list(df_shortlist.columns) + ["Status"] # Adds Status tracker column for n8n
        worksheet.append_row(headers)
        
        # Populate clean data rows
        df_to_save = df_shortlist.copy()
        df_to_save["Status"] = ""  # Set status empty for n8n to monitor
        rows_to_append = df_to_save.fillna("N/A").values.tolist()
        worksheet.append_rows(rows_to_append)
        
        print(f"🎉 GOOGLE SHEET POPULATED! Link to Spreadsheet: https://docs.google.com/spreadsheets/d/{sh.id}")
    except Exception as e:
        print(f"❌ Failed to write to Google Sheets: {e}")


## Optional: Interactive Gradio GUI Dashboard

If you prefer a visual, interactive browser dashboard to run your pipeline dynamically inside your Google Colab canvas, you can execute the code below. This initializes a rich Gradio-powered web interface allowing you to adjust sliders, view curated match matrices, and preview generated letters instantly.
```
import gradio as gr
import pandas as pd
import json
import requests
from google import genai
from google.colab import userdata

def run_consolidated_academic_pipeline(abstract_text, min_if, max_if):
    # ==========================================
    # ❇ ENGINE INITIALIZATION & SYSTEM ACCESS
    # ==========================================
    try:
        GOOGLE_API_KEY = userdata.get('GEMINI_API_KEY')
        client = genai.Client(api_key=GOOGLE_API_KEY)
    except Exception:
        return (
            pd.DataFrame({"Initialization Error": ["GEMINI_API_KEY is missing in your Colab Secrets!"]}),
            "Could not draft letter due to API auth failure.",
            "Could not parse guidelines due to API auth failure."
        )

    if not abstract_text.strip():
        return pd.DataFrame({"Notice": ["Please provide a valid abstract text block."]}), "", ""

    # ==========================================
    # ❐ AGENT 1: TECHNICAL KEYPHRASE EXTRACTOR
    # ==========================================
    keyword_prompt = f"""
    Read the following research abstract and extract the 4 to 6 most important core technical keyphrases.
    Wrap each keyphrase in double quotes, and separate them with spaces so a database can search them perfectly.
    Do not include any introductory text, bullet points, or extra explanation. Only return the quoted phrases.
    Abstract:
    {abstract_text}
    """
    try:
        kw_res = client.models.generate_content(model='models/gemini-flash-latest', contents=keyword_prompt)
        raw_keywords = kw_res.text.strip()
        cleaned_keywords = [k.replace('"', '').strip() for k in raw_keywords.split(" ") if k.strip()]
    except Exception as e:
        return pd.DataFrame({"Processing Error": [f"Keyword extraction failed: {e}"]}), "", ""

    # ==========================================
    # ❐ AGENT 2: CROSSREF DATABASE SCOUT
    # ==========================================
    final_journal_list = []
    seen_journals = set()

    for kw in cleaned_keywords[:3]: # Optimize payload boundaries for fast processing speed
        try:
            url = f"https://api.crossref.org/works?query={kw.replace(' ', '+')}&rows=12"
            res = requests.get(url, headers={'User-Agent': 'AcademicPipelineDashboard/1.0'}, timeout=5).json()
            for item in res.get('message', {}).get('items', []):
                if 'container-title' in item and item['container-title']:
                    j_name = item['container-title'][0]
                    if j_name.lower().strip() not in seen_journals:
                        seen_journals.add(j_name.lower().strip())
                        final_journal_list.append({
                            "Journal Name": j_name,
                            "Publisher": item.get('publisher', 'Unknown Publisher'),
                            "Matched Article Title": item.get('title', ['Untitled Publication'])[0]
                        })
        except Exception:
            pass

    if not final_journal_list:
        return pd.DataFrame({"Database Alert": ["No unique targets surfaced from metadata queries."]}), "", ""

    # ==========================================
    # ❐ AGENT 3: METRICS SCREENER & CHANNELS
    # ==========================================
    journal_names = [item["Journal Name"] for item in final_journal_list][:20]

    metrics_prompt = f"""
    You are an academic database metrics engine.
    Look up or accurately estimate the current real Impact Factor and the standard SCImago Journal Quartile (Q1, Q2, Q3, or Q4) for the scientific journals listed below.
    If a journal is very niche, provide your best accurate scholarly estimate for its metrics.
    Journal List:
    {journal_names}

    Output your entire response strictly as a valid JSON list of objects.
    Do not include markdown formatting like ```json or any conversational text.
    Use this exact structure:
    [
        {{"journal_name": "Exact Journal Name", "impact_factor": 4.5, "quartile": "Q1"}}
    ]
    """

    try:
        metrics_res = client.models.generate_content(model='models/gemini-flash-latest', contents=metrics_prompt)
        clean_json_str = metrics_res.text.replace("```json", "").replace("```", "").strip()
        ai_metrics_list = json.loads(clean_json_str)
        metrics_map = {item["journal_name"].lower().strip(): item for item in ai_metrics_list}
    except Exception:
        metrics_map = {}

    # ==========================================
    # ❐ AGENT 4: CUSTOM SCOPE RE-EVALUATION
    # ==========================================
    enriched_journals = []
    for entry in final_journal_list:
        name = entry["Journal Name"]
        ai_data = metrics_map.get(name.lower().strip(), {})

        try:
            impact = float(ai_data.get("impact_factor", 1.5))
        except ValueError:
            impact = 1.5

        quartile = ai_data.get("quartile", "Q3").upper().strip()

        # Filtering Criteria (Impact Factor boundaries + Quality filters Q1 or Q2)
        if float(min_if) <= impact <= float(max_if) and quartile in ["Q1", "Q2"]:
            enriched_journals.append({
                "Journal Name": name,
                "Impact Factor": impact,
                "Quartile": quartile,
                "Publisher": entry["Publisher"],
                "Matched Target Title": entry["Matched Article Title"]
            })

    if not enriched_journals:
        return pd.DataFrame({"Curation Alert": ["No journals passed your IF boundary and Q1/Q2 filters."]}), "", ""

    # Sort results exactly according to your matrix sorting algorithm
    df_final_metrics = pd.DataFrame(enriched_journals).sort_values(by="Impact Factor", ascending=False)

    # ==========================================
    # ❐ AGENT 5: COVER LETTER DRAFTER
    # ==========================================
    top_journal_name = df_final_metrics.iloc[0]["Journal Name"]

    cover_prompt = f"""
    You are a professional academic writing assistant. Write a formal, persuasive Cover Letter to the Editor-in-Chief for a research manuscript submission.
    Target Journal: {top_journal_name}
    Manuscript Abstract: {abstract_text}

    Requirements:
    1. Use a professional, standard academic cover letter layout (with bracketed placeholders for the date, author name, and affiliation).
    2. Clearly state the title of the manuscript (use a placeholder title like "[Insert Your Manuscript Title Here]").
    3. Explicitly explain why this work aligns with the specific scope of {top_journal_name}. Mention keywords like multi-omics, DNA methylation, and network biology.
    4. Explicitly state that the manuscript is original, has not been published elsewhere, and all authors have approved the submission.
    5. Maintain an elegant, confident, and respectful scholarly tone. Keep it concise (under 400 words).
    """

    try:
        cover_res = client.models.generate_content(model='models/gemini-flash-latest', contents=cover_prompt)
        generated_cover_letter = cover_res.text.strip()
    except Exception as e:
        generated_cover_letter = f"Error compiling letter artifacts: {e}"

    # ==========================================
    # ❐ AGENT 6: STRUCTURAL COMPLIANCE ANALYSIS
    # ==========================================
    top_publisher = df_final_metrics.iloc[0]["Publisher"]

    analysis_prompt = f"""
    You are a professional structural journal formatting agent. Analyze the author guidelines profile for the following journal.
    Journal Name: {top_journal_name}
    Publisher: {top_publisher}

    Based on standard academic publishing constraints for this publisher, extract and return a clean, structured submission checklist.
    You must address the following points explicitly:
    1. Mandatory Section Order: What is the exact step-by-step sequence of sections required? (e.g., Title Page -> Abstract -> Introduction -> ... -> Figures -> Tables).
    2. Figures and Tables Rule: Must figures and tables be embedded directly within the manuscript text file, or must they be uploaded as completely separate high-resolution source files?
    3. Mandatory File Types: What file types are required or accepted for the text, figures, and tables?
    4. Crucial Hidden Rules: Highlight any word count limits, abstract length caps, or specific citation style notes (e.g., Vancouver, Harvard).

    Format your response with clear bold headers, bullet points, and an explicit checklist structure so the researcher can prepare their files perfectly.
    """

    try:
        guidelines_res = client.models.generate_content(model='models/gemini-flash-latest', contents=analysis_prompt)
        generated_guidelines = guidelines_res.text.strip()
    except Exception as e:
        generated_guidelines = f"Error extracting formatting protocols: {e}"

    return df_final_metrics, generated_cover_letter, generated_guidelines

# ==========================================
# ❐ GRAPHICAL INTERFACE BUILDING BLOCK
# ==========================================
with gr.Blocks(theme=gr.themes.Default()) as academic_app:
    gr.Markdown("# ⁶ Automated Multi-Agent Editorial Dashboard")
    gr.Markdown("Submit your research parameters to execute keyword extraction, database searches, metrics filtering, letter formatting, and publisher compliance analytics in sequence.")

    with gr.Row():
        with gr.Column(scale=2):
            gr.Markdown("### 📥 Input Panel")
            abstract_box = gr.Textbox(
                label="Manuscript Abstract Block",
                value="Colon cancer (CRC) remains a major health burden, highlighting the need for specific, non-invasive biomarkers for early detection. Aberrant DNA methylation is a hallmark of cancer, yet CRC-specific markers are still limited. We performed an integrative multi-omics analysis...",
                lines=10
            )
            with gr.Row():
                min_if = gr.Slider(minimum=0.0, maximum=10.0, value=2.0, step=0.5, label="Min Impact Factor")
                max_if = gr.Slider(minimum=5.0, maximum=30.0, value=10.0, step=0.5, label="Max Impact Factor")

            run_btn = gr.Button("🚀 Trigger Pipeline Execution Engine", variant="primary")

        with gr.Column(scale=3):
            gr.Markdown("### 📊 Output Pipeline Data")
            with gr.Tabs():
                with gr.TabItem("🎯 Curation Shortlist Matrix"):
                    matrix_output = gr.Dataframe(label="Filtered Targets (Q1/Q2 Only)")

                with gr.TabItem("📄 Custom Submission Cover Letter"):
                    letter_output = gr.Textbox(label="Editor Cover Letter Artifact", lines=15, show_copy_button=True)

                with gr.TabItem("📋 Publisher Guidelines & Section Order"):
                    guidelines_output = gr.Textbox(label="Structural Alignment Rules Checklist", lines=15, show_copy_button=True)

    run_btn.click(
        fn=run_consolidated_academic_pipeline,
        inputs=[abstract_box, min_if, max_if],
        outputs=[matrix_output, letter_output, guidelines_output]
    )

# Render seamlessly inside your Google Colab display canvas
academic_app.launch(inline=True)
```

# Part 2: The n8n Real-time Loop (Autopilot)

While the Python script is excellent for doing initial calculations and analysis, you don't want to run it manually every time an editor sends you an email. This is where n8n comes in!

## Workflow Overview

The fully_automated_n8n_loop.json template on your visual canvas serves as a 24/7 background listener.

New Gmail Received: Checks your inbox once every minute for any emails matching academic review/submission keywords.

Parsing Script: Analyzes the body of the email and categorizes it into REJECTED, REVISED, ACCEPTED, or PENDING completely hands-free.

Google Sheets Update: Rewrites the status of that specific journal in your tracker row.

Autonomous Switching Logic:

If Accepted/Revised: Pauses the pipeline and emails you with instructions/congratulations.

If Rejected: Automatically queries your Google Sheet, identifies your next-highest prioritised journal with an empty status, calls Gemini's standard guidelines API, generates a New Cover Letter and Publisher Submission Guidelines for that target, and emails both straight to your personal inbox.

## Importing the n8n JSON Template

Copy the raw JSON array in fully_automated_n8n_loop.json from the Canvas.
```
```
Log into your n8n workspace in your browser.

Open a brand new, empty workflow.

Left-click once directly on the grid canvas, then press Ctrl + V (or Cmd + V on Mac).

Open the nodes and authorize your Google Sheets and Gmail modules.

Replace the Google Spreadsheet ID parameter with your exact spreadsheet ID from the URL .

Save the workflow and toggle the active switch to Active (Green)!

Now your academic pipeline is running on pure autopilot!

# 🛡️ Why We Chose n8n Over a Pure Python Tracking Loop

When designing this pipeline, we initially considered writing a pure Python script (e.g., using the imaplib or google-api-python-client libraries) to poll Gmail, update the spreadsheet, and trigger the loop. However, this approach carries severe limitations that make it highly impractical and insecure for production:

## 1. No Real-Time, 24/7 Screening (Without High Costs)

Google Colab is interactive and temporary; it will time out and shut down minutes after you close your browser tab. To run a Python script 24/7, you would have to rent an expensive virtual private server (VPS) or rely on cloud crons (like PythonAnywhere) which only run scripts once a day. This means your script wouldn't notice a rejection until up to 24 hours later, whereas n8n monitors your inbox dynamically and responds instantly.

## 2. Painful Gmail API Setup & Maintenance

Establishing a secure connection to Google Services in raw Python is notoriously difficult. It requires creating a Google Cloud Developer project, turning on the Gmail API, configuring OAuth consent screens, generating security tokens, downloading a fragile credentials.json file, and writing complex authentication refresh handlers in Python.

## 3. Serious Security Vulnerabilities for Personal Mailboxes

Because Python scripts store authentication tokens locally or in environment variables, a pure Python approach forces you to either:

Expose Google App Passwords: Saving custom app passwords or raw login credentials in plain-text script files is a major threat to your Google account's safety.

Expose Local Tokens: Storing offline OAuth tokens inside a public GitHub repository or shared Google Colab notebook poses a severe security risk, potentially granting bad actors full read/write access to your entire personal or institutional Gmail inbox.

## 4. The n8n Solution: Secure, Instant, Visual

n8n bypasses these issues completely. It handles authentication using the industry-standard, secure OAuth 2.0 protocol directly inside its sandboxed, encrypted database. You never have to write custom authentication handlers, your credentials are never exposed in open-source code, and the tool triggers event-driven tasks instantaneously, without draining system resources or costing a dime.

License

Distributed under the MIT License. Feel free to use and adapt this system to automate your own research submissions!
