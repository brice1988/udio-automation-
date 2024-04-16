# AUTOMATE UDIO 
# was working on 2024-04-13

# tldr; get your login cookie, use it to automate udio from python

import requests
import json
from time import sleep
import re 

##############

# HOW TO GET YOUR COOKIE
# (1) open a browser, login to your account on udio.com 
# (2) open your browser's dev console (possibly under View>Developer Tools>Console)
# (3) type in: document.cookie and hit enter to get your cookie.
# (4) paste your cookie here:
# (hint: it starts with "sb-api-auth-token")
cookie = ''

# USER PARAMETERS:::::
data = {
    'prompt': 'Mathcore',
    'lyricInput': 'peanutbutter',
    'samplerOptions': {
        'seed': -1,
        'bypass_prompt_optimization': True
    }
}

################


# Save file at url to local path
def download_mp3(url, save_path):
    try:
        response = requests.get(url, stream=True)
        response.raise_for_status()
        with open(save_path, 'wb') as file:
            for chunk in response.iter_content(chunk_size=8192):
                file.write(chunk)
        print(f"Download successful. File saved as '{save_path}'.")
    except requests.RequestException as e:
        print(f"An error occurred: {e}")

# Cleanup the prompt for saving filenames
def cleanup(text):
    # Pattern to match any non-alphanumeric character
    pattern = r'[^a-zA-Z0-9]'
    # Replace these characters with '_'
    replaced_text = re.sub(pattern, '_', text)
    return replaced_text

# Main process for downloading music from udio 
# cookie must be copied from your browser's document.cookie after you login to udio.com
# data has the music parameters (prompt, bypassing prompt optimization, seed, etc)
def udio(cookie, data):
    headers = {
        'accept': 'application/json, text/plain, */*',
        'accept-language': 'en-US,en;q=0.9',
        'content-type': 'application/json',
        'cookie': cookie,
        'dnt': '1',
        'origin': 'https://www.udio.com',
        'referer': 'https://www.udio.com/my-creations',
        'sec-ch-ua': '"Google Chrome";v="123", "Not:A-Brand";v="8", "Chromium";v="123"',
        'sec-ch-ua-mobile': '?0',
        'sec-ch-ua-platform': '"macOS"',
        'sec-fetch-dest': 'empty',
        'sec-fetch-mode': 'cors',
        'sec-fetch-site': 'same-origin',
        'sec-gpc': '1',
        'user-agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/123.0.0.0 Safari/537.36'
    }
    ## SEND THE USER REQUEST FOR MUSIC
    print("Prompt:", data["prompt"])
    print("Sending request to udio....")
    url = 'https://www.udio.com/api/generate-proxy'
    max_tries = 10
    timeout_seconds = 20
    for i in range(max_tries):
        response = requests.post(url, headers=headers, json=data)
        result = response.json()
        if "error" in result:
            print(result)
            print("..retrying in %s seconds, %s/%s" % (timeout_seconds, i+1, max_tries))
            sleep(timeout_seconds)
            continue
        elif "track_ids" in result:
            break
    track_ids = result["track_ids"]
    print("Request successful")
    print("Track_ids:")
    print(track_ids)

    ## WAIT FOR IT TO FINISH
    print("Generating.. wait 60 seconds")
    sleep(60)
    max_tries = 10
    timeout_seconds = 10
    songs_downloaded = []
    for i in range(max_tries):
        url = "https://www.udio.com/api/songs?songIds=%s,%s" % (track_ids[0], track_ids[1])
        response = requests.get(url, headers=headers)
        result = response.json()
        songs = result["songs"]
        #print(songs)

        for song in songs:
            if "song_path" in song and song["song_path"]:
                if song["id"] not in songs_downloaded:
                    print("Song completed!", track_ids[0])
                    song_path = song["song_path"]
                    prompt = song["prompt"]
                    prompt = cleanup(prompt)
                    id = song["id"]
                    filename = "%s__%s.mp3" % (prompt, id)
                    print("Downloading..")
                    print(song_path, filename)
                    download_mp3(song_path, filename)
                    print("Song downloaded!")
                    songs_downloaded.append(id)
        if len(songs_downloaded) == 2:
            break
        else:
            print("waiting... %s/%s retries" % (i+1, max_tries))
            sleep(timeout_seconds)
    print("All done.")   

###########
