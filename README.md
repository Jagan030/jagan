import streamlit as st
from googleapiclient.discovery import build
import pymongo as pg

from datetime import datetime
import time

import pandas as pd
from sqlalchemy import create_engine
import mysql.connector


st.set_page_config(page_title="Youtube Data", page_icon=":alien:", layout="wide", initial_sidebar_state="auto", menu_items=None)

st.title(":red[YouTube Data] :blue[Harvesting] and :blue[Warehousing] 📡")

#Connecting with Youtube API

API_KEY='AIzaSyAm2s8Pyp0rG3L9wHyM7z5n0MvM3b2bWgA'
youtube = build('youtube', 'v3', developerKey=API_KEY)

st.subheader(":orange[Fetching Data and Uploading to MongoDB Database] ⌛")

#Function to get the channel_details:

@st.cache_data
def channel_statistics(_youtube,channel_ids):
    all_data = []
    request = youtube.channels().list(
    part="snippet,contentDetails,statistics",
    id=channel_ids)
    response = request.execute()

    for i in range(len(response["items"])):
        data = dict(channel_id = response["items"][i]["id"],
                    channel_name = response["items"][i]["snippet"]["title"],
                    channel_views = response["items"][i]["statistics"]["viewCount"],
                    subscriber_count = response["items"][i]["statistics"]["subscriberCount"],
                    total_videos = response["items"][i]["statistics"]["videoCount"],
                    playlist_id = response["items"][i]["contentDetails"]["relatedPlaylists"]["uploads"])
        all_data.append(data)
    return all_data


#Function to get playlist data

@st.cache_data
def get_playlist_data(df):
    playlist_ids = []
     
    for i in df["playlist_id"]:
        playlist_ids.append(i)

    return playlist_ids


#Function to get video ids:

@st.cache_data
def get_video_ids(_youtube,playlist_id_data):
    video_id = []

    for i in playlist_id_data:
        next_page_token = None
        more_pages = True

        while more_pages:
            request = youtube.playlistItems().list(
                        part = 'contentDetails',
                        playlistId = i,
                        maxResults = 50,
                        pageToken = next_page_token)
            response = request.execute()
            
            for j in response["items"]:
                video_id.append(j["contentDetails"]["videoId"])
        
            next_page_token = response.get("nextPageToken")
            if next_page_token is None:
                more_pages = False
    return video_id
        

#Function to get Video details:

@st.cache_data
def get_video_details(_youtube,video_id):

    all_video_stats = []

    for i in range(0,len(video_id),50):
        
        request = youtube.videos().list(
                  part="snippet,contentDetails,statistics",
                  id = ",".join(video_id[i:i+50]))
        response = request.execute()
        
        for video in response["items"]:
            published_dates = video["snippet"]["publishedAt"]
            parsed_dates = datetime.strptime(published_dates,'%Y-%m-%dT%H:%M:%SZ')
            format_date = parsed_dates.strftime('%Y-%m-%d')

            videos = dict(video_id = video["id"],
                          channel_id = video["snippet"]["channelId"],
                         video_name = video["snippet"]["title"],
                         published_date = format_date ,
                         view_count = video["statistics"].get("viewCount",0),
                         like_count = video["statistics"].get("likeCount",0),
                         comment_count= video["statistics"].get("commentCount",0),
                         duration = video["contentDetails"]["duration"])
            all_video_stats.append(videos)

    return (all_video_stats)

#Function to get comment details

@st.cache_data
def get_comments(_youtube,video_ids):
    comments_data= []
    try:
        next_page_token = None
        for i in video_ids:
            while True:
                request = youtube.commentThreads().list(
                    part = "snippet,replies",
                    videoId = i,
                    textFormat="plainText",
                    maxResults = 100,
                    pageToken=next_page_token)
                response = request.execute()

                for item in response["items"]:
                    published_date= item["snippet"]["topLevelComment"]["snippet"]["publishedAt"]
                    parsed_dates = datetime.strptime(published_date,'%Y-%m-%dT%H:%M:%SZ')
                    format_date = parsed_dates.strftime('%Y-%m-%d')
                    

                    comments = dict(comment_id = item["id"],
                                    video_id = item["snippet"]["videoId"],
                                    comment_text = item["snippet"]["topLevelComment"]["snippet"]["textDisplay"],
                                    comment_author = item["snippet"]["topLevelComment"]["snippet"]["authorDisplayName"],
                                    comment_published_date = format_date)
                    comments_data.append(comments) 
                
                next_page_token = response.get('nextPageToken')
                if next_page_token is None:
                    break       
    except Exception as e:
        print("An error occured",str(e))          
            
    return comments_data
  

#User Input:

channel_ids = st.text_input("Enter the channel Id")
channel_list = [channel_ids]

submit = st.button("Fetch Channel details and Upload to MongoDB Database")


# MongoDB Connection
jagan= pg.MongoClient("mongodb+srv://jagan:jagan1998@jagan.hg4xtbj.mongodb.net/?retryWrites=true&w=majority")

#Creating database
db  = jagan["Youtube_Database"]

#Creating Collection:
col1 = db["channel_data"]
col2 = db["video_data"]
col3 = db["comment_data"]


if submit:
    
    if channel_ids:
        channel_details = channel_statistics(youtube,channel_ids)
        df = pd.DataFrame(channel_details) 
        playlist_id_data = get_playlist_data(df)
        video_id = get_video_ids(youtube,playlist_id_data)
        video_details = get_video_details(youtube,video_id)
        get_comment_data = get_comments(youtube,video_id)
        

        with st.spinner('Please wait '):
            time.sleep(5)
            st.success('Done!, Data Fetched Successfully')
            

            if channel_details:
            #Insert the data : 1
                col1.insert_many(channel_details) 
            if video_details:
            #Insert the data : 2
                col2.insert_many(video_details)
            if get_comment_data:
            #Insert the data : 3
                col3.insert_many(get_comment_data)

        with st.spinner('Please wait '):
            time.sleep(5)
            st.success('Done!, Data Uploaded Successfully')
            st.snow()

#Fetching data from MongoDB:

#Function to select channel names for user input from MondoDB

def channel_names():   
    ch_name = []
    for i in db.channel_data.find():
        ch_name.append(i['channel_name'])
    return ch_name

st.subheader(":orange[Inserting Data into MySQL for further Data Analysis] ⌛")
   
user_input =st.multiselect("Select the channel to be inserted into MySQL Tables",options = channel_names())

submit1 = st.button("Upload data into MySQL")

#Fetching Channel details:

if submit1:

    with st.spinner('Please wait '):
        
        def get_channel_details(user_input):
            query = {"channel_name":{"$in":list(user_input)}}
            projection = {"_id":0,"channel_id":1,"channel_name":1,"channel_views":1,"subscriber_count":1,"total_videos":1,"playlist_id":1}
            x = col1.find(query,projection)
            channel_table = pd.DataFrame(list(x))
            return channel_table

        channel_data = get_channel_details(user_input)
        print(channel_data)
        
    
        #Fetching Video details:
        def get_video_details(channel_list):
            query = {"channel_id":{"$in":channel_list}}
            projection = {"_id":0,"video_id":1,"channel_id":1,"video_name":1,"published_date":1,"view_count":1,"like_count":1,"comment_count":1,"duration":1}
            x = col2.find(query,projection)
            video_table = pd.DataFrame(list(x))
            return video_table

        video_data = get_video_details(channel_list)
        print(video_data)
    
        #Fetching Comment details:
        def get_comment_details(video_ids):
            query = {"video_id":{"$in":video_ids}}
            projection = {"_id":0,"comment_id":1,"video_id":1,"comment_text":1,"comment_author":1,"comment_published_date":1}
            x = col3.find(query,projection)
            comment_table = pd.DataFrame(list(x))
            return comment_table

        # #Fetch video_ids from mongoDb

        video_ids = video_data["video_id"].to_list()
       
        
        
        comment_data = get_comment_details(video_ids)
        st.write(comment_data)
        
        jagan.close()

    #MySQL Database Connection:

        connection = mysql.connector.connect(
            host = "localhost",
            port=3306,
            user = "root",
            password = "Jagan*1998",
            database = "youtube_data")

        cursor = connection.cursor()


        #Creating an SQLAlchemy engine to connect to the database:
        engine = create_engine('mysql+mysqlconnector://root:Jagan*1998@localhost/youtube_data')

    # Inserting Channel data into the table using try and except method:
        try:
            # Attempt to insert the data
            channel_data.to_sql('channel_data', con=engine, if_exists='append', index=False, method='multi')
            print("Data inserted successfully")
        except Exception as e:
            if 'Duplicate entry' in str(e):
                print("Duplicate data found. Ignoring duplicate entries.")
            else:
                print("An error occurred:", e)


    # Inserting Video data into the table using try and except method:

        try:
            # Attempt to insert the data
            video_data.to_sql('video_data', con=engine, if_exists='append', index=False, method='multi')
            print("Data inserted successfully")
        except Exception as e: 
            if 'Duplicate entry' in str(e):
                print("Duplicate data found. Ignoring duplicate entries.")
            else:
                print("An error occurred:", e)
        st.success("Data Uploaded Successfully")

        engine.dispose()

        # Inserting comment data into the table using try and except method:

        try:
            # Attempt to insert the data
            comment_data.to_sql('comment_data', con=engine, if_exists='append', index=False, method='multi')
            print("Data inserted successfully")
        except Exception as e: 
            if 'Duplicate entry' in str(e):
                print("Duplicate data found. Ignoring duplicate entries.")
            else:
                print("An error occurred:", e)
        st.success("Data Uploaded Successfully")

        engine.dispose()


st.subheader(":orange[Select any questions to get Insights ?]")

#MySQL Database Connection:

connection = mysql.connector.connect(
        host = "localhost",
        port=3306,
        user = "root",
        password = "Jagan*1998",
        database = "youtube_data")

cursor = connection.cursor()

questions = st.selectbox("Select any questions given below:",
['Click the question that you would like to query',
'1. What are the names of all the videos and their corresponding channels?',
'2. Which channels have the most number of videos, and how many videos do they have?',
'3. What are the top 10 most viewed videos and their respective channels?',
'4. How many comments were made on each video, and what are their corresponding video names?',
'5. Which videos have the highest number of likes, and what are their corresponding channel names?',
'6. What is the total number of likes for each video, and what are their corresponding video names?',
'7. What is the total number of views for each channel, and what are their corresponding channel names?',
'8. What are the names of all the channels that have published videos in the year 2022?',
'9. Which videos have the highest number of comments, and what are their corresponding channel names?'])


# Queries to be stored in Variables:
 
if questions == '1. What are the names of all the videos and their corresponding channels?':
    query1 = "select channel_name as Channel_name ,video_name as Video_names from channel_data c join video_data v on c.channel_id = v.channel_id;"
    cursor.execute(query1)

#Storing the results in Pandas Dataframe:
    result = cursor.fetchall()
    table1 = pd.DataFrame(result,columns = cursor.column_names)
    st.table(table1)

elif questions == '2. Which channels have the most number of videos, and how many videos do they have?':
    query2 = "select channel_name,count(video_name) as Most_Number_of_Videos from video_data v join channel_data c on c.channel_id = v.channel_id group by channel_name order by count(video_name) desc;"
    cursor.execute(query2)
    result1 = cursor.fetchall()
    table2 = pd.DataFrame(result1,columns =cursor.column_names)
    st.table(table2)
    st.bar_chart(table2.set_index("channel_name"))

elif questions == '3. What are the top 10 most viewed videos and their respective channels?':
    query3 = "select channel_name as Channel_name,video_name as Video_name,view_count as Top_10_Viewed_Videos from channel_data c join video_data v on c.channel_id = v.channel_id order by view_count desc limit 10;"
    cursor.execute(query3)
    result2 = cursor.fetchall()
    table3 = pd.DataFrame(result2,columns=cursor.column_names)
    st.table(table3)
   

elif questions == '4. How many comments were made on each video, and what are their corresponding video names?':
    query4 = "select channel_name as Channel_name, video_name as Video_name,comment_count as Comments_Count from video_data v join channel_data c on c.channel_id = v.channel_id order by comment_count desc;"
    cursor.execute(query4)
    result3 = cursor.fetchall()
    table4 = pd.DataFrame(result3,columns=cursor.column_names)
    st.table(table4)
    

elif questions == '5. Which videos have the highest number of likes, and what are their corresponding channel names?':
    query5 = "select channel_name as Channel_name,video_name as Video_name,like_count as Number_of_likes from video_data v join channel_data c on c.channel_id = v.channel_id order by like_count desc;"
    cursor.execute(query5)
    result4 = cursor.fetchall()
    table5 = pd.DataFrame(result4,columns=cursor.column_names)
    st.table(table5)
    

elif questions == '6. What is the total number of likes for each video, and what are their corresponding video names?':
    query6 = "select channel_name as Channel_name,video_name as Video_name,like_count as Like_count from video_data v join channel_data c on c.channel_id = v.channel_id order by like_count desc;"
    cursor.execute(query6)
    result5 = cursor.fetchall()
    table6 = pd.DataFrame(result5,columns=cursor.column_names)
    st.table(table6)

elif questions == '7. What is the total number of views for each channel, and what are their corresponding channel names?':
    query7 = "select channel_name as Channel_name,channel_views as Total_No_of_views from video_data v join channel_data c on c.channel_id = v.channel_id group by c.channel_id,v.channel_id order by channel_views desc;"
    cursor.execute(query7)
    result6 = cursor.fetchall()
    table7 = pd.DataFrame(result6,columns=cursor.column_names)
    st.table(table7)
    st.bar_chart(table7.set_index("Channel_name"))

elif questions == '8. What are the names of all the channels that have published videos in the year 2022?':

    query8 = "select distinct c.channel_name as Channel_name,year(published_date) as Published_year from channel_data c join video_data v on c.channel_id = v.channel_id where year(published_date) = 2022;"
    cursor.execute(query8)
    result7 = cursor.fetchall()
    table8 = pd.DataFrame(result7,columns=cursor.column_names)
    st.table(table8)


elif questions =='9. Which videos have the highest number of comments, and what are their corresponding channel names?':
    query9 = "select channel_name as Channel_name,video_name as Video_name,comment_count as Highest_No_of_comments from channel_data c join video_data v on c.channel_id = v.channel_id order by comment_count desc limit 10;"
    cursor.execute(query9)
    result8 = cursor.fetchall()
    table9 = pd.DataFrame(result8,columns=cursor.column_names)
    st.table(table9)

#Closing the Connection:
cursor.close()
connection.close()


