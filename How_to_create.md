
## GASで簡易サーバー作成

このコードコピペで、変数部分は変えれば、そのままコードは完成ですが、
少しだけコメントアウトで書きます！

<details>

<summary>全コードはこのアコーディオンの中</summary>

```javascript

// Tokenの変数
const VERIFICATION_TOKEN = 'xxxxxxxxxxxxxxxxxxxx';
const BOT_USER_OAUTH_TOKEN = 'xoxb-xxxxxxxxxxxxx-xxxxxxxxxxxxx-xxxxxxxxxxxxxxxxxxxxxxxxxx'
const SLACK_POST_URL = 'https://slack.com/api/chat.postMessage';


/* Google MeetのURLを作成 */
// Googleカレンダーに一時的なイベントを挿入し、そのイベントに関連付けられたGoogle MeetのURLを取得する関数。
function getMeetUrl() {
  const calendarId = 'primary'; 
  const dt = new Date();
  const date = dt.getFullYear() + '-' + (dt.getMonth() + 1) + '-' + dt.getDate();
  const requestId = Math.random().toString(32).substring(2); 
  // イベントをGoogleカレンダーに挿入し、conferenceDataVersionを指定
  const events = Calendar.Events.insert({
    summary: 'tmp_event',
    singleEvents: true,
    allDayEvent: true,
    start: { date },
    end: { date },
    conferenceData: {
      createRequest: {
        requestId,
        conferenceSolutionKey: {
          type: 'hangoutsMeet'
        },
      }
    }
  }, calendarId, { conferenceDataVersion: 1 })

  //　カレンダーはいらないから削除
  Calendar.Events.remove(calendarId, events.id);

  // イベントの作成が成功した場合はGoogle MeetのURLを返す
  if (events.conferenceData.createRequest.status.statusCode === 'success') {
    const meetUrl = events.conferenceData.entryPoints[0].uri;
    return meetUrl;
  }
}

/* Slackに投稿 */
function postMessage(event, message) {
  const thread_ts = event.thread_ts ?? event.ts;
  const params = {
    method: 'post',
    payload: {
      token: BOT_USER_OAUTH_TOKEN,
      channel: event.channel,
      thread_ts: thread_ts,
      text: message,
    },
  };
  UrlFetchApp.fetch(SLACK_POST_URL, params);
}

/* Slackにメッセージを送信 */
function doPost(e) {
  const meetUrl = getMeetUrl(); // Google MeetのURLを取得
  let message = meetUrl !== undefined ? `Meetの部屋を作ったよ\n${meetUrl}` : 'URL生成できませんでした。';
  let response = {
    response_type: 'in_channel',
    text: message,
  };

  if (e.parameter.command) {
    if (e.parameter.token !== VERIFICATION_TOKEN) {
      return null;
    }

    return ContentService.createTextOutput(JSON.stringify(response)).setMimeType(ContentService.MimeType.JSON);

  } else if (e.postData) {
    const contents = JSON.parse(e.postData.getDataAsString())

    if (contents.token !== VERIFICATION_TOKEN) {
      return null;
    }

    if (contents.type === 'url_verification') {
      return ContentService.createTextOutput(contents.challenge)

    } else if (contents.type === 'event_callback') {

      if (contents.event.subtype && contents.event.subtype === 'bot_message') {
        return null;
      }
  
      // Slackにメンションがあった場合、メッセージを投稿
      if (contents.event.type === 'app_mention') {
        postMessage(contents.event, message);
      }
    }
  }
}



```

</details>

<br />

コードの作成が完了したら、Google Calender APIを追加します。

### GAS：　Service追加

Serviceを押下して、Google Calender追加する。

![!\[\](<https://storage.googleapis.com/zenn-user-upload/2bd2176f93be-20240217.png>)](readme_pic/addService.png)

**これを追加しないと,ErrorでGoogle Meetの作成ができません！**(そりゃそう)

### GAS：　デプロイする

上のデプロイを押して、ウェブアプリでデプロイします。

![Alt text](readme_pic/deploy.png)

URLが出るので、コピーしておきましょう！
（後でevent設定で使うので、どっかにおいておいてね）

## slackApiの作成

[ここからじぶんのappを作成します。](https://api.slack.com/apps)

<https://api.slack.com/apps>

![Alt text](readme_pic/slackapp.png)

上段の欄で名前をつけて、下段の欄でチャンネルの選択をします。

![Alt text](readme_pic/choose.png)

作成後、
**`Verification Token`と`BotUser OAuthToken`**　をコピーして
先に作った**GASの定数部分に貼り付ける**　必要がありますが、
`OAuthToken`はアプリにスコープを設定しないとできないので、
先にその設定をします。

### slackのscope設定

左のサイドバーの`OAuth＆Permission`をクリックして、
Scopesの、`BotTokenScopes`の`Add an OAuth Scope`をクリック。

![Alt text](readme_pic/scope.png)

追加するスコープは、以下３つです。
**`app_mention:read`**
**`chat:write`**
**`command`**

![Alt text](readme_pic/scopechoose.png)

そうすると、以下画像のように、
**`OAuth Tokens for Your Workspace`**　のボタンが活性化するので、
**`Install To WorkSpace`**　をおそう。

![Alt text](readme_pic/oauth.png)

許可します！！！
![Alt text](readme_pic/ok.png)

### GASの定数部分に代入しよう

**`BotUser OAuthToken`が出来上がりました！！！！
コピーして、GASの変数に代入しよう！！！**
![](readme_pic/token.png)

**Basic Informationから`Verification Token`もコピーして代入**です。

`Verification Token`と`BotUser OAuthToken`をコピーし
先に作ったGASの定数部分に貼り付け完了したら、今度は...

### slash commandを有効に

サイドバーのslash commandをクリックして、create！

![Alt text](readme_pic/slash.png)

### Event Subscriptionsを有効に

上記でコピーしたデプロイURLをRequest URLに貼り付けよう。
