---
layout: post
title:  "毎朝、その日の天気に応じた色でHueが点灯するようにしてみた"
categories: smart-home
---

毎朝、自動的にその日の天気情報を取得し、それに応じてHueが点灯するようにしてみた。

次のものを繋いで実現した。

- AWS Lambda
- OpenWeatherMap
- IFTTT
- Phillips Hue

プログラムの毎朝の自動実行には、AWS Lambdaを使った。

プログラムでは、

- OpenWeatherMapのAPIを使ってその日の天気情報を取得し、
- 天気から、Hueにセットしたい色を決め、
- IFTTTのMaker Channelへ送信

している。

ここから先は、IFTTTにあらかじめ用意しておいたレシピを使った。
プログラムから送られてきた色に合わせてPhillips Hueを点灯するレシピを作った。

### やりたかったこと

毎朝、自動的にHueが点灯するとうれしい。
起床に合わせて明かりがついたら、きっと少しは快適な目覚めになる気がする。

そのとき、明かりの色が違ったら楽しい。
起きた時の光景が毎日少しずつ違ったら、朝にうんざりしなくてすみそうである。
実用的にも、わざわざスマホの天気予報アプリとかを見なくても、
その日傘を持って外出すべきかすぐにわかって便利。

### OpenWeatherMapのセットアップ

詳細は割愛。

今回の方法では、天気の取得元としてOpenWeatherMapのWebAPIを用いている。
このWebAPIを利用するためには、APIキーが必要である。

サインインして、[API Keys](https://home.openweathermap.org/api_keys)を開いたときに書いてあるものがAPIキー。

APIキーは控えておき、あとでAWS Lambdaのコードに埋めこむ。

### IFTTTのセットアップ

先にIFTTTのレシピを作っておく。

当然、点灯したいPhillips Hueと接続済である必要があるが、割愛する。

1. IFTTTのホームを開く

2. "My Recipes"ボタンを押す

3. "Create a Recipe"ボタンを押す

  ![IFTTT Step3](/images/screenshots/2016-11-03-ifttt3.png)

4. "this"ボタンを押す

5. "Choose Trigger Channel"画面が表示されるので、"Maker"を選ぶ

  ![IFTTT Step5](/images/screenshots/2016-11-03-ifttt5.png)
  
6. "Receive a web request"を選ぶ

7. "Event Name"を"color_received"として、"Create Trigger"ボタンを押す

  ![IFTTT Step7](/images/screenshots/2016-11-03-ifttt7.png)
  
8. "that"ボタンを押す

9. "Choose Action Channel"画面が表示されるので、"Phillips Hue"を選ぶ

  ![IFTTT Step9](/images/screenshots/2016-11-03-ifttt9.png)

10. "Change color"を選ぶ

11. 次のように設定して、"Create Action"ボタンを押す:
  - Which lights: 点灯したいHue
  - Color value or name: {{Value1}}

  ![IFTTT Step11](/images/screenshots/2016-11-03-ifttt11.png)

12. "Recipe Title"は適当に設定し、"Create Recipe"ボタンを押す

  ![IFTTT Step12](/images/screenshots/2016-11-03-ifttt12.png)
  
13. [Maker Channel](https://internal-api.ifttt.com/maker)を開いて、表示されているキーを控える。

  ![IFTTT Step13](/images/screenshots/2016-11-03-ifttt13.png)
  
### AWS Lambdaのセットアップ

1. AWS Lambdaのホームを開く

2. "Create Lambda Function"ボタンを押す

3. "Blank Function"を選ぶ

4. "Configure triggers"の画面が出るので、Triggerのプルダウンから、"CloudWatch Events - Schedule"を選ぶ

5. 次のように設定する:
  - Rule name: "everyday0840"
  - Schedule expression: cron(40 23 ? * SUN-THU *)
  - Enable trigger: チェック

  自分は8:40に点灯して欲しいので、上記のようにした。
  "Schedule expression"にはCRONスタイルの指定ができる。
  ただしUTCなので、日本時間では9時間分引き算しないと狙った時間に発動しない。

  ![Lambda Step5](/images/screenshots/2016-11-03-lambda5.png)

6. "Next"ボタンを押す

7. "Configure function"の画面が出るので、次のように設定する
  - Name: WeatherEveryMorning (ここは何でもいい)
  - Runtime: Node.js 4.3

8. コードは以下:

  `iftttSecretKey`と`openWeatherMapApiKey`は各自で書き換えること
  当然、*書き換えなければ動かない*
    
```javascript
var http = require('http')
var https = require('https')

exports.handler = (event, context, callback) => {
    callback(null, 'Start WeatherToday')
    var iftttEventName = 'color_received'
    var iftttSecretKey = 'YourSecretKeyForIFTTTMakeChannel'
    var openWeatherMapApiKey = "YourOpenWeatherMapApiKey"
    var openWeatherMapApiCityId = "1850147" // Tokyo
    
    fetchWeather((weatherId) =>{ 
        sendIfttt({
            value1: colorByWeatherId(weatherId)
        },
        (res) => {
            console.log('done')
            context.done()
        },
        (err) => {
            console.log("error: " + err)
            context.done()
        })
    },
    (err) => {
        console.log("error: " + err)
        context.done()
    })
    
    function colorByWeatherId(weatherId) {
        /**
         * Convert weather id to light color.
         * Refer: [Weather Conditions - OpenWeatherMap](http://openweathermap.org/weather-conditions)
         */
        switch (Math.floor(weatherId / 100)) {
        case 2: // Group 2xx: Thunderstorm
            return "#55bbff"
        case 3: // Group 3xx: Drizzle
            return "#8888aa"
        case 5: // Group 5xx: Rain
            return "#5555ff"
        case 6: // Group 6xx: Snow
            return "#666666"
        case 8:
            if (weatherId == 800)
                // Group 800: Clear
                return "#ffaa00"
            else
                // Group 80x: Clouds
                return "#886644"
        }
        return '#f0f0f0'
    }

    function fetchWeather(success, error) {
        /**
         * Featch weather information from OpenWeatherMap.
         * Refer: [5 day weather forecast - OpenWeatherMap](http://openweathermap.org/forecast5)
         */
        var uri = "http://api.openweathermap.org/data/2.5/forecast/daily?id=" + openWeatherMapApiCityId + "&cnt=1&APPID=" + openWeatherMapApiKey

        http.get(uri, (res) => {
            var body = ""
        
            res.on("data", (chunk) => {
                body += chunk
            });
        
            res.on("end", () => {
                console.log("openweathermap response: " + body)
                var res = JSON.parse(body)
                var weather = res["list"][0]["weather"][0]["id"] // Get weather id
                success(weather)
            })
            
        }).on("error", error)
    }
    
    function sendIfttt(data, success, error) {
        var hostname = 'maker.ifttt.com'
        var path = '/trigger/' + iftttEventName + '/with/key/' + iftttSecretKey

        var postData = JSON.stringify(data);
        var options = {
            hostname: hostname,
            path: path,
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Content-Length': Buffer.byteLength(postData)
            }
        }
        var req = https.request(options, (res) => {
            var body = ""
        
            res.on("data", (chunk) => {
                body += chunk
            })
        
            res.on("end", () => {
                console.log("sent ifttt: " + body)
                success(body)
            })
        })
        req.on("error", error)
        req.write(postData)
        req.end()
    }
}
```


9. "Save and test"ボタンを押す。

    ![Lambda Step9](/images/screenshots/2016-11-03-lambda9.png)

10. 10秒くらい待って、Hueの色が変わったら成功。

