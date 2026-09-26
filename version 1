import streamlit as st
import json
import hashlib
import os
import smtplib
import ssl
import time
from email.message import EmailMessage
from pathlib import Path
from datetime import datetime, date, timedelta

import pandas as pd
import requests
from geopy.geocoders import Nominatim
from geopy.distance import geodesic


# ============================================================
# 1. BASIC APP SETUP
# ============================================================

st.set_page_config(
    page_title="Cancer Care Companion",
    page_icon="💙",
    layout="wide",
)

DATA_DIR = Path("data")
USERS_FILE = DATA_DIR / "users.json"
EMAIL_OUTBOX_FILE = DATA_DIR / "email_outbox.json"

DATA_DIR.mkdir(exist_ok=True)

if not USERS_FILE.exists():
    USERS_FILE.write_text("{}", encoding="utf-8")

if not EMAIL_OUTBOX_FILE.exists():
    EMAIL_OUTBOX_FILE.write_text("[]", encoding="utf-8")


# ============================================================
# 2. JSON STORAGE
# ============================================================

def load_json(path, default):
    """Load JSON safely."""
    try:
        with open(path, "r", encoding="utf-8") as file:
            return json.load(file)
    except (FileNotFoundError, json.JSONDecodeError):
        return default


def save_json(path, data):
    """Save JSON in a readable format."""
    with open(path, "w", encoding="utf-8") as file:
        json.dump(data, file, indent=4)


def load_users():
    return load_json(USERS_FILE, {})


def save_users(users):
    save_json(USERS_FILE, users)


def get_user(username):
    users = load_users()
    return users.get(username)


def save_user(username, user_data):
    users = load_users()
    users[username] = user_data
    save_users(users)


# ============================================================
# 3. LOGIN HELPERS
# ============================================================

def hash_password(password):
    """Simple hash for a school prototype."""
    return hashlib.sha256(password.encode("utf-8")).hexdigest()


def new_user_record(password):
    """Blank data structure for a new account."""
    return {
        "password_hash": hash_password(password),
        "personal": {
            "name": "",
            "carer_name": "",
            "carer_email": "",
            "address": "",
            # These are prototype defaults only.
            # A real patient app should use thresholds set by the care team.
            "temperature_high_alert": 38.0,
            "temperature_low_alert": 35.0
        },
        "medications": [],
        "appointments": [],
        "checkins": []
    }


# ============================================================
# 4. EMAIL ALERTS
# ============================================================

def save_email_to_outbox(to_email, subject, body):
    """If SMTP is not configured, save the message as a demo alert."""
    outbox = load_json(EMAIL_OUTBOX_FILE, [])
    outbox.append({
        "created_at": datetime.now().isoformat(timespec="seconds"),
        "to": to_email,
        "subject": subject,
        "body": body
    })
    save_json(EMAIL_OUTBOX_FILE, outbox)


def send_carer_email(to_email, subject, body):
    """
    Send an email using SMTP environment variables.

    Required environment variables:
    CANCERCARE_SMTP_HOST
    CANCERCARE_SMTP_PORT
    CANCERCARE_SMTP_USER
    CANCERCARE_SMTP_PASSWORD
    CANCERCARE_FROM_EMAIL

    If they are missing, the alert is saved to data/email_outbox.json.
    """
    if not to_email:
        return False, "No carer email has been saved."

    host = os.getenv("CANCERCARE_SMTP_HOST")
    port = int(os.getenv("CANCERCARE_SMTP_PORT", "465"))
    username = os.getenv("CANCERCARE_SMTP_USER")
    password = os.getenv("CANCERCARE_SMTP_PASSWORD")
    from_email = os.getenv("CANCERCARE_FROM_EMAIL", username or "")

    if not all([host, username, password, from_email]):
        save_email_to_outbox(to_email, subject, body)
        return False, "Email settings are not configured, so the alert was saved to the demo outbox."

    message = EmailMessage()
    message["From"] = from_email
    message["To"] = to_email
    message["Subject"] = subject
    message.set_content(body)

    try:
        if port == 465:
            context = ssl.create_default_context()
            with smtplib.SMTP_SSL(host, port, context=context) as server:
                server.login(username, password)
                server.send_message(message)
        else:
            with smtplib.SMTP(host, port) as server:
                server.starttls(context=ssl.create_default_context())
                server.login(username, password)
                server.send_message(message)

        return True, "Carer email sent."
    except Exception as error:
        save_email_to_outbox(to_email, subject, body)
        return False, f"Email could not be sent, so it was saved to the demo outbox. ({error})"


# ============================================================
# 5. LOCATION + DISTANCE HELPERS
# ============================================================

def geocode_address(address):
    """Convert an address into latitude and longitude."""
    if not address.strip():
        return None

    geolocator = Nominatim(user_agent="cancer-care-companion-school-project")
    location = geolocator.geocode(address, country_codes="gb", timeout=10)

    if location is None:
        return None

    return {
        "lat": location.latitude,
        "lon": location.longitude,
        "display_name": location.address
    }


def distance_between_addresses(address_1, address_2):
    """
    Return straight-line distance between two addresses.
    This is not driving distance.
    """
    first = geocode_address(address_1)
    second = geocode_address(address_2)

    if not first or not second:
        return None

    km = geodesic(
        (first["lat"], first["lon"]),
        (second["lat"], second["lon"])
    ).km

    return {
        "km": round(km, 1),
        "miles": round(km * 0.621371, 1)
    }


def nearest_hospital_from_address(address):
    """
    Use OpenStreetMap data to find nearby places tagged as hospitals.
    Returns the nearest by straight-line distance.
    """
    home = geocode_address(address)

    if not home:
        return None

    lat = home["lat"]
    lon = home["lon"]

    query = f"""
    [out:json][timeout:25];
    (
      node["amenity"="hospital"](around:20000,{lat},{lon});
      way["amenity"="hospital"](around:20000,{lat},{lon});
      relation["amenity"="hospital"](around:20000,{lat},{lon});
    );
    out center tags;
    """

    response = requests.post(
        "https://overpass-api.de/api/interpreter",
        data={"data": query},
        timeout=30
    )
    response.raise_for_status()

    elements = response.json().get("elements", [])

    hospitals = []

    for element in elements:
        hospital_lat = element.get("lat")
        hospital_lon = element.get("lon")

        if hospital_lat is None or hospital_lon is None:
            center = element.get("center", {})
            hospital_lat = center.get("lat")
            hospital_lon = center.get("lon")

        if hospital_lat is None or hospital_lon is None:
            continue

        tags = element.get("tags", {})
        name = tags.get("name", "Unnamed hospital")

        km = geodesic((lat, lon), (hospital_lat, hospital_lon)).km

        hospitals.append({
            "name": name,
            "km": round(km, 1),
            "miles": round(km * 0.621371, 1),
            "lat": hospital_lat,
            "lon": hospital_lon
        })

    if not hospitals:
        return None

    hospitals.sort(key=lambda item: item["km"])
    return hospitals[0]


# ============================================================
# 6. HEALTH CHECK-IN HELPERS
# ============================================================

def temperature_status(temperature, low_alert=35.0, high_alert=38.0):
    """
    Prototype traffic-light system.

    RED:
    - at/above the configured high alert
    - below the configured low alert

    AMBER:
    - within 0.5 C of high alert
    - within 1 C above low alert

    GREEN:
    - otherwise

    This is a check-in flag, not a diagnosis.
    """
    if temperature >= high_alert or temperature < low_alert:
        return "RED"

    if temperature >= high_alert - 0.5 or temperature < low_alert + 1.0:
        return "AMBER"

    return "GREEN"


def calculate_streak(checkins):
    """Count consecutive days with at least one check-in."""
    if not checkins:
        return 0

    checkin_dates = sorted(
        {datetime.fromisoformat(item["submitted_at"]).date() for item in checkins},
        reverse=True
    )

    today = date.today()

    if checkin_dates[0] not in {today, today - timedelta(days=1)}:
        return 0

    streak = 1
    expected = checkin_dates[0] - timedelta(days=1)

    for checkin_date in checkin_dates[1:]:
        if checkin_date == expected:
            streak += 1
            expected -= timedelta(days=1)
        elif checkin_date < expected:
            break

    return streak


def medication_alert_text(medication):
    """Work out whether a medication reminder is near its saved time."""
    try:
        saved_time = datetime.strptime(medication["time"], "%H:%M").time()
    except (ValueError, KeyError):
        return "Time not set"

    now = datetime.now()
    med_datetime = datetime.combine(now.date(), saved_time)
    difference_minutes = (med_datetime - now).total_seconds() / 60

    if -30 <= difference_minutes <= 30:
        return "DUE NOW"
    elif 30 < difference_minutes <= 120:
        return "COMING UP"
    elif difference_minutes < -30:
        return "PASSED"
    else:
        return "LATER"


# ============================================================
# 7. LOGIN / REGISTER SCREEN
# ============================================================

def authentication_screen():
    st.title("💙 Cancer Care Companion")
    st.write(
        "A simple Python prototype for check-ins, medication reminders, "
        "appointments and carer updates."
    )

    st.info(
        "Prototype only: this app is not a replacement for medical advice, "
        "an oncology team, NHS 111 or emergency services."
    )

    login_tab, register_tab = st.tabs(["Log in", "Create account"])

    with login_tab:
        with st.form("login_form"):
            username = st.text_input("Username")
            password = st.text_input("Password", type="password")
            submitted = st.form_submit_button("Log in")

        if submitted:
            users = load_users()
            user = users.get(username.strip())

            if user and user["password_hash"] == hash_password(password):
                st.session_state.logged_in = True
                st.session_state.username = username.strip()
                st.rerun()
            else:
                st.error("Username or password was not recognised.")

    with register_tab:
        with st.form("register_form"):
            new_username = st.text_input("Choose a username")
            new_password = st.text_input("Choose a password", type="password")
            confirm_password = st.text_input("Confirm password", type="password")
            create = st.form_submit_button("Create account")

        if create:
            new_username = new_username.strip()
            users = load_users()

            if len(new_username) < 3:
                st.error("Choose a username with at least 3 characters.")
            elif new_username in users:
                st.error("That username already exists.")
            elif len(new_password) < 6:
                st.error("Use a password with at least 6 characters.")
            elif new_password != confirm_password:
                st.error("The passwords do not match.")
            else:
                users[new_username] = new_user_record(new_password)
                save_users(users)
                st.success("Account created. You can now log in.")


# ============================================================
# 8. HOME PAGE
# ============================================================

def home_page(username, user):
    st.title("🏠 Home")

    name = user["personal"].get("name") or username
    st.subheader(f"Hello, {name}")

    streak = calculate_streak(user["checkins"])

    col1, col2, col3 = st.columns(3)

    col1.metric("Check-in streak", f"{streak} day{'s' if streak != 1 else ''}")
    col2.metric("Total check-ins", len(user["checkins"]))
    col3.metric("Medications saved", len(user["medications"]))

    st.divider()

    st.subheader("🔔 Today's alerts")

    alerts_found = False
    today = date.today().isoformat()

    # Medication reminders
    for medication in user["medications"]:
        status = medication_alert_text(medication)

        if status in {"DUE NOW", "COMING UP"}:
            alerts_found = True
            st.warning(
                f"Medication: **{medication['name']}** "
                f"({medication.get('dose', 'dose not entered')}) "
                f"at **{medication['time']}** — {status}"
            )

    # Appointments
    for appointment in user["appointments"]:
        if appointment.get("date") == today:
            alerts_found = True
            distance_text = ""
            if appointment.get("distance_km") is not None:
                distance_text = (
                    f" — about {appointment['distance_km']} km "
                    f"({appointment['distance_miles']} miles) straight-line from home"
                )

            st.info(
                f"Appointment today: **{appointment['title']}** at "
                f"**{appointment['time']}**, {appointment['venue']}{distance_text}"
            )

    if not alerts_found:
        st.success("No medication or appointment alerts are due right now.")

    st.divider()

    left, right = st.columns(2)

    with left:
        st.subheader("📅 Upcoming appointments")

        upcoming = []
        now = datetime.now()

        for appointment in user["appointments"]:
            try:
                appointment_dt = datetime.fromisoformat(
                    f"{appointment['date']}T{appointment['time']}"
                )
            except ValueError:
                continue

            if appointment_dt >= now:
                upcoming.append((appointment_dt, appointment))

        upcoming.sort(key=lambda item: item[0])

        if upcoming:
            for appointment_dt, appointment in upcoming[:5]:
                distance = ""
                if appointment.get("distance_km") is not None:
                    distance = (
                        f" | {appointment['distance_km']} km "
                        f"({appointment['distance_miles']} miles)"
                    )

                st.write(
                    f"**{appointment['title']}**  \n"
                    f"{appointment_dt.strftime('%d %b %Y at %H:%M')}  \n"
                    f"{appointment['venue']}{distance}"
                )
        else:
            st.write("No upcoming appointments have been added.")

    with right:
        st.subheader("💊 Medication summary")

        if user["medications"]:
            for medication in user["medications"]:
                st.write(
                    f"**{medication['name']}** — "
                    f"{medication.get('dose', 'Dose not entered')} — "
                    f"{medication['time']}"
                )
        else:
            st.write("No medications have been added yet.")

        st.subheader("👤 Personal details")
        st.write(f"**Name:** {user['personal'].get('name') or 'Not added'}")
        st.write(f"**Carer:** {user['personal'].get('carer_name') or 'Not added'}")
        st.write(f"**Address:** {user['personal'].get('address') or 'Not added'}")


# ============================================================
# 9. PERSONAL DETAILS PAGE
# ============================================================

def personal_details_page(username, user):
    st.title("👤 Personal Details")

    personal = user["personal"]

    with st.form("personal_details_form"):
        name = st.text_input("Name", value=personal.get("name", ""))
        carer_name = st.text_input(
            "Carer / trusted adult name",
            value=personal.get("carer_name", "")
        )
        carer_email = st.text_input(
            "Carer email",
            value=personal.get("carer_email", "")
        )
        address = st.text_input(
            "Home address or postcode",
            value=personal.get("address", ""),
            help="Used only to estimate distances in this prototype."
        )

        st.caption(
            "Temperature thresholds should ideally match instructions from the "
            "patient's own care team."
        )

        high_temp = st.number_input(
            "High temperature alert threshold (°C)",
            min_value=35.0,
            max_value=42.0,
            value=float(personal.get("temperature_high_alert", 38.0)),
            step=0.1
        )

        low_temp = st.number_input(
            "Low temperature alert threshold (°C)",
            min_value=30.0,
            max_value=37.0,
            value=float(personal.get("temperature_low_alert", 35.0)),
            step=0.1
        )

        save_details = st.form_submit_button("Save personal details")

    if save_details:
        personal["name"] = name.strip()
        personal["carer_name"] = carer_name.strip()
        personal["carer_email"] = carer_email.strip()
        personal["address"] = address.strip()
        personal["temperature_high_alert"] = float(high_temp)
        personal["temperature_low_alert"] = float(low_temp)

        user["personal"] = personal
        save_user(username, user)
        st.success("Personal details saved.")

    st.divider()

    st.subheader("💊 Add medication")

    with st.form("add_medication_form", clear_on_submit=True):
        medication_name = st.text_input("Medication name")
        dose = st.text_input("Dose / instructions")
        medication_time = st.time_input("Daily reminder time")
        add_medication = st.form_submit_button("Add medication")

    if add_medication:
        if not medication_name.strip():
            st.error("Enter the medication name.")
        else:
            user["medications"].append({
                "id": datetime.now().strftime("%Y%m%d%H%M%S%f"),
                "name": medication_name.strip(),
                "dose": dose.strip(),
                "time": medication_time.strftime("%H:%M")
            })
            save_user(username, user)
            st.success("Medication added.")
            st.rerun()

    if user["medications"]:
        st.write("### Saved medications")
        for medication in list(user["medications"]):
            col_a, col_b = st.columns([5, 1])
            with col_a:
                st.write(
                    f"**{medication['name']}** — "
                    f"{medication.get('dose') or 'No dose notes'} — "
                    f"{medication['time']}"
                )
            with col_b:
                if st.button(
                    "Delete",
                    key=f"delete_med_{medication['id']}"
                ):
                    user["medications"] = [
                        item for item in user["medications"]
                        if item["id"] != medication["id"]
                    ]
                    save_user(username, user)
                    st.rerun()

    st.divider()

    st.subheader("📅 Add appointment")

    with st.form("appointment_form", clear_on_submit=True):
        appointment_title = st.text_input(
            "Appointment",
            placeholder="e.g. Oncology appointment"
        )
        appointment_date = st.date_input("Date")
        appointment_time = st.time_input("Time")
        venue = st.text_input(
            "Hospital / clinic name",
            placeholder="e.g. University College Hospital"
        )
        appointment_address = st.text_input(
            "Hospital / clinic address or postcode"
        )

        add_appointment = st.form_submit_button("Add appointment")

    if add_appointment:
        if not appointment_title.strip() or not venue.strip():
            st.error("Add an appointment title and venue.")
        else:
            distance = None

            if personal.get("address") and appointment_address.strip():
                try:
                    distance = distance_between_addresses(
                        personal["address"],
                        appointment_address.strip()
                    )
                except Exception:
                    distance = None

            appointment = {
                "id": datetime.now().strftime("%Y%m%d%H%M%S%f"),
                "title": appointment_title.strip(),
                "date": appointment_date.isoformat(),
                "time": appointment_time.strftime("%H:%M"),
                "venue": venue.strip(),
                "address": appointment_address.strip(),
                "distance_km": distance["km"] if distance else None,
                "distance_miles": distance["miles"] if distance else None
            }

            user["appointments"].append(appointment)
            save_user(username, user)

            if distance:
                st.success(
                    f"Appointment added. It is approximately "
                    f"{distance['km']} km ({distance['miles']} miles) "
                    f"away in a straight line."
                )
            else:
                st.success("Appointment added.")

            st.rerun()

    if user["appointments"]:
        st.write("### Saved appointments")
        appointments_sorted = sorted(
            user["appointments"],
            key=lambda item: (item["date"], item["time"])
        )

        for appointment in appointments_sorted:
            col_a, col_b = st.columns([5, 1])

            with col_a:
                distance_text = ""
                if appointment.get("distance_km") is not None:
                    distance_text = (
                        f" — {appointment['distance_km']} km "
                        f"({appointment['distance_miles']} miles)"
                    )

                st.write(
                    f"**{appointment['title']}** — "
                    f"{appointment['date']} at {appointment['time']} — "
                    f"{appointment['venue']}{distance_text}"
                )

            with col_b:
                if st.button(
                    "Delete",
                    key=f"delete_appointment_{appointment['id']}"
                ):
                    user["appointments"] = [
                        item for item in user["appointments"]
                        if item["id"] != appointment["id"]
                    ]
                    save_user(username, user)
                    st.rerun()

    st.divider()

    st.subheader("🏥 Find a nearby hospital")
    st.caption(
        "This uses OpenStreetMap and reports straight-line distance. "
        "It is a convenience feature, not an emergency-service locator."
    )

    if st.button("Find nearest mapped hospital"):
        if not personal.get("address"):
            st.error("Save a home address or postcode first.")
        else:
            try:
                with st.spinner("Looking for nearby hospitals..."):
                    hospital = nearest_hospital_from_address(personal["address"])

                if hospital:
                    st.success(
                        f"Nearest mapped hospital found: **{hospital['name']}** — "
                        f"about {hospital['km']} km ({hospital['miles']} miles) "
                        f"away in a straight line."
                    )
                else:
                    st.warning("No hospital was found within the search area.")
            except Exception as error:
                st.warning(
                    "The map service could not be reached. "
                    f"Try again later. ({error})"
                )


# ============================================================
# 10. QUESTIONNAIRE PAGE
# ============================================================

def questionnaire_page(username, user):
    st.title("📝 Daily Check-in")

    st.write(
        "Complete one check-in each day. The results are saved to your JSON data "
        "and used on the Stats page."
    )

    personal = user["personal"]

    with st.form("daily_questionnaire"):
        sleep_hours = st.number_input(
            "Roughly how many hours did you sleep?",
            min_value=0.0,
            max_value=24.0,
            value=8.0,
            step=0.5
        )

        temperature = st.number_input(
            "What was your temperature today? (°C)",
            min_value=30.0,
            max_value=45.0,
            value=37.0,
            step=0.1
        )

        mood = st.slider(
            "What is your mood like?",
            min_value=1,
            max_value=10,
            value=5,
            help="1 = very low, 10 = very positive"
        )

        tiredness = st.slider(
            "How tired are you?",
            min_value=1,
            max_value=10,
            value=5,
            help="1 = not tired, 10 = extremely tired"
        )

        st.write("Anything else you want to log?")
        nausea = st.checkbox("Nausea")
        pain = st.checkbox("Pain")
        dizziness = st.checkbox("Dizziness")
        appetite_change = st.checkbox("Change in appetite")
        other_symptom = st.checkbox("Something else")

        message_to_carer = st.text_area(
            "Optional message for your carer / trusted adult"
        )

        send_message = st.checkbox(
            "Email my written message to my carer when I submit"
        )

        submit = st.form_submit_button("Submit check-in")

    if submit:
        low_alert = float(personal.get("temperature_low_alert", 35.0))
        high_alert = float(personal.get("temperature_high_alert", 38.0))

        status = temperature_status(
            float(temperature),
            low_alert=low_alert,
            high_alert=high_alert
        )

        symptoms = []
        if nausea:
            symptoms.append("Nausea")
        if pain:
            symptoms.append("Pain")
        if dizziness:
            symptoms.append("Dizziness")
        if appetite_change:
            symptoms.append("Change in appetite")
        if other_symptom:
            symptoms.append("Other")

        checkin = {
            "submitted_at": datetime.now().isoformat(timespec="seconds"),
            "sleep_hours": float(sleep_hours),
            "temperature": float(temperature),
            "temperature_status": status,
            "mood": int(mood),
            "tiredness": int(tiredness),
            "symptoms": symptoms,
            "message_to_carer": message_to_carer.strip()
        }

        user["checkins"].append(checkin)
        save_user(username, user)

        if status == "GREEN":
            st.success("🟢 Temperature check-in: GREEN")
        elif status == "AMBER":
            st.warning(
                "🟠 Temperature check-in: AMBER. "
                "The reading is close to one of the saved alert thresholds."
            )
        else:
            st.error(
                "🔴 Temperature check-in: RED. "
                "The saved carer alert will be triggered."
            )

        carer_email = personal.get("carer_email", "")
        patient_name = personal.get("name") or username

        if status == "RED":
            subject = f"Cancer Care Companion alert for {patient_name}"
            body = (
                f"Cancer Care Companion recorded a RED temperature check-in.\n\n"
                f"Name: {patient_name}\n"
                f"Temperature entered: {temperature:.1f} °C\n"
                f"Time: {datetime.now().strftime('%d %b %Y at %H:%M')}\n\n"
                f"This automated alert is not a diagnosis. Please follow the "
                f"patient's own care-team instructions."
            )

            sent, message = send_carer_email(
                carer_email,
                subject,
                body
            )

            if sent:
                st.success(message)
            else:
                st.info(message)

        if send_message and message_to_carer.strip():
            subject = f"Check-in message from {patient_name}"
            body = (
                f"{patient_name} sent this message from Cancer Care Companion:\n\n"
                f"{message_to_carer.strip()}\n\n"
                f"Check-in time: {datetime.now().strftime('%d %b %Y at %H:%M')}"
            )

            sent, message = send_carer_email(
                carer_email,
                subject,
                body
            )

            if sent:
                st.success("Message emailed to carer.")
            else:
                st.info(message)

        st.success("Check-in saved.")


# ============================================================
# 11. STATS PAGE
# ============================================================

def stats_page(user):
    st.title("📈 Stats & Progress")

    checkins = user["checkins"]

    if not checkins:
        st.info("Complete your first questionnaire to see your stats.")
        return

    dataframe = pd.DataFrame(checkins)
    dataframe["date"] = pd.to_datetime(dataframe["submitted_at"])
    dataframe = dataframe.sort_values("date")

    average_sleep = dataframe["sleep_hours"].mean()
    average_mood = dataframe["mood"].mean()
    average_temperature = dataframe["temperature"].mean()
    average_tiredness = dataframe["tiredness"].mean()

    col1, col2, col3, col4 = st.columns(4)

    col1.metric("Average sleep", f"{average_sleep:.1f} h")
    col2.metric("Average mood", f"{average_mood:.1f}/10")
    col3.metric("Average temperature", f"{average_temperature:.1f} °C")
    col4.metric("Average tiredness", f"{average_tiredness:.1f}/10")

    st.caption(
        "Averages summarise what has been entered. They do not decide whether "
        "a reading is medically safe."
    )

    chart_data = dataframe.set_index("date")

    st.subheader("Sleep")
    st.line_chart(chart_data[["sleep_hours"]])

    st.subheader("Mood")
    st.line_chart(chart_data[["mood"]])

    st.subheader("Temperature")
    st.line_chart(chart_data[["temperature"]])

    st.subheader("Tiredness")
    st.line_chart(chart_data[["tiredness"]])

    st.divider()

    st.subheader("🏆 Check-in rewards")

    total = len(checkins)
    streak = calculate_streak(checkins)

    rewards = []

    if total >= 1:
        rewards.append("🌟 First check-in — started tracking")
    if total >= 3:
        rewards.append("📝 Three check-ins — building a routine")
    if total >= 7:
        rewards.append("💙 Seven check-ins — one week of records")
    if streak >= 3:
        rewards.append("🔥 3-day streak")
    if streak >= 7:
        rewards.append("🏅 7-day streak")

    if rewards:
        for reward in rewards:
            st.success(reward)
    else:
        st.write("Complete check-ins to unlock consistency rewards.")

    with st.expander("View check-in history"):
        display_columns = [
            "submitted_at",
            "sleep_hours",
            "temperature",
            "temperature_status",
            "mood",
            "tiredness",
            "symptoms"
        ]

        st.dataframe(
            dataframe[display_columns],
            use_container_width=True,
            hide_index=True
        )


# ============================================================
# 12. WELLBEING PAGE
# ============================================================

def wellbeing_page():
    st.title("🌿 Wellbeing")

    st.write(
        "This page is for simple relaxation and support resources. "
        "It does not replace professional or medical support."
    )

    st.subheader("🫁 Guided breathing")

    st.write(
        "Try one gentle cycle: breathe in for 4 seconds, hold for 4, "
        "then breathe out for 6."
    )

    if st.button("Start one breathing cycle"):
        placeholder = st.empty()
        progress = st.progress(0)

        phases = [
            ("Breathe in", 4),
            ("Hold", 4),
            ("Breathe out slowly", 6)
        ]

        total_seconds = sum(seconds for _, seconds in phases)
        completed = 0

        for phase, seconds in phases:
            for remaining in range(seconds, 0, -1):
                placeholder.markdown(
                    f"## {phase}\n### {remaining}"
                )
                time.sleep(1)
                completed += 1
                progress.progress(min(completed / total_seconds, 1.0))

        placeholder.markdown("## Cycle complete 🌿")
        st.success("Finished.")

    st.divider()

    st.subheader("💙 More support")
    st.markdown(
        "- [Macmillan Cancer Support](https://www.macmillan.org.uk/cancer-information-and-support)\n"
        "- [NHS cancer information](https://www.nhs.uk/conditions/cancer/)\n"
        "- [Cancer Research UK](https://www.cancerresearchuk.org/about-cancer)\n"
        "- [Macmillan local support finder](https://www.macmillan.org.uk/cancer-information-and-support/in-your-area)"
    )


# ============================================================
# 13. EMAIL OUTBOX PAGE (DEMO / DEVELOPMENT)
# ============================================================

def demo_outbox_page():
    st.title("📨 Demo Email Outbox")
    st.write(
        "When real SMTP email is not configured, carer alerts are saved here "
        "so you can still demonstrate the feature."
    )

    outbox = load_json(EMAIL_OUTBOX_FILE, [])

    if not outbox:
        st.info("No demo emails have been created.")
        return

    for item in reversed(outbox):
        with st.expander(
            f"{item['subject']} — {item['created_at']}"
        ):
            st.write(f"**To:** {item['to']}")
            st.code(item["body"])

    if st.button("Clear demo outbox"):
        save_json(EMAIL_OUTBOX_FILE, [])
        st.rerun()


# ============================================================
# 14. MAIN APP
# ============================================================

if "logged_in" not in st.session_state:
    st.session_state.logged_in = False

if "username" not in st.session_state:
    st.session_state.username = None


if not st.session_state.logged_in:
    authentication_screen()
    st.stop()


username = st.session_state.username
user = get_user(username)

if user is None:
    st.session_state.logged_in = False
    st.session_state.username = None
    st.rerun()


with st.sidebar:
    st.title("💙 Companion")
    st.caption(f"Logged in as {username}")

    page = st.radio(
        "Go to",
        [
            "Home",
            "Personal Details",
            "Questionnaire",
            "Stats",
            "Wellbeing",
            "Demo Email Outbox"
        ]
    )

    st.divider()

    if st.button("Log out"):
        st.session_state.logged_in = False
        st.session_state.username = None
        st.rerun()


if page == "Home":
    home_page(username, user)

elif page == "Personal Details":
    personal_details_page(username, user)

elif page == "Questionnaire":
    questionnaire_page(username, user)

elif page == "Stats":
    stats_page(user)

elif page == "Wellbeing":
    wellbeing_page()

elif page == "Demo Email Outbox":
    demo_outbox_page()


st.divider()
st.caption(
    "Cancer Care Companion — school prototype. "
    "Do not use this prototype as a real clinical monitoring system."
)
