---
layout: post
title: เมื่อทษช.เปิดเผยตัวละครลับเป็นแคนดิเดตนายกฯ (Codes)
subtitle: วิเคราะห์สถานการณ์ทางการเมืองด้วยข้อมูลจาก Twitter
gh-repo: naponjatusripitak/twitter_thai
gh-badge: [star, fork, follow]
tags: [Python, Twitter, Politics]
---
### ขั้นตอนแรก: Scraping data from Twitter
ขั้นตอนนี้ได้รับแรงบันดาลใจจาก https://galeascience.wordpress.com/2016/03/18/collecting-twitter-data-with-python/ โดยใช้ Tweepy ซึ่งเป็น library ใน python ในการดึงข้อมูลจาก Twitter API

ก่อนอื่นต้อง import module ต่างๆ ที่ใช้ในการรวบรวมข้อมูล และในการวิเคราะห์

```python
# Import modules
import json
import tweepy
import csv
import json
import pandas as pd
import numpy as np
import pickle
from IPython.display import display
import matplotlib.pyplot as plt
import seaborn as sns
import fileinput
import pickle
from pandas import Series
import datetime
import pytz
from tqdm import tqdm
```

หลังจากทำการสร้าง application ใน Twitter เป็นที่เรียบร้อยแล้ว เราจะได้ Token ทั้งหมด 4 ตัว ซึ่งในเชิงปฏิบัติเป็นเหมือนรหัสประจำตัว ที่เอาไว้สื่อสารกับ server

เนื่องจากรหัสควรเก็บไว้เป็นความลับ เราจึงต้องเก็บไว้ในรูปแบบของ JSON ไว้ใน working directory ของเราเอง แล้วค่อยเรียกใช้เมื่อจำเป็น แทนที่จะเก็บไว้ใน code โดยตรง ดังนี้

```python
# Create a dictionary to store your twitter credentials

twitter_cred = dict()

# Enter your own consumer_key, consumer_secret, access_key and access_secret
# Replacing the stars ("********")

twitter_cred['CONSUMER_KEY'] = '("********")'
twitter_cred['CONSUMER_SECRET'] = '("********")'
twitter_cred['ACCESS_KEY'] = '("********")'
twitter_cred['ACCESS_SECRET'] = '("********")'

# Save the information to a json so that it can be reused in code without exposing
# the secret info to public

with open('twitter_credentials.json', 'w') as secret_info:
    json.dump(twitter_cred, secret_info, indent=4, sort_keys=True)
```
และสามารถเรียกใช้จาก python โดยใช้ function นี้

```python
def load_api():
    with open('twitter_credentials.json') as cred_data:
        info = json.load(cred_data)
        consumer_key = info['CONSUMER_KEY']
        consumer_secret = info['CONSUMER_SECRET']
        access_key = info['ACCESS_KEY']
        access_secret = info['ACCESS_SECRET']
    auth = tweepy.AppAuthHandler(consumer_key, consumer_secret)
    # load the twitter API via tweepy
    api = tweepy.API(auth, wait_on_rate_limit=True, wait_on_rate_limit_notify=True)
    if (not api):
        print ("Can't Authenticate")
        sys.exit(-1)
    return api
```

ต่อไปเป็นการ scrape ข้อมูลจาก Twitter โดยปกติแล้ว standard search api ถ้าใช้ user authentication จะมี rate limit อยู่ที่ 180 requests ต่อ 15 นาที โดย max query = 100 ซึ่งเท่ากับ 18,000 tweet ต่อ 15 นาที

ผมได้ข้อมูลจาก <a href='https://bhaskarvk.github.io/2015/01/how-to-use-twitters-search-rest-api-most-effectively./'>ที่นี่</a> ว่า Tweepy สามารถรองรับ application-only authentication ซึ่งมี rate limit อยู่ที่ 450 requests ต่อ 15 นาที โดย max query = 100 ซึ่งเท่ากับ 45,000 tweet ต่อ 15 นาที (เร็วกว่าเดิมพอสมควร) ผมจึงเลือกใช้วิธีนี้ในการดึงข้อมูลจาก Twitter api

นี่คือองค์ประกรอบหลักของ python script ที่ผมสร้่างขึ้นเพื่อ scrape ข้อมูล

ส่วนแรกคือตัว function ที่เอาไว้สื่อสารกับ server 

```python
# Function for searching tweets    
def tweet_search(api, query, max_tweets, max_id, since_id):
    searched_tweets = []
    try:
        new_tweets = []
        for tweet in tweepy.Cursor(api.search,q=query, count=100, tweet_mode='extended', since_id=str(since_id), max_id=str(max_id-1)).items(max_tweets):
            new_tweets.append(tweet)
        print('found',len(new_tweets),'tweets', api.rate_limit_status()['resources']['search'])
        if not new_tweets:
            print('no tweets found')
            return searched_tweets, max_id
        else:
            searched_tweets.extend(new_tweets)
            max_id = new_tweets[-1].id
    except tweepy.TweepError:
        print('exception raised, waiting 15 minutes')
        print('(until:', dt.datetime.now()+dt.timedelta(minutes=15), ')')
        time.sleep(15*60)      
    return searched_tweets, max_id
```

ส่วนที่สองคือ function ที่ค้นหา id ของ tweet ที่เก่าที่สุดจากช่วงวันที่ ที่เราได้ระบุไว้ ซึ่ง id นี้ จะทำหน้าที่เหมือนเป็นจุดเริ่มต้นของการ scrape

```python
def get_tweet_id(api, date='', days_ago=9, query='a'):
    ''' Function that gets the ID of a tweet. This ID can then be
        used as a 'starting point' from which to search. The query is
        required and has been set to a commonly used word by default.
        The variable 'days_ago' has been initialized to the maximum
        amount we are able to search back in time (9).'''

    if date:
        # return an ID from the start of the given day
        td = date + dt.timedelta(days=1)
        tweet_date = '{0}-{1:0>2}-{2:0>2}'.format(td.year, td.month, td.day)
        tweet = api.search(q=query, count=1, until=tweet_date)
    else:
        # return an ID from __ days ago
        td = dt.datetime.now() - dt.timedelta(days=days_ago)
        tweet_date = '{0}-{1:0>2}-{2:0>2}'.format(td.year, td.month, td.day)
        # get list of up to 10 tweets
        tweet = api.search(q=query, count=10, until=tweet_date)
        print('search limit (start/stop):',tweet[0].created_at)
        # return the id of the first tweet in the list
        return tweet[0].id
```

ส่วนที่สามคือ function สำหรับการจัดเก็บข้อมูลเอาไว้ใน working directory ของเรา
        
```python
def write_tweets(tweets, filename):
    ''' Function that appends tweets to a file. '''

    with open(filename, 'a') as f:
        for tweet in tweets:
            json.dump(tweet._json, f)
            f.write('\n')      
 ```
 
 ส่วนสุดท้ายคือตัว main function ที่เราใช้ในการ loop over สิ่งที่เราต้องการให้ Tweepy ค้นหาบน Twitter search
 
 อันนี้ยาวนิดนึง
 
 ```python
def main():
    ''' This is a script that continuously searches for tweets
        that were created over a given number of days. The search
        dates and search phrase can be changed below. '''



    ''' search variables: '''
    search_phrases = ['#เลือกตั้งปี62', '#แม่มาแล้วธานอส', '#พระราชโองการ', '#ทรงพระสแลนด์เดอร์', '#เลือกตั้ง2562']
    
    time_limit = 240                          # runtime limit in hours
    max_tweets = 10000                           # number of tweets per search (will be
                                               # iterated over) - maximum is 100
    min_days_old, max_days_old = 2, 8          # search limits e.g., from 7 to 8
                                               # gives current weekday from last week,
                                               # min_days_old=0 will search from right now
    #USA = '39.8,-95.583068847656,2500km'       # this geocode includes nearly all American
                                               # states (and a large portion of Canada)
    

    # loop over search items,
    # creating a new file for each
    for search_phrase in search_phrases:

        print('Search phrase =', search_phrase)

        ''' other variables '''
        name = search_phrase.split()[0]
        json_file_root = name + '/'  + name
        os.makedirs(os.path.dirname(json_file_root), exist_ok=True)
        read_IDs = False
        
        # open a file in which to store the tweets
        if max_days_old - min_days_old == 1:
            d = dt.datetime.now() - dt.timedelta(days=min_days_old)
            day = '{0}-{1:0>2}-{2:0>2}'.format(d.year, d.month, d.day)
        else:
            d1 = dt.datetime.now() - dt.timedelta(days=max_days_old-1)
            d2 = dt.datetime.now() - dt.timedelta(days=min_days_old)
            day = '{0}-{1:0>2}-{2:0>2}_to_{3}-{4:0>2}-{5:0>2}'.format(
                  d1.year, d1.month, d1.day, d2.year, d2.month, d2.day)
        json_file = json_file_root + '_' + day + '.json'
        if os.path.isfile(json_file):
            print('Appending tweets to file named: ',json_file)
            read_IDs = True
        
        # authorize and load the twitter API
        api = load_api()
        
        # set the 'starting point' ID for tweet collection
        if read_IDs:
            # open the json file and get the latest tweet ID
            with open(json_file, 'r') as f:
                lines = f.readlines()
                max_id = json.loads(lines[-1])['id']
                print('Searching from the bottom ID in file')
        else:
            # get the ID of a tweet that is min_days_old
            if min_days_old == 0:
                max_id = -1
            else:
                max_id = get_tweet_id(api, days_ago=(min_days_old-1))
        # set the smallest ID to search for
        since_id = get_tweet_id(api, days_ago=(max_days_old-1))
        print('max id (starting point) =', max_id)
        print('since id (ending point) =', since_id)
        


        ''' tweet gathering loop  '''
        start = dt.datetime.now()
        end = start + dt.timedelta(hours=time_limit)
        count, exitcount = 0, 0
        while dt.datetime.now() < end:
            count += 1
            print('count =',count)
            # collect tweets and update max_id
            tweets, max_id = tweet_search(api, search_phrase, max_tweets,
                                          max_id=max_id, since_id=since_id)
            # write tweets to file in JSON format
            if tweets:
                write_tweets(tweets, json_file)
                exitcount = 0
            else:
                exitcount += 1
                if exitcount == 3:
                    if search_phrase == search_phrases[-1]:
                        sys.exit('Maximum number of empty tweet strings reached - exiting')
                    else:
                        print('Maximum number of empty tweet strings reached - breaking')
                        break


if __name__ == "__main__":
    main()
```
### ขั้นตอนที่สอง: Parsing JSON data to Pandas

หลังจากรวบรวมข้อมูลเรียบร้อยแล้ว ขั้นตอนต่อไปคือการเรียบเรียงข้อมูลให้อยู่ในรูปแบบที่เราสามารถวิเคราะห์ได้ง่ายๆ โดยเลือกเฉพาะข้อมูลที่เราสนใจเช่น id, created_at, text ฯลฯ

อย่างแรกที่ควรทำคือสร้าง function สำหรับการจัดเก็บข้อมูล hashtag ในแต่ละ tweet ให้อยู่ใน list จะได้ไม่ต้องใช้ string search เก็บเอาในภายหลัง

```python
# Function for extracting hashtags into lists
def hash_parse(tweet):
    hashes = list()
    for hashtag in tweet:
        text = hashtag['text']
        hashes.append(text)
    return hashes
```

และ function สำหรับดึงข้อมูล text จาก tweet

```python
def getText(data):       
    # Try for extended text of original tweet, if RT'd (streamer)
    try: text = data['retweeted_status']['extended_tweet']['full_text']
    except: 
        # Try for extended text of an original tweet, if RT'd (REST API)
        try: text = data['retweeted_status']['full_text']
        except:
            # Try for extended text of an original tweet (streamer)
            try: text = data['extended_tweet']['full_text']
            except:
                # Try for extended text of an original tweet (REST API)
                try: text = data['full_text']
                except:
                    # Try for basic text of original tweet if RT'd 
                    try: text = data['retweeted_status']['text']
                    except:
                        # Try for basic text of an original tweet
                        try: text = data['text']
                        except: 
                            # Nothing left to check for
                            text = ''
    return text
```

แทนที่จะโหลดไฟล์ทั้งหมดเข้าไปใน python และ append เพื่อสร้าง list อันใหญ่ ผมแนะนำให้ process ทีละบรรทัด และเขียนเข้า csv แล้วค่อยโหลดเข้า pandas อีกทีครับ


```python
# Load JSON files
tweet_files = ['#ทรงพระสเลนเดอร์/#ทรงพระสเลนเดอร์_2019-02-07_to_2019-02-12.json',
               '#ทรงพระสแลนด์เดอร์/#ทรงพระสแลนด์เดอร์_2019-02-07_to_2019-02-12.json',
               '#ไทยรักษาชาติ/#ไทยรักษาชาติ_2019-02-07_to_2019-02-12.json',
               '#พระราชโองการ/#พระราชโองการ_2019-02-07_to_2019-02-12.json',
               '#เลือกตั้ง62/#เลือกตั้ง62_2019-02-07_to_2019-02-12.json',
               '#เลือกตั้งปี62/#เลือกตั้งปี62_2019-02-07_to_2019-02-12.json',
               '#เลือกตั้ง2562/#เลือกตั้ง2562_2019-02-07_to_2019-02-12.json',
               '#แม่มาแล้วธานอส/#แม่มาแล้วธานอส_2019-02-07_to_2019-02-12.json'
              ]


f = open('tweets.csv', 'w')
writer = csv.writer(f)
for file in tweet_files:
    print(file)
    with open(file, 'r') as f:
        for line in tqdm(f.readlines()):
            tweets = []
            data = json.loads(line)
            tweets.extend([data['id'], data['created_at'], data['user']['screen_name'], data['user']['id'],
                           getText(data), data['retweet_count'], data['favorite_count'],
                           0 if ('retweeted_status' not in data) else 1, data['in_reply_to_status_id'],
                           data['in_reply_to_user_id'], data['user']['location'], data['place']['full_name']
                           if data['place'] != None else '',
                           hash_parse(data['entities']['hashtags']), data['place']['country_code']
                           if data['place'] != None else '', data['coordinates']['coordinates'][0]
                           if data['coordinates'] != None else 'NaN',
                           data['coordinates']['coordinates'][1]
                            if data['coordinates'] != None else 'NaN'
                          ])
            writer.writerow(tweets)            
f.close()
```

### ขั้นตอนที่สาม: Analysis and Visualization

เริ่มด้วยการอ่านข้อมูล csv ที่เราได้สร้างขึ้นมาจากขั้นตอนที่แล้ว

```python
# Applying the function
keys =  ['id', 'created_at', 'user_name', 'user_id', 'text', 'retweet_count', 'favorite_count', 'retweeted_status', 'in_reply_to_status_id',
              'in_reply_to_user_id', 'user_location', 'place', 'hashtags', 'country_code', 'long', 'latt']
df= pd.read_csv('tweets.csv', names = keys, converters={'hashtags': eval}) # use eval in order to retain list type object for hashtags

```
เนื่องจาก tweet หนึ่งอันอาจมี hashtag ที่อยู่ใน search query ของเรามากกว่า 1 อัน จึงต้องทำการ drop tweet ที่ซ้ำกัน โดยใช้ id เป็นตัวประเมิน

```python
# Drop duplicates. This is an important step since a single tweet may contain multiple hashtags and we use the search api with hashtags as our query.
df = df.drop_duplicates(subset='id')
```
แปลง 'created_at' ให้อยู่ในรูปแบบของ datetime จากนั้นจึงเปลี่ยน timezone และ filter เอาเฉพาะ tweet ระหว่างวันที่ 7 - 11

```python
df['time'] = pd.DatetimeIndex(df['created_at'])
df['time'] = pd.to_datetime(df['time'], format='%Y-%b-%d %H:%M:%S.%f').dt.tz_localize('UTC').dt.tz_convert('Asia/Bangkok')
df = df[(df['time'] >= '2019-02-07 00:00:00+07:00') & (df['time'] <= '2019-02-11 23:59:59+07:00')]
```
```python
df.shape
```
    (2761112, 16)
    
สร้าง function สำหรับ นับจำนวน hashtag ทั้งหมด

```python
# Function for counting hashtags
def count_hashtags(df):
    hashtag_list = []                          #CREATE EMPTY LIST 
    for i in df.hashtags:    #LOOP OVER EVERY CELL IN ENTITIES_HASHTAGS
        j = ', '.join(i)
        if pd.notnull(j):                      #IF CELL NOT EMPTY
            tags = j.split()                   #SPLIT EACH CELL INTO SEPARATE HASHTAGS
            for t in tags:                     #FOR EACH TAG IN THE CELL
                t = "#"+t                      #ADD '#' SYMBOL TO BEGINNING OF EACH TAG
                t = t.replace(',', '')         #REMOVE COMMAS FROM END OF TAGS
                t = t.lower()                  #MAKE TAG LOWER CASE
                hashtag_list.append(t)         #ADD TAG TO OUR LIST
    top = pd.DataFrame(Series(hashtag_list).value_counts(), columns=['count'])
    return(top)
```
นับจำนวน hashtag ทั้งหมด และเลือกเฉพาะ top 15
    
```python

# Count hashtags and store a list of top hashtags in tags
df_top = count_hashtags(df)
tags = df_top[0:15].index.tolist()
print(tags)
```

    ['#ไทยรักษาชาติ', '#เลือกตั้ง62', '#ทรงพระสเลนเดอร์', '#เลือกตั้งปี62', '#อนาคตใหม่', '#แม่มาแล้วธานอส', '#พระราชโองการ', '#ทรงพระสแลนด์เดอร์', '#ฟ้ารักพ่อ', '#ไทยรัฐเลือกตั้ง62', '#เลือกตั้ง2562', '#พรรคอนาคตใหม่', '#tucuball73', '#ฟ้ารักพ่อเพราะนโยบายของพ่อ', '#ทรงพระแคนเซิล']



สร้าง column จาก hashtag ดังกล่าว ถ้ามี = 1 ไม่มี = 0


```python
# Generating columns of hashtags
hashtags = pd.DataFrame(columns=['column_one'])
for tag in tags:
    df[tag] = df.hashtags.apply(lambda x: 1 if tag[1:] in x else 0)
    hashtags[tag] = df[tag]
hashtag_matrix = hashtags.drop('column_one', axis=1)
```

สร้าง heatmap เพื่อดู co-occurrence ของ hashtag

```python
# Create a heatmap of hashtags to see co-occurence
correlations = hashtag_matrix.corr()

# plot the correlation matrix
plt.figure(figsize=(10,10))
plt.rcParams["font.family"] = "Sukhumvit Set"
sns.heatmap(correlations,
    cmap='RdBu',
    vmin=-1,
    vmax=1,
    square = True,
    cbar_kws={'label':'correlation'})
plt.show()
```
<img src="https://naponjatusripitak.github.io/img/twitter_feb7-11_17_0.png">

ทำการนับความถี่ของ hashtag ต่อนาที

```python
# Counting hashtags per minute
# df['time'] = pd.DatetimeIndex(df['created_at'])
grouper = df.groupby([pd.Grouper(key='time', freq = '1Min')])
df1 = grouper[[s for s in tags]].sum()
```
แปล time series ให้อยู่ในรูปแบบยาว และจัดการเรื่อง timezone

```python
# Convert from wide to long format & adjust for time difference
df1.reset_index(level=0, inplace=True)
df1 = df1.melt(id_vars='time')
df1['time'] = pd.to_datetime(df1['time'], format='%Y-%b-%d %H:%M:%S.%f').dt.tz_localize('UTC').dt.tz_convert('Asia/Bangkok')
# Remove timezone just in case
df1['time'] = df1['time'].dt.tz_localize(None)
```
ทำการ plot ด้วย Plotly

```python
# Log in with plotly credentials
import plotly 
import plotly.plotly as py
import plotly.graph_objs as go
key= ''
plotly.tools.set_credentials_file(username='taozaze', api_key=key)
```


```python
# Graph Time Series
d={}
data = []
for tag in tags:
        d["trace_{0}".format(tag)]= go.Scatter(
            x = df1[df1.variable == tag]['time'],
            y = df1[df1.variable == tag]['value'],
            mode = 'lines',
            name = tag
        )
        data.append(d["trace_{0}".format(tag)])

# slider set up:
layout = dict(
    title='Tweet Time Distribution by Hashtag',
    xaxis=dict(
        title = 'Time',
        rangeselector=dict(
            buttons=list([
                dict(count=1,
                     label='1d',
                     step='day',
                     stepmode='backward'),
                dict(count=6,
                     label='6h',
                     step='hour',
                     stepmode='backward'),
                dict(count=1,
                    label='1h',
                    step='hour',
                    stepmode='backward'),
                dict(step='all')
            ])
        ),
        rangeslider=dict(
            visible = True
        ),
        type='date'
    ),
    yaxis = dict(title = 'Tweets Per Minute')
)
py.iplot(go.Figure(data=data, layout=layout), filename = 'slider-line', auto_open=True)
```
<iframe id="igraph" scrolling="no" style="border:none;" seamless="seamless" src="https://plot.ly/~taozaze/15.embed" height="525px" width="100%"></iframe>




```python
# Creat bar plot for top hashtags
top_hash = df_top.reset_index()[0:15]
data = [
    go.Bar(
        x=top_hash['index'], # assign x as the dataframe column 'x'
        y=top_hash['count']
    )
]

# Layout and plot
layout = dict(title = 'Top Hashtags',
              xaxis = dict(title = 'Hashtag'),
              yaxis = dict(title = 'Hashtag Frequency'),
              )

py.iplot(go.Figure(data=data, layout=layout), filename = 'pandas-bar-chart', auto_open=True)
```
<iframe id="igraph" scrolling="no" style="border:none;" seamless="seamless" src="https://plot.ly/~taozaze/9.embed" height="525px" width="100%"></iframe>

สำหรับ bar chart ที่เลือกวันที่ได้

```python
# Count Hashtags for Feb 7 - 11
df_top_feb_7_11 = count_hashtags(df).reset_index()[0:15]

# Create dataframes for each day
d = {}
for i in [7, 8, 9, 10, 11]:
    d['df_top_feb_{}'.format(i)] = count_hashtags(df[(df['time'] >= '2019-02-0{} 00:00:00+07:00'.format(i)) & (df['time'] <= '2019-02-0{} 23:59:59+07:00'.format(i))]).reset_index()[0:15]

# Create bar charts
import plotly.plotly as py
import plotly.graph_objs as go 

from datetime import datetime
import pandas as pd

trace = {}
data = []

trace_all = go.Bar(
        x=df_top_feb_7_11['index'], # assign x as the dataframe column 'x'
        y=df_top_feb_7_11['count'],
        name = 'All')

for i in [7, 8, 9, 10, 11]:
    trace['{}'.format(i)] = go.Bar(
        x= d['df_top_feb_{}'.format(i)]['index'],
        y= d['df_top_feb_{}'.format(i)]['count'],
        name = 'Feb {}'.format(i),
        visible = False
    )
    data.append(trace["{0}".format(i)])
    
data.append(trace_all)
        
updatemenus = list([
    dict(type="buttons",
         active=-1,
         buttons=list([
            dict(label = 'Feb 7',
                 method = 'update',
                 args = [{'visible': [True, False, False, False, False, False]},
                         {'title': 'Top Hashtags on Feb 7'}]),
              dict(label = 'Feb 8',
                 method = 'update',
                 args = [{'visible': [False, True, False, False, False, False]},
                         {'title': 'Top Hashtags on Feb 8'}]),
              dict(label = 'Feb 9',
                 method = 'update',
                 args = [{'visible': [False, False, True, False, False, False]},
                         {'title': 'Top Hashtags on Feb 9'}]),
              dict(label = 'Feb 10',
                 method = 'update',
                 args = [{'visible': [False, False, False, True, False, False]},
                         {'title': 'Top Hashtags on Feb 10'}]),
             dict(label = 'Feb 11',
                 method = 'update',
                 args = [{'visible': [False, False, False, False, True, False]},
                         {'title': 'Top Hashtags on Feb 11'}]),
                dict(label = 'All',
                 method = 'update',
                 args = [{'visible': [False, False, False, False, False, True]},
                         {'title': 'Top Hashtags (Feb 7-11)'}])
        ]),
         direction = 'up',
         xanchor = 'right',
         yanchor = 'top',
         x = 1,
         y = 1
    )
])

layout = dict(title='Top Hashtags (Feb 7-11)',
              yaxis = dict(title = 'Hashtag Frequency'), showlegend=False,
              updatemenus=updatemenus)

py.iplot(go.Figure(data=data, layout=layout), filename = 'buttons-bar-chart', auto_open=True)
```
<iframe id="igraph" scrolling="no" style="border:none;" seamless="seamless" src="https://plot.ly/~taozaze/17.embed" height="525px" width="100%"></iframe>










