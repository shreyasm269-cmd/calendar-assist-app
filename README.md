# calendar-assist-app
Productivity Technology &amp; Calendar Management
import os
import datetime
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from google.auth.transport.requests import Request
from googleapiclient.discovery import build
import streamlit as st
from langchain_openai import ChatOpenAI
from langchain.agents import initialize_agent, Tool
from langchain.agents import AgentType

# --- 1. GOOGLE CALENDAR SETUP ---
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

service = get_calendar_service()

# --- 2. LOGIC & TOOLS ---
def check_conflicts(start_time, end_time):
    events_result = service.events().list(
        calendarId='primary', timeMin=start_time, timeMax=end_time,
        singleEvents=True).execute()
    return events_result.get('items', [])

def create_event(query):
    """
    Expects a string with format: 'Summary, StartISO, EndISO'
    Example: 'Math Class, 2024-05-10T10:00:00Z, 2024-05-10T11:00:00Z'
    """
    details = [x.strip() for x in query.split(',')]
    summary, start, end = details[0], details[1], details[2]
    
    conflicts = check_conflicts(start, end)
    if conflicts:
        return f"Conflict detected with: {conflicts[0].get('summary')}. Event not added."
    
    event = {
        'summary': summary,
        'start': {'dateTime': start, 'timeZone': 'UTC'},
        'end': {'dateTime': end, 'timeZone': 'UTC'},
    }
    service.events().insert(calendarId='primary', body=event).execute()
    return f"Successfully scheduled {summary}."

def list_upcoming_events(n=10):
    now = datetime.datetime.utcnow().isoformat() + 'Z'
    events_result = service.events().list(
        calendarId='primary', timeMin=now, maxResults=n, 
        singleEvents=True, orderBy='startTime').execute()
    events = events_result.get('items', [])
    return "\n".join([f"{e['start'].get('dateTime', e['start'].get('date'))}: {e['summary']}" for e in events])

# --- 3. AI AGENT CONFIG ---
llm = ChatOpenAI(model="gpt-4", temperature=0) # Or use Gemini via LangChain
tools = [
    Tool(name="CreateEvent", func=create_event, description="Useful for scheduling classes or assignments. Input: 'Summary, StartISO, EndISO'"),
    Tool(name="ListEvents", func=list_upcoming_events, description="Lists upcoming schedule items.")
]

agent = initialize_agent(tools, llm, agent=AgentType.STRUCTURED_CHAT_ZERO_SHOT_REACT_DESCRIPTION, verbose=True)

# --- 4. STREAMLIT UI ---
st.set_page_config(page_title="AI Student Assistant", layout="wide")
st.title("📅 Smart Student Calendar")

if "messages" not in st.session_state:
    st.session_state.messages = []

col1, col2 = st.columns([1, 1])

with col1:
    st.subheader("Chat with your Assistant")
    for message in st.session_state.messages:
        with st.chat_message(message["role"]):
            st.markdown(message["content"])

    if prompt := st.chat_input("e.g., Schedule Chemistry Lab for tomorrow at 2pm"):
        st.session_state.messages.append({"role": "user", "content": prompt})
        with st.chat_message("user"):
            st.markdown(prompt)

        with st.chat_message("assistant"):
            # Context injection: Tell the AI what 'today' is
            today_context = f"Today is {datetime.date.today()}. "
            response = agent.run(today_context + prompt)
            st.markdown(response)
            st.session_state.messages.append({"role": "assistant", "content": response})

with col2:
    st.subheader("Upcoming Schedule")
    if st.button("Refresh Schedule"):
        schedule = list_upcoming_events(15)
        st.text_area("Your Agenda", value=schedule, height=400)
