import streamlit as st
import datetime
import os
import google.generativeai as genai
from dotenv import load_dotenv
from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build

# --- 1. SETTINGS & SECURE CONFIG ---
load_dotenv()
genai.configure(api_key=os.getenv("GEMINI_API_KEY"))
model = genai.GenerativeModel('gemini-1.5-flash')

# Full scope to allow adding events
SCOPES = ['https://www.googleapis.com/auth/calendar']

def get_calendar_service():
    creds = None
    if os.path.exists('token.json'):
        creds = Credentials.from_authorized_user_file('token.json', SCOPES)
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            flow = InstalledAppFlow.from_client_secrets_file('credentials.json', SCOPES)
            creds = flow.run_local_server(port=0)
        with open('token.json', 'w') as token:
            token.write(creds.to_json())
    return build('calendar', 'v3', credentials=creds)

# --- 2. CALENDAR LOGIC ---
def get_calendar_events():
    service = get_calendar_service()
    now = datetime.datetime.utcnow().isoformat() + 'Z'
    events_result = service.events().list(calendarId='primary', timeMin=now,
                                        maxResults=10, singleEvents=True,
                                        orderBy='startTime').execute()
    events = events_result.get('items', [])
    
    # Remove duplicates based on summary and start time
    seen = set()
    unique_events = []
    for event in events:
        key = (event.get('summary'), event['start'].get('dateTime', event['start'].get('date')))
        if key not in seen:
            seen.add(key)
            unique_events.append(event)
    
    return unique_events

def add_all_day_event(summary, event_date):
    """Adds an event that lasts the entire day."""
    service = get_calendar_service()
    date_str = event_date.isoformat()
    
    event = {
        'summary': summary,
        'start': {'date': date_str},
        'end': {'date': date_str},
    }
    return service.events().insert(calendarId='primary', body=event).execute()

# --- 3. AI LOGIC ---
def ai_summarize_meetings(events):
    if not events:
        return "You have a clear schedule. Perfect for deep work."
    
    lines = []
    for e in events:
        start = e['start'].get('dateTime', e['start'].get('date'))
        lines.append(f"- {e['summary']} at {start}")
    
    meeting_data = "\n".join(lines)
    prompt = f"As a professional assistant, provide a 2-sentence summary of these tasks/meetings:\n{meeting_data}"
    
    try:
        response = model.generate_content(prompt)
        return response.text
    except:
        return "Review your upcoming timeline below."

# --- 4. STREAMLIT UI (Crimson Dark Mode) ---
st.set_page_config(page_title="Crimson AI Assistant", layout="wide")

st.markdown("""
    <style>
    .stApp { background: linear-gradient(180deg, #1a0505 0%, #000000 100%); color: white; }
    h1 { color: #ff4b4b; font-size: 2.8rem; text-align: center; text-shadow: 0px 0px 15px rgba(255, 75, 75, 0.4); }
    .stButton>button {
        background: linear-gradient(90deg, #8b0000 0%, #ff4b4b 100%);
        color: white; border-radius: 25px; border: none; padding: 0.8rem; font-weight: bold; width: 100%;
        transition: 0.3s ease;
    }
    .stButton>button:hover { transform: scale(1.02); box-shadow: 0px 0px 20px rgba(255, 75, 75, 0.4); }
    .summary-box {
        background: rgba(255, 75, 75, 0.1); border-left: 4px solid #ff4b4b;
        padding: 1.5rem; border-radius: 12px; margin: 1.5rem 0;
    }
    .event-item {
        background: rgba(255, 255, 255, 0.03); padding: 1.5rem; border-radius: 12px; 
        margin: 1rem 0; border: 1px solid rgba(255, 75, 75, 0.2);
    }
    .event-title { color: #ff4b4b; font-size: 1.25rem; font-weight: 600; }
    </style>
""", unsafe_allow_html=True)

col1, col2, col3 = st.columns([1, 2, 1])

with col2:
    st.markdown("<h1>Calendar Assistant</h1>", unsafe_allow_html=True)
    st.markdown("<p style='text-align: center; color: #9ca3af;'>Intelligent Schedule Management</p>", unsafe_allow_html=True)

    # --- ADD EVENT SECTION ---
    with st.expander("➕ Add Event"):
        with st.form("quick_add"):
            new_title = st.text_input("Event Title", placeholder="What's happening?")
            e_date = st.date_input("Date", datetime.date.today())
            
            submit = st.form_submit_button("Confirm Event")
            
            if submit:
                if new_title:
                    try:
                        add_all_day_event(new_title, e_date)
                        st.success(f"'{new_title}' is now set for the entire day!")
                    except Exception as e:
                        st.error(f"Error: {e}")
                else:
                    st.warning("Please provide an event title.")

    st.divider()

    # --- VIEW EVENTS SECTION ---
    if st.button("My Schedule"):
        with st.spinner("Syncing with Google Calendar..."):
            try:
                events = get_calendar_events()
                if not events:
                    st.info("Your calendar is currently clear.")
                else:
                    summary = ai_summarize_meetings(events)
                    st.markdown(f"<div class='summary-box'><strong style='color: #ff4b4b;'>AI INSIGHT:</strong><br>{summary}</div>", unsafe_allow_html=True)
                    
                    st.markdown("### Upcoming Timeline")
                    for event in events:
                        
                        start_info = event['start'].get('dateTime', event['start'].get('date'))
                        
                        
                        if 'T' not in start_info:
                            display_time = f"📅 {start_info}"
                        else:
                            
                            dt_obj = datetime.datetime.fromisoformat(start_info.replace('Z', ''))
                            display_time = f"🕒 {dt_obj.strftime('%b %d, %I:%M %p')}"

                        st.markdown(f"""
                            <div class='event-item'>
                                <div class='event-title'>{event['summary']}</div>
                                <div style='color: #9ca3af;'>{display_time}</div>
                            </div>
                        """, unsafe_allow_html=True)
            except Exception as e:
                st.error(f"Access Error: {e}")
