# Web-security-homelab
Configured and deployed a secure homelab on a Raspberry Pi, using Docker. 

— Project Synopsis —

Configured and deployed a secure Homelab on a Raspberry Pi, using Docker. I containerized 3 environments - Ubuntu, Kali, and NginX. Ubuntu would serve as the “victim”, or host machine - Kali serves as the attacking machine - and NginX serves as the reverse Proxy for the victim. This project is to simulate monitoring fraudulent traffic to a web server. 

— Project Scope — 

- Setup a Raspberry Pi and enable SSH (I used a Pi 4 B, you don’t need this exact model.)
- Configure Docker, and download 3 docker images for Ubuntu, Kali, and NginX.
- Install all necessary tools and shells for all containers.
- Create a basic HTML page.
- Configure NginX, as to direct host OS directory to NGINX web directory to serve a static website over port 80 - We want to view HTML on host machines browser.
- Configure .C2 config file to enable the reverse proxy.

---

Before we begin, small disclaimer: this article isn’t particularly a step-by-step for complete beginners. While I do describe my process, I won’t go into superb detail that one may need to learn from scratch. For that, I highly recommend you follow the same guide I did, by Grant Collins [here](https://www.youtube.com/watch?v=MshVeYlpE90&t=420s&ab_channel=GrantCollins).

Another disclaimer: Grant’s tutorial, while an incredible hands-on exercise, has multiple holes. Due to being slightly outdated, some of his on-screen methods simply don’t work at face value, and you’ll be forced to troubleshoot and research alternative ways to configure, or install certain tools. **This is a blessing in disguise, if your main goal is to learn. Embrace the process - if I can do it, so can you.** Containerization has proven to be incredibly efficient for self-learning, so regardless of what you do, learn it somehow!

Final disclaimer: You don’t need a Raspberry Pi to complete this project. You can absolutely run Docker locally on Windows, Mac or Linux and develop this home-lab there. Skip over any mention of Raspberry Pi if this is the case.

---

Let’s begin!

First, I ensured my Raspberry Pi had Raspbian, along with a set hostname and SSH was enabled. 

I used `curl -fsSL https://get.docker.com -o [get-docker.sh](http://get-docker.sh)` in SSH to download docker onto my Pi. Ensure it’s installed by using `ls` to view the file on Pi. From here, you can input docker and see all the available syntax you can execute with Docker. 

From here, we’re going to install our necessary images (after logging in via SSH). These images will contain the elements of the homelab we need, such as Ubuntu, Linux and NGINX. (If you follow along in Grant Collins’ tutorial, you’ll already see your first “hole” I mentioned earlier. The docker image of kali he provides is completely broken, so you'll have to install that one from the official command.)

Now, to make sure Docker is working properly, I went ahead and typed `docker run hello-world`. This will automatically pull an official Docker image that, once installed, will run and let you know that your installation is working. 

I ran each docker image to make sure each individually worked. I named the Ubuntu image “web_server”, the Kali machine “attacker”, and the Nginx server as “reverse_proxy”. I made sure to run the NGINX image using port 80. This particular server is running Linux Alpine, a lighter-weight OS designed to be small and simple. 

To save myself time, I ran `sudo usermod -aG docker pi`. This adds the “pi” user to the “docker” group, escalating my privileges just enough to avoid typing sudo before every future command within the confines of Docker. 

Starting with the Kali container, I executed `docker exec -i attacker /bin/bash`, followed by `apt update`, and then `apt install iproute2`, `apt install iputils-ping`, and lastly `apt install git`. (All utilities can be installed in the same command! I.e. `apt install iproute2 iputilts-ping`)

Following the same protocol, I opened a new cmd on my primary Windows PC to SSH to my Raspberry pi, to have a live view of all terminals of each container at once. I ran `docker exec -i web_server /bin/bash`, and installed ip and ping utilities. 

I again ran `apk update`, then installed the Bash shell in NGINX via `apk add bash-completion` (this step isn’t necessary, but I want continuity between all containers). `ping` and `ip` should already be available here.

After establishing they all have potential connectivity with each other, I installed my own HTML webpage I created (which includes very basic HTML and imagery), a .png, and c2 configuration files all onto the Nginx container. The [starter tutorial](https://youtu.be/MshVeYlpE90) provides these, but I personally substituted my own webpage. `wget —no-check-certificate` is your best friend here.

After navigating back into my Nginx container (`docker exec -i reverse_roxy /bin/bash`), I made sure my files copied over successfully and moved them into the `nginx` folder, and replaced the default `index.html` file with my own. The location where we’re going to place the .c2 file is /etc/nginx/conf.d. `mvc2_config.c2/etc/nginx/conf.d` Again, navigate to the appropriate file path and make sure your file is there.

Now, we need to use OpenSSL to generate a self-signed certificate - we want HTTPS instead of HTTP for our reverse proxy. I first made a directory named “private”, in `/etc/ssl/private/`.

I then ran this syntax that generated a RSA private key: `openssl req -x509 -nodes -days 365 -newkey rsa:4096 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt -subj "/C=US/ST=NYO=DEE, BOODAH./OU=IT/CN=deeboodah.com"`

Using `ip addr` for appropriate containers, I used the the `vi` text editor that comes preinstalled on Alpine to edit the config file, and changed the first line in the .c2 from `listen 172.17.0.2:443 http2 ssl default_server;`to the IP that your NGINX container is. Then under the `location /msf` I added the Kali Linux container IP in place of the provided URL, along with the port number 5555. Finally, I executed `nginx -s reload` in the same Nginx environment, and curl -k https://<your IP here>. If all goes well, you should see the HTML code associated with the webpage you installed. 

Now if you go into your browser on either your Raspberry Pi or your local machine (if you don’t use Pi), and visit the reverse proxy IP, you’ll be visited by the .PNG page that was uploaded earlier. 

Back into the Kali container, I downloaded python 3, the pip utility, and copied the link to the HTTPS Python server code from [here](https://github.com/collinsmc23/simple-https-python-server/blob/main/simple_https.py).  If you want to learn how to write this script yourself, follow [this tutorial](https://anvileight.com/blog/posts/simple-python-http-server/)! Otherwise, git clone the link I provided.

Concatenating the .py script will reveal two variables at the bottom, both for a key and a certification for a website. To transfer both the key and cert file from the reverse proxy, I went to the Nginx server, navigated to `/etc/ssl/certs/` and `/etc/ssl/private/`, and copied them into freshly created files (`touch nginx-selfsigned.crt` and `touch nginx-self-signed.key`) my Kali machine, as well as installing the `nano` editor to achieve this. Nano into the the .py file, and fill the `keyfile=` and `certfile=` fields with the appropriate files that the keys are in. 

Make sure http is installed in python, by executing `python3` to open the py shell, then `import http`. `pip3 install http` would work, if you don’t already have it.

Finally, after executing `./simple_https.py` , there should be no output. Go over to the reverse proxy and do `curl -k https://<kali server ip here>:5555`, you **should** receive a “Communication Established” message back. We’re in. 

Ctrl + C on the Kali machine will interrupt the server, causing it to quit. Open the Ubuntu cmd window, and we’re all setup! On the Ubuntu machine, I did `curl -k https://<proxy ip>`, and should see the HTML code. 

`curl -k https://<proxy ip>/msf` should grant you a satisfying “Communication Established” message, as well as the “GET 200” code on the Kali server window. Now you can route through any command you want!

---

Thanks for reading! This is great practice for installing new tools on different systems, and is 100% real-world applicable. The vast amount of tools and protocols used in this project also gives you a realistic approach on configuration, file management, containerization, and even hands-on with Python! 

As I always do, I recommend this to anyone who’s interested in taking a deep dive in Linux, Docker, Python and web-server activity monitoring. This project alone checks off a lot of boxes, including all the goals set by the Project Scope at the top of this page. 

Detailed step-by-step approach from scratch: https://www.youtube.com/watch?v=MshVeYlpE90&t=420s&ab_channel=GrantCollins

Files used: https://drive.google.com/drive/folders/1mATuaN3Zx8tnSwB7iA8ysfmgIVfCx1oa

Python/HTTPS server script: [https://github.com/collinsmc23/simple-https-python-server/blob/main/simple_https.py](https://github.com/collinsmc23/simple-https-python-server/blob/main/README.md)

Detailed tutorial on how to write the above Python script: https://anvileight.com/blog/posts/simple-python-http-server/
