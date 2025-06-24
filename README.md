# EZ
Building a Smart Assistant for Research Summarization
import numpy as np
import pandas as pd
import requests
import streamlit as st
import openai
import matplotlib
import pdfplumber
from docx import Document
from openpyxl import load_workbook
import pandas as pd
import openai
import streamlit as st
from sqlalchemy import create_engine

openai_api_key = 'your OpenAI key here'

st.title("Document Summarization and Chatbot Web Application ")
"""
Note: This chatbot will give a warning if you put any sensitive information
"""
def get_openai_api_key():
    return st.session_state.get("openai_api_key", "")

# Function to set the OpenAI API key in session state
def set_openai_api_key(api_key):
    st.session_state["openai_api_key"] = api_key

# Check if the OpenAI API key is already set in session state
if get_openai_api_key():
    # Your existing chatbot code goes here
    # Make sure to replace `openai_api_key` with `get_openai_api_key()` in your function calls
    pass
else:
    # Input for OpenAI API key
    api_key = st.text_input("Enter your OpenAI API Key to start:", type="password")
    
    # Button to set the API key and start the chatbot
    if st.button("Set API Key and Start"):
        if api_key:  # Check if the API key is not empty
            set_openai_api_key(api_key)
            st.success("API Key set successfully. Refresh or continue with the chatbot.")
            # You might need to instruct users to refresh the page or automatically rerun the app
            st.experimental_rerun()
        else:
            st.error("Please enter a valid OpenAI API Key.")

# Example of proceeding after setting the API key
if get_openai_api_key():
    st.write("Proceed with your chatbot functionalities here.")


DATABASE_URI = 'mysql+pymysql://username:password@localhost:xxxx/employees'
engine = create_engine(DATABASE_URI)

def save_dataframe_to_database(dataframe, table_name):
    dataframe.to_sql(table_name, engine, if_exists='replace', index=False, method='multi')

# Function to check for personal information using OpenAI's content filter
def check_for_personal_info(prompt,openai_api_key):
    openai.api_key = openai_api_key
    try:
        response = openai.ChatCompletion.create(
            model="gpt-3.5-turbo",
            messages=[
                {"role": "system", "content": "You're a highly intelligent assistant. Your task is to determine if the following message contains any personal information such as names, email addresses, phone numbers, or any other details that could be used to identify an individual.Please Classify it only as 1 or 0 where 1 means it contains personal information and 0 means it doesnt contain any personal information"},
                {"role": "user", "content": prompt}
            ]
        )
        
        ai_response = response['choices'][0]['message']['content'].strip()  # Adjusted based on correct API usage
        return ai_response
    except Exception as e:
        print(f"An error occurred: {e}")
        return None

def file_check(prompt,openai_api_key):
    openai.api_key = openai_api_key
    try:
        response = openai.ChatCompletion.create(
            model="gpt-3.5-turbo",
            messages=[
                {"role": "system", "content": "You're a highly intelligent assistant. Your task is to read the file . Please summarize the file in 100 words only"},
                {"role": "user", "content": prompt}

            ]
        )
        
        ai_response = response['choices'][0]['message']['content'].strip()  # Adjusted based on correct API usage
        return ai_response
    except Exception as e:
        print(f"An error occurred: {e}")
        return None
    
if "messages" not in st.session_state:
    st.session_state["messages"] = [
        {"role": "system", "content": "You are a serious assistant, you are an AI enthusiast."}
    ]

# Display previous messages
for message in st.session_state["messages"]:
    # Check the role to determine how to display the message
    if message["role"] == "user":
        st.chat_message("user").write(message["content"])
    elif message["role"] == "assistant":
        st.chat_message("assistant").write(message["content"])
    else:  # Default case, for system messages or any other type
        st.text(message["content"])  # Using st.text for system messages or other roles

if prompt := st.chat_input():
    openai.api_key = openai_api_key
    if check_for_personal_info(prompt,openai_api_key)=='1':
        st.warning("Warning: Please do not share personal information.")
    else:
        st.session_state.messages.append({"role": "user", "content": prompt})
    
    # Assuming this function call is correctly displaying the user's message in your setup
        st.chat_message("user").write(prompt)
    
        response = openai.ChatCompletion.create(model="gpt-3.5-turbo-0613", messages=st.session_state.messages)
        msg = response.choices[0].message
        st.session_state.messages.append({"role": "assistant", "content": msg.content})
    
    # Display the assistant's response immediately after getting it
        st.chat_message("assistant").write(msg.content)

if st.button('Clear Conversation'):
    st.session_state.messages = [
        {"role": "system", "content": ""}
    ]
    st.experimental_rerun()


uploaded_file = st.file_uploader("Upload a file for a quick summary", type=['txt','csv', 'pdf', 'docx','xlsx'], help="Upload a text, PDF, or Word document")


if uploaded_file is not None:
    # Initialize file_content variable
    file_content = ""
    
    # Handling different file types
    if uploaded_file.type == "text/plain":
        # Reading text file
        file_content = str(uploaded_file.read(), "utf-8")
    elif uploaded_file.type == "application/pdf":
        # Reading PDF file
        with pdfplumber.open(uploaded_file) as pdf:
            file_content = ''.join(page.extract_text() for page in pdf.pages)
    elif uploaded_file.name.endswith('.csv'):
        dataframe = pd.read_csv(uploaded_file)
        st.write("Uploaded CSV Data:")
        st.dataframe(dataframe)
        table_name = st.text_input("Enter the table name for the database:")
        if st.button("Save to Database"):
            if table_name:
                save_dataframe_to_database(dataframe, table_name)
                st.success(f"Data successfully saved to table '{table_name}' in the database.")
            else:
                st.error("Please enter a valid table name.")
        file_content = dataframe.to_string()


    # Add handling for other file types (csv, docx, xlsx) here
    
    # Now, file_content holds the text extracted from the file
    # You can use this text with OpenAI's API as needed
    
    # Example: Using file_check to summarize the content
    summary = file_check(file_content, openai_api_key)
    st.write("Summary:", summary)
