---
title: "Access your API (Django) running on Localhost from Android Device"
seoTitle: "Call your API endpoint from your phone browser or and app you might be"
seoDescription: "Recently I was building a flutter app and was using Django to build its backend. I wanted to call by APIs from my app which was running on my phone after se"
datePublished: Sat Feb 25 2023 16:28:46 GMT+0000 (Coordinated Universal Time)
cuid: clek6fkq5000l0akwcfcxbna8
slug: access-your-api-django-running-on-localhost-from-android-device
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1677342900943/5716b22e-e42c-4169-94b0-2d0cddfe71ec.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1677342920325/6325c239-40dc-4868-9eb8-76c39fe76ee4.png
tags: django, flutter, apis

---

%[https://media.giphy.com/media/flbcUFdLSHwZC03p11/giphy.gif] 

Recently I was building a flutter app and was using Django to build its backend. I wanted to call by APIs from my app which was running on my phone after searching the net I finally found the way!!.

### Step 1

First, we need to make sure that our computer and phone are connected to the same Wifi (or you can connect your computer to your mobile HotSpot)

### Step 2

Find the IP address of your Desktop

For Type in ComandLine Windows

```bash
ipconfig
```

For Linux/mac Type in terminal

```bash
ifconfig
```

The command shall show your IP Address (Example = 192.168.20.22)

### Step 3

Normally when we run our Django server using

```python
python manage.py runserver
```

It runs on localHost( 127.0.0.1) which we cannot access from another device so we run our server on our computers IP Address in Django we do that by-

```python
python manage.py runserver 0.0.0.0:8000
```

This command will run our server on our computer so that other devices on the same network can also access the server. Also, don't forget to add your IP address in the ALLOWED\_HOSTS in the setting.py file of your Django project.

### Step 4

Now open any browser and open http://YOUR\_IP:8000/api\_endpoint

example:= http://192.168.20.22/my\_site and you will be able to see the expected result

### Conclusion

The Above example is for Django but could be done on any other framework all you have to do is figure out how to run your server on your machine IP address and connect to the same network. You can use the URL with the IP address and call from any application you are building