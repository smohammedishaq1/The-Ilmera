import streamlit as st
import json
import os # Import os for better file path handling

# --- Configuration ---
# Determine the directory of the script to reliably find the JSON file
script_dir = os.path.dirname(os.path.abspath(__file__))
JSON_FILE_PATH = os.path.join(script_dir, 'career_data.json') # Make sure career_data.json is in the same folder as your script

# --- Functions ---
@st.cache_data # Cache the data loading for efficiency after first load
def load_career_data(file_path):
    """Loads career data from the specified JSON file."""
    if not os.path.exists(file_path):
        st.error(f"Error: '{os.path.basename(file_path)}' not found at expected path: {file_path}")
        st.info(f"Please ensure the JSON file is in the same directory as the Python script ({script_dir}).")
        return []
    try:
        # Specify encoding for broader compatibility, especially on Windows
        with open(file_path, 'r', encoding='utf-8') as f:
            # --- DIAGNOSTIC STEP: Read the first ~500 characters ---
            # This helps diagnose if the file is being read incorrectly if JSONDecodeError occurs
            file_content_start_debug = f.read(500)
            f.seek(0) # Reset file pointer
            # --- END DIAGNOSTIC STEP ---

            data = json.load(f)
            # Ensure the root key 'IT_Roles' exists and is a list
            if "IT_Roles" in data and isinstance(data["IT_Roles"], list):
                # Successfully loaded the list of roles
                return data["IT_Roles"]
            else:
                st.error(f"Error: JSON file '{os.path.basename(file_path)}' should contain a top-level key 'IT_Roles' with a list of roles.")
                return []
    except json.JSONDecodeError as e:
        st.error(f"Error: Could not decode JSON from '{os.path.basename(file_path)}'. Please check the file for formatting errors (e.g., trailing commas, incorrect quotes).")
        # --- DIAGNOSTIC STEP ---
        st.error(f"JSONDecodeError details: {e}") # Print the specific error from json module
        st.subheader("Start of file content being read (DEBUG):")
        # Display the raw text Python tried to read - useful for finding hidden chars or BOMs
        st.code(file_content_start_debug, language=None)
        # --- END DIAGNOSTIC STEP ---
        return []
    except Exception as e:
        st.error(f"An unexpected error occurred while loading data: {e}")
        return []

# --- Main App Logic ---
st.set_page_config(page_title="AI Career Planner", layout="wide") # Use wide layout for better table display
st.title("🧠 AI Career Planner for Engineers")
st.markdown("Enter your desired IT role below, or select from the list, to get a personalized roadmap.")

# Load career data
# This function reads the JSON_FILE_PATH
career_data = load_career_data(JSON_FILE_PATH)

# Check if data loading was successful
if not career_data:
    st.warning("Could not load career data. Unable to proceed. Please check the JSON file format and path.")
    st.stop() # Stop execution if data loading failed

# --- UI Elements ---
# THIS PART DYNAMICALLY CREATES THE LIST OF ROLES FROM THE LOADED DATA
# It iterates through 'career_data' (which holds all roles from the JSON)
# and extracts the 'role_name' for each entry.
available_roles = sorted([role.get("role_name", "Unknown Role")
                         for role in career_data if role.get("role_name")]) # Filter out entries potentially missing role_name

# THIS SELECTBOX USES THE DYNAMICALLY CREATED 'available_roles' LIST
# It will automatically include all 140 roles if they were loaded correctly from the JSON.
job_role_input = st.selectbox(
    "🎯 Select or type your target job role:",
    options=[""] + available_roles, # Start with an empty option, then add all roles
)

# --- Processing and Display ---
if job_role_input: # Check if a role was selected or entered
    with st.spinner(f"Generating career roadmap for {job_role_input}..."):
        # Find the matched data (case-insensitive comparison)
        job_role_lower = job_role_input.strip().lower()
        matched_data = None
        for role in career_data:
            # Make comparison more robust by stripping whitespace from JSON data too
            if role.get("role_name", "").strip().lower() == job_role_lower:
                matched_data = role
                break

        if matched_data:
            st.success(f"### Career Roadmap for: {matched_data['role_name']}")

            # Use columns for better layout of skills, tools etc.
            col1, col2 = st.columns(2)

            with col1:
                st.subheader("🛠️ Technical Skills")
                tech_skills = matched_data.get("technical_skills", [])
                if tech_skills:
                    st.markdown("\n".join([f"- {skill}" for skill in tech_skills]))
                else:
                    st.caption("N/A")

                st.subheader("🤝 Soft Skills")
                soft_skills = matched_data.get("soft_skills", [])
                if soft_skills:
                     st.markdown("\n".join([f"- {skill}" for skill in soft_skills]))
                else:
                    st.caption("N/A")

                st.subheader("💡 Project Ideas")
                projects = matched_data.get("projects", [])
                if projects:
                    st.markdown("\n".join([f"- {p}" for p in projects]))
                else:
                    st.caption("N/A")

                st.subheader("🔧 Tools")
                tools = matched_data.get("tools", [])
                if tools:
                    st.markdown("\n".join([f"- {tool}" for tool in tools]))
                else:
                    st.caption("N/A")

            with col2:
                st.subheader("🧪 Internships")
                internships = matched_data.get("internships", [])
                if internships:
                     st.markdown("\n".join([f"- {r}" for r in internships]))
                else:
                    st.caption("N/A")

                st.subheader("🏢 Company Types")
                company_types = matched_data.get("company_types", [])
                if company_types:
                     st.markdown("\n".join([f"- {t}" for t in company_types]))
                else:
                    st.caption("N/A")

                st.subheader("📚 Prerequisite Subjects")
                prereqs = matched_data.get("prerequisite_subjects", [])
                if prereqs:
                     st.markdown("\n".join([f"- {sub}" for sub in prereqs]))
                else:
                    st.caption("N/A")

                st.subheader("🧠 Interview Preparation Topics")
                interview_topics = matched_data.get("interview_preparation", {}).get("topics", [])
                if interview_topics:
                     st.markdown("\n".join([f"- {topic}" for topic in interview_topics]))
                else:
                    st.caption("N/A")

            st.divider()

            st.subheader("🎓 Top Courses")
            courses = matched_data.get("courses", [])
            if courses:
                course_data_for_display = [
                    {
                        "Course": c.get("course_name", "N/A"),
                        "Platform": c.get("platform", "N/A"),
                        "Link": c.get("link", None)
                    }
                    for c in courses
                ]
                st.dataframe(
                    course_data_for_display,
                    column_config={
                        "Link": st.column_config.LinkColumn("Link", display_text="Go to Course")
                    },
                    use_container_width=True,
                    hide_index=True
                )
            else:
                st.caption("N/A")

            st.subheader("📜 Certifications")
            certifications = matched_data.get("certifications", [])
            if certifications:
                 certification_data_for_display = [
                    {
                        "Certification": cert.get("certification_name", "N/A"),
                        "Platform": cert.get("platform", "N/A"),
                        "Link": cert.get("link", None)
                    }
                    for cert in certifications
                ]
                 st.dataframe(
                     certification_data_for_display,
                     column_config={
                        "Link": st.column_config.LinkColumn("Link", display_text="Go to Certification")
                     },
                     use_container_width=True,
                     hide_index=True
                )
            else:
                st.caption("N/A")

        elif job_role_input: # Added this check back
             st.error(f"Sorry, no data found for the job role '{job_role_input}'. Please check the spelling or select from the list.")

# Add a footer or info message
st.markdown("---")
st.info(f"Data loaded from `{os.path.basename(JSON_FILE_PATH)}`. Ensure the file is up-to-date and correctly formatted.")