---
title: "Scraping song lyrics from Genius.com 🎶"
date: 2017-08-21
layout: post
tag:
- python
- tutorial
- music
- lyrics
- API
category: blog
author: john
description: Tutorial on scraping song lyrics from Genius.com
---

[Genius.com](https://www.genius.com) is a fun website. If you aren't familiar with it, Genius hosts a bunch of song lyrics and lets users highlight and annotate passages with interpretations, explanations, and references. Originally called RapGenius.com and devoted to lyrics from rap and hip-hop songs, the website now includes lyrics and annotations from all genres of music. You can figure out what "[Words are flowing out like endless rain into a paper cup](https://genius.com/3287551)" from *Across the Universe* really means, or what Noname was referring to when she said "[Moses wrote my name in gold and Kanye did the eulogy](https://genius.com/10185147)".

It's actually not too difficult to start pulling data from the Genius website. Genius is hip enough to provide a free application programming interface (API) that makes it easy for nerds to programmatically access song and artist data from their website. What the Genius API *doesn't* provide, however, is a way to download the lyrics themselves. With a little help from [Beautiful Soup](https://www.crummy.com/software/BeautifulSoup/) though, it's possible to grab the song lyrics without too much extra work.

If you'd like to follow along with the code described in this post, download my Python Jupyter notebook [here](/assets/code/scraping-genius-lyrics.ipynb). The culmination of this post is my LyricsGenius package, which is available on [GitHub](https://github.com/johnwmillr/LyricsGenius). The pacakge has evolved quite a bit from when I originally wrote this post, but it should still serve as a useful overview of the core functionality.

## tl;dr ##
I wrote a [Python package](https://pypi.org/project/lyricsgenius/) that wraps the Genius.com API and makes it easy to download song lyrics.

Use `pip` to install the package:

```
pip install lyricsgenius
```

Look at the [examples](https://github.com/johnwmillr/LyricsGenius#usage) on my GitHub page to get started.

## The Genius API ##
Before you can start making requests to the Genius API, you'll need to [sign up](http://genius.com/api-clients) for a (free) account that authorizes you to access their API. The authorization key you're assigned is then incorporated into each request you make to the API. If you're interested in all of the API details, check out the Genius [documentation](https://docs.genius.com).

The documentation is organized by the different sort of requests that can be made to the API. I'm most interested in *Search*, *Song*, and *Artist* requests, so I'll focus on those in this post. *Search* requests allow you to search Genius.com for any given string, just like you would in the search box on the website. The *Song* and *Artist* requests allow you to directly access an item from the Genius database by providing the item's corresponding API ID. After receiving a request, the database returns a JSON object containing search results, song names, etc.

### Making an API request ###
Each different type of API request is referred to as an endpoint, and follows its own [Uniform Resource Identifier](https://en.wikipedia.org/wiki/Web_API) (URI) format. Here are the formatted URIs for the *Song*, *Artist*, and *Search* endpoints:
  * *Song*:   ```https://api.genius.com/songs/[song_api_id]```
  * *Artist*: ```https://api.genius.com/artists/[artist_api_id]```
  * *Search*:  ```https://api.genius.com/search?q=[search_term]```

For example, if you wanted to search the database for songs about being stuck in an airport (like I currently am), you'd make the following API *Search* request:

```
https://api.genius.com/search?q=stuck%20%in%20%the%20airport
```

The ```%20``` values replace spaces in a URI. You'll need to use the ```quote()``` function from the ```urllib2``` Python library to format your search requests.

Here's how we might search for an artist using the Genius API:
```python
import requests
import urllib2

# Format a request URI for the Genius API
search_term = 'Andy Shauf'
_URL_API = "https://api.genius.com/"
_URL_SEARCH = "search?q="
querystring = _URL_API + _URL_SEARCH + urllib2.quote(search_term)
request = urllib2.Request(querystring)
request.add_header("Authorization", "Bearer " + client_access_token)
request.add_header("User-Agent", "")
```

Along with formatting the URI, we need to add headers to the request so that the database has a bit of info on the client (our Python program) making the request. You'll get your ```client_access_token``` after registering for an account on Genius. If you don't include these headers in your request you'll get either a 504 authorization error or 431 invalid request error in response.

Now that we've formatted the URI, we can make a request to the database.

```python
response = urllib2.urlopen(request, timeout=3)
json_obj = response.json()
```

We use the built-in `.json()` command to convert the raw response into a JSON object. We can access fields in the JSON object just like we would a normal Python dictionary:

```python
# The JSON object operates just like a normal Python dictionary
>>> json_obj.viewkeys()
dict_keys([u'meta', u'response'])
```

The *response* key contains a *hits* key which is a list of all search results. From there it's easy to grab the song title, album, etc.
```python
# List each key contained within a single search hit
>>> print(json_obj['response']['hits'][0]['result'].keys())
[u'song_art_image_thumbnail_url',
 u'api_path',
 u'stats',
 u'lyrics_state',
 u'title',
 u'pyongs_count',
 u'lyrics_owner_id',
 u'url',
 u'full_title',
 u'title_with_featured',
 u'header_image_thumbnail_url',
 u'path',
 u'primary_artist',
 u'annotation_count',
 u'id',
 u'header_image_url']
```

If you have an artist or song ID, you can access that entry directly by reformatting the request URI.

```python
# If you have an artist or song ID, you can access that entry
# directly by reformatting the request URI.
song_id = 82926
querystring = "https://api.genius.com/songs/" + str(song_id)  # Songs endpoint
request = urllib2.Request(querystring)
request.add_header("Authorization", "Bearer " + client_access_token)
request.add_header("User-Agent", "")
response = urllib2.urlopen(request, timeout=3)
raw = response.read()
json_obj = json.loads(raw)['response']['song']
```

```python
# Print the song title and artist name
>>> print((json_obj['title'], json_obj['primary_artist']['name']))
(u'All I Want', u'Joni Mitchell')
```

## Scraping song lyrics ##
As I mentioned above, Genius doesn't actually let you pull lyrics from their API directly. This isn't a big deal, because after finding a song's URI using the search function, we can use the ```BeautifulSoup``` library to scrape the page's HTML for song lyrics.

The actual code for scraping lyrics from a page isn't too complicated:
```python
from bs4 import BeautifulSoup
import re
URL = 'https://genius.com/Andy-shauf-the-magician-lyrics'
page = requests.get(URL)
html = BeautifulSoup(page.text, "html.parser") # Extract the page's HTML as a string

# Scrape the song lyrics from the HTML
lyrics = html.find("div", class_="lyrics").get_text()
```

```python
>>> print(lyrics[:150])
"[Verse 1]
Do you find
It gets a little easier each time you make it disappear?
Oh fools, the magician bends the rules
As the crowd watches his every..."
```

---

## *LyricsGenius* ##
I decided to write a Python wrapper for the Genius API to make it a bit easier to grab data from the database. The easiest way to get started with the package is to install it via [PyPI](https://pypi.org/project/lyricsgenius/) using `pip`:

`$pip install lyricsgenius`

If you'd prefer to clone and install the latest version of the repository from GitHub, follow these steps:
1. Clone this repo:
`$git clone https://github.com/johnwmillr/LyricsGenius.git`
2. Enter the new directory:
`$cd LyricsGenius`
3. Install using pip:
`$pip install .`


### Examples ###
Search for songs by a given artist

```python
>>> import lyricsgenius as genius
>>> api = genius.Genius()
>>> artist = api.search_artist('Andy Shauf', max_songs=3)
Searching for Andy Shauf...

Song 1: "Alexander All Alone"
Song 2: "Begin Again"
Song 3: "Comfortable With Silence"

Reached user-specified song limit (3).
Found 3 songs.

Done.
>>> print(artist)
('Andy Shauf', '3 songs')
```

Add a song to the artist object
```python
>>> song = api.search_song('Wendell Walker', artist.name)
Searching for "Wendell Walker" by Andy Shauf...
Done.
>>> artist.add_song(song)
>>> print(artist)
('Andy Shauf', '4 songs')
```

View song lyrics
```python
>>> print(artist.songs[0])
"Alexander All Alone" by Andy Shauf:
    Alexander all alone
    Smoking a cigarette
    The last pack hed ever buy
    At least thats what he said
    He st...
```

Save song lyrics to a .json file
```python
artist.save_lyrics(format='json')
```

Check out my post on a textual analysis of [country music](http://www.johnwmillr.com/trucks-and-beer/) for an example of one way to put this code to use.

Let me know what you think!
