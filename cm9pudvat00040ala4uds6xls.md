---
title: "blog 2 dsdssd - uu"
datePublished: Sun Apr 20 2025 16:09:01 GMT+0000 (Coordinated Universal Time)
cuid: cm9pudvat00040ala4uds6xls
slug: blog-2-dsdssd-uu

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