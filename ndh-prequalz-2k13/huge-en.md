Huge.js - Writeup
================

## Intro

"Find the correct unlock code. Be ready to wait, it is huuuuuge."

## File recovery
In first open file with vim and we can directly see interesting functions at the beginning

```
function d() {
  return (window.console &&
         (window.console.firebug || window.console.exception));
}

function x(c, k) {
  o = '';
  for (i = 0; i < c.length; i++) {
    o += String.fromCharCode(c.charCodeAt(i) ^ k.charCodeAt(i % k.length));
  }
}
if (!d()) {
    window[x('\x40\x33\x58\x58\x5e','\x21\x5f\x3d\x2a\x2a\x2a\x24\x24')]=null; //alert()
    window[x('\x44\x29\x42\x42','\x21\x5f\x23\x2e\x2b\x28\x3f\x2d')]( //eval()
```

To solutions to retrieve the content 
Two solutions are available to retrieve the contents.
With firebug (firebox add-on), by writing `console.log(window)` we notice that the function is loaded and displayed in plain text.
Or via a simple python script

```
#!/usr/bin/python

#function x(c,k){
#    o='';
#    for (i=0;i<c.length;i++) {
#        o+=String.fromCharCode(c.charCodeAt(i)^k.charCodeAt(i%k.length));
#    }
#    return o;
#}

#Rebuild x function
def x(c, k):
    o = ''
    for i in range(len(c)):
        o += (chr(ord(c[i])^ord(k[i%len(k)])))
    return o


#Decode string from file and save in output filename
def decode(filename, output):
    content = open(filename, 'r').read()
    pos = content.find("x('\\", 0)
    while pos != -1:
        sep = content.find(",", pos) 
        end = content.find(")", pos)
        #ignore "x('" chars with +3
        c = content[pos+3:sep-1].replace("\\x", "").decode("hex")
        #ignore ");" with sep+2
        k = content[sep+2:end-1].replace("\\x", "").decode("hex")
        #Replace content
        content = content[:pos] + x(c, k) + content[end+1:]
        #Update position
        pos = content.find("x('\\", 0)
    f = open(output, 'w')
    f.write(content)
    f.close()

decode('huge.js', 'huge-readable.js')
```


## Main information? 

Open the huge-readable.js file and admire the result

```
xxx('hello') != '5d41402abc4b2a76b9719d911017c592'
```
This look like an md5 hash and it's confirmed quickly. So xxx() generate md5 hash ;)


```
function unlock(node) {
    var code = node.value;
    if (code && (code.length == 5)) {
        if ((xxx(code).substr(0, 16) == 'b3336efd42e29780') && (xxx(kkk(code)).substr(16, 16) == '261804c5f2f0a47e')) {
            var flag = document.createElement('span');
            var t = document.createTextNode('YAY, you got the flag ! --> ' + xxx(code));
            flag.appendChild(t);
            flag.style.font = 'Helvetica, 1.2em, bold';
            document.getElementById('panel').appendChild(flag);
            node.style.display = 'none';
            node.value = '';
        }
    }
} /* Create an input box */
```

We must find a code with 5 characters: code.length == 5
And an md5 hash beginning by `b3336efd42e29780̀` and for the rest we do not care because HashCat can save us \o/


## HashCat FTW

For those who do not know Hashcat is the world’s fastest CPU-based password recovery tool.

```
./cudaHashcat-lite64.bin -m 5100 --status --pw-max 5 --custom-charset1 "?l?u?d?s" b3336efd42e297800000000000000000 "?1?1?1?1?1"
cudaHashcat-lite v0.14 by atom starting...

Password lengths: 1 - 5
Watchdog: Temperature abort trigger set to 90c
Watchdog: Temperature retain trigger set to 80c
Device #1: GeForce GTX 460M, 1535MB, 1350Mhz, 4MCU

b3336efd42e297800000000000000000:i<3Js

Session.Name...: cudaHashcat-lite
Status.........: Cracked
Hash.Target....: b3336efd42e297800000000000000000
Hash.Type......: Half MD5
Time.Started...: Mon Mar 11 21:15:24 2013 (10 secs)
Time.Estimated.: Mon Mar 11 21:15:43 2013 (7 secs)
Plain.Mask.....: ?1?1?1?1?1
Plain.Text.....: *4fHp
Plain.Length...: 5
Progress.......: 4365926400/7737809375 (56.42%)
Speed.GPU.#1...:   436.8M/s
HWMon.GPU.#1...: -1% Util, 63c Temp, -1% Fan

Started: Mon Mar 11 21:15:24 2013
Stopped: Mon Mar 11 21:15:36 2013

```

It find "i<3Js"
We write this in the input, and omfg, an hash: `b3336efd42e2978035cb54f85f1654f6`

**Flag**: `b3336efd42e2978035cb54f85f1654f6`

## Authors
* GoT <pierre.rambaud86@gmail.com>
