# Spotify ETL
This project focuses on performing ETL on Spotify data

The project leverages Python, PostgreSQL, Docker and Apache Airflow to extract data from Spotify APIs, transform data and load it onto PostreSQL database, as well as to orchestrate the pipeline.

![icons](https://github.com/SimonPoparda/SpotifyETL/assets/108056198/b83296f1-1168-4537-8926-f38d0271493e)


<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#data-source">Data Source</a>
      <a href="#execution">Execution</a>
      <ul>
        <li><a href="#extracting-data-from-apis">Extracting Data from APIs</a></li>
      </ul>
      <ul>
        <li><a href="#transform">Transform</a></li>
      </ul>
      <ul>
        <li><a href="#load">Load</a></li>
      </ul>
    </li>
      <a href="#summary">Summary</a>
    </li>
  </ol>

  
</details> 



-----------------------------------------------------------------------------------------

## Data Source
The data used for this project was my private Spotify user data. I wanted to extract all of my playlists and their items, so I can run analytics on them in the future.

-----------------------------------------------------------------------------------------

## Execution
### Extracting Data from APIs

To get extract my data I opened Spotify documentation


![apis](https://github.com/SimonPoparda/SpotifyETL/assets/108056198/5bcad962-fbb7-4f2b-8590-4f6d7b5f72eb)

I had to create an app in Spotify UI


![apis2](https://github.com/SimonPoparda/SpotifyETL/assets/108056198/ed6b853b-919b-4222-ae7b-48c5ea142ed1)

I also had to get a authentication token. In order to do so I wrote some code basing on Spotify docs

```py
import requests
import pandas as pd
import psycopg2


# Settings provided by Spotify
token_endpoint = "https://accounts.spotify.com/api/token"
client_id = <my_id>
client_secret = <my_secrets>
username = <my_username>

data = {
    'client_id': client_id,
    'client_secret': client_secret,
    'grant_type': 'client_credentials'
}

headers = {
    'Content-Type': 'application/x-www-form-urlencoded'
}

# POST request
response = requests.post(token_endpoint, data=data, headers=headers)

# Check reponse status
if response.status_code == 200:
    # Success 
    access_token = response.json().get('access_token')
    print("Access token:", access_token)
else:
    # Error
    print("Error:", response.status_code, response.reason)

```

After getting my token I requested information about my playlists, which came in JSON format

```py
url = f'https://api.spotify.com/v1/users/{username}/playlists'

# header with access token
headers = {
    'Authorization': 'Bearer ' + access_token
}

# GET request
response = requests.get(url, headers=headers)

# Check reponse status
if response.status_code == 200:
    # Success 
    playlists = response.json()
    print(response.json())
else:
    # Error
    print("Error:", response.status_code, response.reason)
```

I put all of my playlist's names into a dataframe, as well as put ids into a list
```py
playlists = pd.json_normalize(playlists['items'])
playlists["name"].to_frame()

playlists_list = playlists['id'].to_list()
```

I also extracted playlist's URLs into a list and using that list I imported items from my playlists
```py
playlists_urls = []

for playlist_id in playlists_list:
    url = f'https://api.spotify.com/v1/playlists/{playlist_id}/tracks'
    playlists_urls.append(url)

headers = {
    'Authorization': 'Bearer ' + access_token
}

playlists_jsons = []

for url in playlists_urls:
    # GET request
    response = requests.get(url, headers=headers)

    # Check response status
    if response.status_code == 200:
        # Success
        playlists_jsons.append(response.json())
    else:
        # Error
        print("Error:", response.status_code, response.reason)
```

I also extracted them from JSON to dataframe
```py
headers = {
    'Authorization': 'Bearer ' + access_token
}

playlists_jsons = []

for url in playlists_urls:
    # GET request
    response = requests.get(url, headers=headers)

    # Check response status
    if response.status_code == 200:
        # Success
        playlists_jsons.append(response.json())
    else:
        # Error
        print("Error:", response.status_code, response.reason)
```

After that, I decided to my data into dictionary with playlist names as keys and dataframes as values
```py
playlist_names = playlists['name']

#removing unnecessary characters
for i in range(len(playlist_names)):
    playlist_names[i] = playlist_names[i].replace(" ", "_")
    playlist_names[i] = playlist_names[i].replace("'", "")
    playlist_names[i] = playlist_names[i].replace(",", "")
    playlist_names[i] = playlist_names[i].replace("?", "")
    playlist_names[i] = playlist_names[i].replace("&", "")

song_dict_fin = {}
 
for i in range(len(playlist_names)):
    song_dict_fin[playlist_names[i]] = playlists_items[i]
```

### Transform 
```py
def check_if_valid_data(data):
    # Check if df is empty
    if data.empty:
        print("No songs downloaded. Finishing execution")
        return False

    # Check for nulls
    if data.isnull().any(axis=1).any():
        print("NULL values in df.")
        data = data.dropna()
    else:
        print("No NULL values.")
    
    return data 

# convert duration_ms to Hour-Minute-Second format
def convert_time_ms(data):
    data['duration_time'] = pd.to_datetime(data['duration_ms'] / 1000, unit='s').dt.strftime('%H:%M:%S')
    return data

for i in playlist_names:
    song_dict_fin[i] = check_if_valid_data(song_dict_fin[i])
    song_dict_fin[i] = convert_time_ms(song_dict_fin[i])
    song_dict_fin[i] = song_dict_fin[i].drop(columns=['duration_ms']) #drop duration_ms column
```

### Load
I established connection with my PostgreSQL database
```py
# Defining database connection parameters
db_params = {
    "host": ,
    "database": ,
    "user": ,
    "password": ,
    "port": 
}

try:
    conn = psycopg2.connect(**db_params)
except psycopg2.Error as e:
    print("Error: Could not make connection to the Postgres database")
    print(e)
```

```py
#create a database
try:
    cur.execute("CREATE DATABASE Spotify_ETL;")
except psycopg2.Error as e:
    print("Error: ")
    print(e)

cur.close()
conn.close()
```

```py
# Defining connection with new database
db_params = {
    "host": "localhost",
    "database": "spotify_etl",
    "user": "postgres",
    "password": "kielbasa123",
    "port": "5432"
}


try:
    conn = psycopg2.connect(**db_params)
except psycopg2.Error as e:
    print("Error: Could not make connection to the Postgres database")
    print(e)
```

```py
try:
    cur = conn.cursor()
except psycopg2.Error as e:
    print("Error: Could not get cursor to the Database")
    print(e)

conn.autocommit = True
```

```py
#creating tables

for i in playlist_names:
    sql_query = f"""
    CREATE TABLE IF NOT EXISTS {i}(
        song_name VARCHAR(200),
        artist_name VARCHAR(200),
        duration_time TIME
    )
    """

    cur.execute(sql_query)
```

```py
#inserting the data

for i in playlist_names:
    sql_query = f"""
    INSERT INTO {i}(
        song_name,
        artist_name,
        duration_time
    )
    VALUES (%s, %s, %s)
    """

    for index, row in song_dict_fin[i].iterrows():
        cur.execute(sql_query, list(row))
```

### Orchestration
I created two docker files to initialize my image, as well as container

```dockerfile
FROM apache/airflow:latest

USER root

RUN apt-get update && \
    apt-get -y install git && \
    apt-get clean
USER airflow
```

```yml
version: "3"

services:
  spotifyetl:
    image: spotifyetl:latest
    volumes:
      - ./airflow:/opt/airflow
    ports:
      - "8080:8080"
    command: airflow standalone
```

![docker](https://github.com/SimonPoparda/SpotifyETL/assets/108056198/327c0920-5cb0-4f5f-81bc-9d47102b07d8)


![docker2](https://github.com/SimonPoparda/SpotifyETL/assets/108056198/22b608d6-0525-44c5-a840-22f1ac295e0d)


New directory called 'airflow' appeared and I also created new folder called dags


![docker3](https://github.com/SimonPoparda/SpotifyETL/assets/108056198/ad89bacc-b7bb-489d-b212-d4941e51cf5f)


Next, I opened AirFlow UI using Docker


![docker4](https://github.com/SimonPoparda/SpotifyETL/assets/108056198/abf506e5-35dd-4435-a6ea-30b17dc82e7a)


![image](https://github.com/SimonPoparda/SpotifyETL/assets/108056198/7d54e81e-2695-4412-89cf-1d7376b23e04)


For now there was no DAGs available, so I created new one inside my dags folder


![airflow3](https://github.com/SimonPoparda/SpotifyETL/assets/108056198/351e217a-2a1c-41f4-b831-dab54629bbbe)


![airflow2](https://github.com/SimonPoparda/SpotifyETL/assets/108056198/0e6f7655-e693-491e-a59c-695eca307faf)


And it resulted in the following pipeline, which I can now run daily to get my private Spotify data that I can analyze:


![airflow](https://github.com/SimonPoparda/SpotifyETL/assets/108056198/80b95dbf-55e3-47b2-8764-b581e5a11b67)


-----------------------------------------------------------------------------------------
## Summary
The Spotify ETL project is focused on extracting, transforming, and loading (ETL) data from Spotify APIs into a PostgreSQL database using Python, Docker, and Apache Airflow.

Project begins by obtaining an access token from Spotify's API. This token is then used to retrieve information about the playlists, including the names and IDs of each playlist. Subsequently, the project extracts the tracks from each playlist and transforms the data into a structured format.

The transformation process involves checking for data validity, converting the duration of songs from milliseconds to a more human-readable format (hours:minutes:seconds), and removing unnecessary columns. Finally, the transformed data is loaded into a PostgreSQL database.

The orchestration of the ETL pipeline is managed using Apache Airflow, which allows for scheduling and monitoring of tasks. Docker is used to containerize the Airflow environment, ensuring consistency and portability across different systems.

## Authors

- [@Szymon Poparda](https://www.linkedin.com/in/szymon-poparda-02b96a248/)






