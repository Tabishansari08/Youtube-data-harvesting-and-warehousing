Here's a basic workflow and execution of the entire project:

Workflow and Execution:
Initial Setup:

Install the required libraries by running pip install streamlit google-api-python-client pymongo.
MongoDB Setup:

Ensure you have a MongoDB cluster set up with the appropriate credentials.
Replace "mongodb+srv://tabiansari:Passkey123@guviproject.eeauktn.mongodb.net/?retryWrites=true&w=majority" with your MongoDB connection string in the code.
SQLite Database Setup:

The code initializes an SQLite database named youtube_data.db.
SQLite does not require any additional setup as it's a self-contained database.
Running the Application:

Run the Python script containing the code using streamlit run your_script.py.
This will start a Streamlit web application on your local machine.
User Input:

Enter your YouTube Data API Key in the text input field labeled "Enter your YouTube Data API Key."
Enter the YouTube Channel IDs you want to fetch data from, separated by commas, in the text input field labeled "Enter YouTube Channel IDs (comma-separated)."
Data Retrieval:

Click the "Submit" button to retrieve data from the specified YouTube channels.
The application will display channel information, videos, and comments for each channel.
Data Storage:

You can click the "Store date to MongoDB" button to push the collected data to your MongoDB cluster.
Alternatively, you can click the "Migrate Data from MongoDB to SQL" button to transfer data from MongoDB to the SQLite database.
Data Analysis:

After collecting and storing the data, you can explore various questions by selecting them from the dropdown list.
Click the "Get Answer" button to see the answers to the selected question.
The answers will be displayed as tables within the Streamlit application.
Database Closure:

The SQLite database connection is closed after retrieving answers to the questions from the dropdown.
Exploring More Questions:

You can continue exploring other questions by selecting different options from the dropdown and clicking the "Get Answer" button.
Application Termination:

Close the Streamlit application when you're finished by stopping the script.
Note:
The workflow assumes that you have already obtained a valid YouTube Data API Key.
Ensure you have proper internet connectivity as the application makes requests to the YouTube API to fetch data.
You can customize the questions in the questions list to suit your analysis needs.
Remember to replace the MongoDB connection string and API key with your own credentials and keys for security reasons.
