---
date: 2024-08-21T17:56:43+07:00
title: "Setup Hugo Blog on Vps"
description:
tldr: Guide for setting up hugo blog on vps
tags: []
draft: true
toc: false
---

Hi. This is a guide on how to setup a hugo blog on vps. It will be simple and it is how i host this site h0neybun.com

## requirements
first of all you need this stuff:
- a domain. i buy it from cloudflare
- a vps. a cheap one will do

## setup ssl for nginx
In this guide i will be using Letsencrypt with wildcard ssl. The main reason is because its free and widely used. I also want to add wildcard ssl so it also covers all the subdomains (\*.honeybun.com) for my site in addition to the honeybun.com. 

Usually i would use Let's encrypt and do renewal every 3 months. This is easy to script using certbot's plugins of your domain registrar and automated using cron. However since i moved to cloudflare (rip google domains, im not using squarespace sorry) i noticed that cloudflare provide a free ssl cert that can last a long time without renewal (15 years!). But theres a catch and its a bit different. note: if you wish to use let's encrypt instead on cloudflare just turn off the 'proxied' option in the dns entry.


```
Cloudflare SSL                                   
┌─────────┐      ┌────────────┐        ┌────────┐
│ browser ┼──┬───► cloudflare ┼───┬────► origin │
└─────────┘  │   └────────────┘   │    │ server │
             │                    │    └────────┘
           edge               origin             
           certificate        certificate        
                                                 
                                                 
Let's Encrypt SSL                                
      ┌─────────┐            ┌─────────┐         
      │ browser ┼─────┬──────► origin  │         
      └─────────┘     │      │ server  │         
                      │      └─────────┘         
                 let's encrypt                   
                 certificate                     
```

In letsencrypt theres a single certificate that secures communication between user and origin server. But in cloudflare there are 2 legs of ssl certs, edge and origin:
- edge certificate encrypts traffic between your end user's browser with cloudflare.
- origin certificate encrypts the traffic between your origin server(vps) with cloudflare


There are multiple encryption modes that is described [here](https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/#custom-ssltls) but heres the short version:
- using off and flexible is a bad idea for prod but acceptable for dev cause its convenient.
- full(strict), and full(ssl-only origin pull) is preferred for production. IMO full is not preferred because although it requires to use an origin cert it is not enforced meaning ANY cert will do (self signed, expired, or anything really will work).


After looking into it i decided to use full(strict) encryption and I will use the cert provided by cloudflare. Okay now lets get into it!
1. Navigate thru the menu `Domain Name > SSL/TLS > Origin Server > Create Certificate`
2. Choose `Generate Private key and CSR with Cloudflare` - i chose not to bother with these and let CF do it for me
3. In the hostnames make sure theres 2 entries: your domain and wildcard. for me the value is `*.h0neybun.com, h0neybun.com`
4. Choose 15 years for `Certificate Validity` - no renewal at least for the next 1.5 decade, yay
5. Click create and save your certificate and private key. note: save those files now because you can always redownload your origin certs but NOT your private key.

And its done! on to the next part

## setup dns records
probably i should've done this first but it doesnt matter. For now ill just set up DNS for `h0neybun.com` and `www.h0neybun.com`. Here are the steps:
1. Navigate thru the menu `Domain Name > DNS > Records `
2. Add these records:
| type  | name             | content                 | proxy status | ttl  |
|-------|------------------|-------------------------|--------------|------|
| A     | @                | {your server public ip} | proxied      | auto |
| CNAME | www.h0neybun.com | h0neybun.com            | proxied      | auto |

Now lets wait for a while for the dns records to be updated and we can check it using whois. When completed it will shows that the domain is registered and it's registrar is cloudflare like this:
```bash
reisa@legion:~/projects/ssl$ whois h0neybun.com | grep "Registry database" -A 10
The Registry database contains ONLY .COM, .NET, .EDU domains and
Registrars.
Domain Name: H0NEYBUN.COM
Registry Domain ID: 2908736670_DOMAIN_COM-VRSN
Registrar WHOIS Server: whois.cloudflare.com
Registrar URL: https://www.cloudflare.com
Updated Date: 2024-08-22T09:41:57Z
Creation Date: 2024-08-17T07:56:14Z
Registrar Registration Expiration Date: 2025-08-17T07:56:14Z
Registrar: Cloudflare, Inc.
Registrar IANA ID: 1910
reisa@legion:~/projects/ssl$
```

## setup nginx + server
heres generally how the nginx config is
```conf
server {
  listen 80;
  server_name h0neybun.com www.h0neybun.com;
  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl;
  server_name h0neybun.com www.h0neybun.com;

  ssl_certificate /{path to origin cert}
  ssl_certificate_key /{path to private key}

  root /{path to blog}
  index index.html;

  location / {
    try_files $uri $uri/ =404;
  }

  location /static/ {
    autoindex on;
  }

}
```

## setup hugo 
welp [im not gonna sugarcoat it](https://images-wixmp-ed30a86b8c4ca887773594c2.wixmp.com/f/2913b5a9-6192-4abf-a365-977a20e4afa9/dg7itd7-3a080063-bd0f-45da-87ab-c605e262d46b.jpg?token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1cm46YXBwOjdlMGQxODg5ODIyNjQzNzNhNWYwZDQxNWVhMGQyNmUwIiwiaXNzIjoidXJuOmFwcDo3ZTBkMTg4OTgyMjY0MzczYTVmMGQ0MTVlYTBkMjZlMCIsIm9iaiI6W1t7InBhdGgiOiJcL2ZcLzI5MTNiNWE5LTYxOTItNGFiZi1hMzY1LTk3N2EyMGU0YWZhOVwvZGc3aXRkNy0zYTA4MDA2My1iZDBmLTQ1ZGEtODdhYi1jNjA1ZTI2MmQ0NmIuanBnIn1dXSwiYXVkIjpbInVybjpzZXJ2aWNlOmZpbGUuZG93bmxvYWQiXX0.FcwVAqhvWLPQ7WxiglGMl-lNTCtogHvXkknNvXjkj6k), i didnt make my own blog site. I use this theme called [archie](https://themes.gohugo.io/themes/archie/) and it is nice and clean. You can see the sauce code for this site [here](https://github.com/h0neybun/h0neybun.com)

hugo is a static site generator. It builds the site into a static html css file so its lightweight and fast. to deploy it i use this script from my localhost (make sure the path is owned by ssh user):
```bash
hugo && rsync -avz --delete ./public/ {server}:{path to blog}
```



