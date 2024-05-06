---
title: "YouTube Data APIを利用して、動画タイトルとコメントを収集する"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Python","YouTube"]
published: true
---

## YouTube Data APIについて
### 概要
「YouTube Data API v3」を利用。Google Cloudで提供されています。
無料で利用可能ですが、1日10000クォータの利用制限があります。
この「10000クォータ」は、どのようなリクエストを行うかによって計算されるようです。なお、今回のリクエスト（50件の動画でそれぞれ300コメント取得）を数回実行しても、1000クォータも消費していない程度です。動画のアップロードなどで消費が多いようです。
https://developers.google.com/youtube/v3/determine_quota_cost?hl=ja
https://apidog.com/jp/blog/youtube-data-api/

### 利用方法
利用するにあたっては、Googleアカウントの登録が必要となります。
プロジェクトを作成し、プロジェクトに紐づける形でAPIを有効化します。（他のAPIも同様）
APIキーを払い出し、利用が可能となります。APIキーは悪用されないよう、管理には注意しましょう。
https://qiita.com/kaburankattara/items/240a457f79ade800a4fb

## 環境
Google Colaboratoryを利用。

## 使ってみる
リファレンスは以下です。
https://developers.google.com/youtube/v3/docs?hl=ja

以下の条件で動画データを取得。
- 特定のプレイリストを指定
- 最新50件の動画の情報を取得（※）
- 各動画「動画ID」「タイトル」「投稿日時」「コメント」を取得
- コメントは最新300件を取得
- 取得したデータはCSV形式で保存する

※動画の取得上限について
YouTube Data APIでは、一度の検索で取得できる件数が最大50件とのことです
https://zenn.dev/jqinglong/articles/1161615fdaa6f6

以下のコードを実行。

### 動画情報の取得
```py
#ライブラリのインポート
from googleapiclient.discovery import build
from datetime import datetime

# APIキーを設定
API_KEY = 'YOUR_API_KEY' #取得したAPIキーをコピペする

# YouTubeデータAPIクライアントを作成
youtube = build('youtube', 'v3', developerKey=API_KEY)

# プレイリストID
PLAYLIST_ID = 'PLAYLIST_ID' #対象のプレイリストのIDをコピペ

# 動画情報を格納するリスト
video_data = []

# プレイリスト内の動画を取得
playlist_response = youtube.playlistItems().list(
    playlistId=PLAYLIST_ID,
    part='snippet',
    maxResults=50  # 最大取得件数（YouTube APIの制限）
).execute()

# 動画の情報を取得し、リストに追加
for playlist_item in playlist_response.get('items', []):
    video_id = playlist_item['snippet']['resourceId']['videoId']
    video_title = playlist_item['snippet']['title']
    video_published_at = playlist_item['snippet']['publishedAt']

    # 動画の詳細情報を取得
    video_response = youtube.videos().list(
        id=video_id,
        part='snippet,statistics'
    ).execute()

    # 動画の投稿日時
    published_at = datetime.strptime(video_published_at, '%Y-%m-%dT%H:%M:%SZ')

    # 最新300件のコメントを取得
    comment_threads = youtube.commentThreads().list(
        videoId=video_id,
        part='snippet',
        maxResults=300,
        textFormat='plainText'
    ).execute()

    comments = [comment['snippet']['topLevelComment']['snippet']['textDisplay']
                for comment in comment_threads.get('items', [])]

    # 動画情報をリストに追加
    video_data.append({
        'video_id': video_id,
        'title': video_title,
        'published_at': published_at,
        'comments': '\n'.join(comments)
    })
```

### CSV形式で保存
取得した動画のデータをCSV形式で保存。
今回はGoogle ColaboratoryにGoogle Driveをマウントし、ドライブ内に保存しています。
```py
# ライブラリのインポート
import csv

#Google ColabにGoogleドライブをマウント（必要な場合のみ）
from google.colab import drive
drive.mount('/content/drive')

# CSVファイルに保存
with open('YOUR_FILE_PATH', 'w', newline='', encoding='utf-8') as csvfile: #ファイルパスをコピペ
    fieldnames = ['video_id', 'title', 'published_at', 'comments']
    writer = csv.DictWriter(csvfile, fieldnames=fieldnames)

    writer.writeheader()
    for video in video_data:
        writer.writerow({
            'video_id': video['video_id'],
            'title': video['title'],
            'published_at': video['published_at'].isoformat(),
            'comments': video['comments']
        })
```

## おわりに
noteに掲載したこちらの取り組みの中でYouTube Data APIを利用しました。
ご興味のある方はこちらもご覧ください。
https://note.com/yebi_crab/n/n4866e35703cd
