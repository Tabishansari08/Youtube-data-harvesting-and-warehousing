import streamlit as st
from googleapiclient.discovery import build
import pymongo
import sqlite3

# Initialize MongoDB client
mongo_client = pymongo.MongoClient("mongodb+srv://tabiansari:Passkey123@guviproject.eeauktn.mongodb.net/?retryWrites=true&w=majority")
db = mongo_client["youtube_data_lake"]

# Initialize SQLite database
sqlite_conn = sqlite3.connect('youtube_data.db')
sqlite_cursor = sqlite_conn.cursor()

# Create tables in SQLite database
sqlite_cursor.execute('''
    CREATE TABLE IF NOT EXISTS channel_details (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        channel_name TEXT,
        total_subscribers INTEGER,
        total_videos INTEGER
    )
''')

sqlite_cursor.execute('''
    CREATE TABLE IF NOT EXISTS video_details (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        channel_id TEXT,
        video_title TEXT,
        video_likes INTEGER
    )
''')

sqlite_cursor.execute('''
    CREATE TABLE IF NOT EXISTS comments (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        video_id TEXT,
        comment_author TEXT,
        comment_text TEXT
    )
''')

st.title('YouTube Date Harvesting and Warehousing')

# Input field for entering the YouTube Data API key
api_key = st.text_input('Enter your YouTube Data API Key:')
channel_ids = st.text_input('Enter YouTube Channel IDs (comma-separated):')

# Create a submit button to retrieve data
if st.button('Submit'):
    if not api_key:
        st.warning('Please enter your API Key to continue.')
    elif not channel_ids:
        st.warning('Please enter at least one YouTube Channel ID.')
    else:
        channel_ids = channel_ids.split(',')

        # Build the YouTube API client with the user-provided API key
        youtube = build('youtube', 'v3', developerKey=api_key)

        for channel_id in channel_ids:
            st.subheader(f'Channel ID: {channel_id}')
            channel_info = youtube.channels().list(part='snippet,statistics', id=channel_id).execute()

            if channel_info.get('items'):
                channel_data = channel_info['items'][0]
                channel_name = channel_data['snippet']['title']
                total_subscribers = channel_data['statistics']['subscriberCount']
                total_videos = channel_data['statistics']['videoCount']

                st.write(f'Channel Name: {channel_name}')
                st.write(f'Total Subscribers: {total_subscribers}')
                st.write(f'Total Videos: {total_videos}')

                # Fetch and display video details and comments
                videos = youtube.search().list(part='id', channelId=channel_id, type='video').execute()

                if videos.get('items'):
                    for video in videos['items']:
                        video_id = video['id']['videoId']
                        video_info = youtube.videos().list(part='snippet,statistics', id=video_id).execute()
                        if video_info.get('items'):
                            video_data = video_info['items'][0]['snippet']
                            video_statistics = video_info['items'][0]['statistics']
                            video_title = video_data['title']
                            video_likes = video_statistics['likeCount']

                            st.write(f'Video Title: {video_title}')
                            st.write(f'Likes: {video_likes}')

                            # Fetch and display comments for the video
                            comments = youtube.commentThreads().list(
                                part='snippet',
                                videoId=video_id,
                                textFormat='plainText'
                            ).execute()

                            if comments.get('items'):
                                st.write('Comments:')
                                for comment in comments['items']:
                                    comment_author = comment['snippet']['topLevelComment']['snippet']['authorDisplayName']
                                    comment_text = comment['snippet']['topLevelComment']['snippet']['textDisplay']
                                    st.write(f'Author: {comment_author}')
                                    st.write(f'Comment: {comment_text}')

                                    # Store data in MongoDB
                                    db.channel_details.insert_one({
                                        'channel_name': channel_name,
                                        'total_subscribers': total_subscribers,
                                        'total_videos': total_videos
                                    })

                                    db.video_details.insert_one({
                                        'channel_id': channel_id,
                                        'video_title': video_title,
                                        'video_likes': video_likes
                                    })

                                    db.comments.insert_one({
                                        'video_id': video_id,
                                        'comment_author': comment_author,
                                        'comment_text': comment_text
                                    })

                                    # Store data in SQLite
                                    sqlite_cursor.execute('''
                                        INSERT INTO channel_details (channel_name, total_subscribers, total_videos)
                                        VALUES (?, ?, ?)
                                    ''', (channel_name, total_subscribers, total_videos))

                                    sqlite_cursor.execute('''
                                        INSERT INTO video_details (channel_id, video_title, video_likes)
                                        VALUES (?, ?, ?)
                                    ''', (channel_id, video_title, video_likes))

                                    sqlite_cursor.execute('''
                                        INSERT INTO comments (video_id, comment_author, comment_text)
                                        VALUES (?, ?, ?)
                                    ''', (video_id, comment_author, comment_text))

                            else:
                                st.write('No comments found for this video.')
                else:
                    st.write('No videos found for this channel.')

# Create a button to push data to MongoDB
if st.button('Store date to MongoDB'):
    st.info('Data has been pushed to MongoDB.')

# Create a button to transfer data from MongoDB to SQLite
if st.button('Migrate Data from MongoDB to SQL'):
    # Retrieve data from MongoDB and insert it into SQLite
    for channel in db.channel_details.find():
        sqlite_cursor.execute('''
            INSERT INTO channel_details (channel_name, total_subscribers, total_videos)
            VALUES (?, ?, ?)
        ''', (channel['channel_name'], channel['total_subscribers'], channel['total_videos']))

    for video in db.video_details.find():
        sqlite_cursor.execute('''
            INSERT INTO video_details (channel_id, video_title, video_likes)
            VALUES (?, ?, ?)
        ''', (video['channel_id'], video['video_title'], video['video_likes']))

    for comment in db.comments.find():
        sqlite_cursor.execute('''
            INSERT INTO comments (video_id, comment_author, comment_text)
            VALUES (?, ?, ?)
        ''', (comment['video_id'], comment['comment_author'], comment['comment_text']))

    sqlite_conn.commit()
    st.info('Data has been transferred from MongoDB to SQLite.')

# Display tables after pushing data to SQLite
if st.button('Display Tables'):
    # Commit changes to the database
    sqlite_conn.commit()

    # Display tables
    st.subheader('Channel Details Table')
    channel_details_query = sqlite_cursor.execute('''
        SELECT DISTINCT channel_name, total_subscribers, total_videos
        FROM channel_details
    ''').fetchall()
    st.table(channel_details_query)

    # Display distinct video details
    st.subheader('Video Details Table')
    video_details_query = sqlite_cursor.execute('''
        SELECT DISTINCT channel_id, video_title, video_likes
        FROM video_details
    ''').fetchall()
    st.table(video_details_query)

    # Display distinct comments
    st.subheader('Comments Table')
    comments_query = sqlite_cursor.execute('''
        SELECT DISTINCT video_id, comment_author, comment_text
        FROM comments
    ''').fetchall()
    st.table(comments_query)

# Close SQLite connection
sqlite_conn.close()

# Define a list of available questions
questions = [
    "What are the names of all the videos and their corresponding channels?",
    "How many comments were made on each video, and what are their corresponding video names?",
    "Which channels have the most number of videos, and how many videos do they have?",
    "What are the top 10 most viewed videos and their respective channels?",
    "Which videos have the highest number of likes, and what are their corresponding channel names?",
    "What is the total number of likes and dislikes for each video, and what are their corresponding video names?",
    "What is the total number of views for each channel, and what are their corresponding channel names?",
    "What are the names of all the channels that have published videos in the year 2022?",
    "What is the average duration of all videos in each channel, and what are their corresponding channel names?",
    "Which videos have the highest number of comments, and what are their corresponding channel names?",
]

# Create a dropdown with the questions that have answers
selected_question = st.selectbox("Select a question:", questions)

# Define functions to answer each question
def answer_question(selected_question, sqlite_cursor):
    if selected_question == "What are the names of all the videos and their corresponding channels?":
        # Display video names and corresponding channel names
        video_details_query = sqlite_cursor.execute('''
            SELECT DISTINCT c.channel_name, v.video_title
            FROM channel_details c
            JOIN video_details v ON c.id = v.channel_id
        ''').fetchall()
        st.table(video_details_query)

    elif selected_question == "How many comments were made on each video, and what are their corresponding video names?":
        # Display video names and the number of comments for each video
        comments_query = sqlite_cursor.execute('''
            SELECT DISTINCT v.video_title, COUNT(c.id) AS comment_count
            FROM video_details v
            LEFT JOIN comments c ON v.id = c.video_id
            GROUP BY v.video_title
        ''').fetchall()
        st.table(comments_query)

# Call the function to answer the selected question
if st.button("Get Answer"):
    answer_question(selected_question, sqlite_cursor)



