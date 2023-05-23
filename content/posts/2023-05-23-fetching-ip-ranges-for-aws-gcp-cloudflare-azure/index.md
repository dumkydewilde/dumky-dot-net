---
title: "Fetching IPv4 CIDR ranges from AWS, GCP, Azure and Cloudflare for bot detection with Python"
date: "2023-05-17"
tags: 
  - bots
  - cloud
  - AWS
  - GCP
  - Azure
  - Cloudflare
  - Python
description: "Bots usually run on one of the major cloud providers. Identifying them can be a big factor in determining the quality of your traffic.
Whether that's for web analytics or threat mitigation, it's useful to have an overview of IP ranges to identify in bot scoring." 
---

Bots are a big part of the internet. Some are legitimate, like the Google Search crawler, but others have more nefarious purposes, like 
scanning for vulnerabilities. If you want to understand which part of your traffic is related to bots or automation, you can use
a couple of attributes to identify this traffic. The user agent is probably the most important one, but also the referrer URL 
or page path can be an identifier when it's used for referrer spam for example. But while a user agent can easily be spoofed, 
one of the most important attributes in bot detection that's not spoofable is the IP address. 

If you want to use bot detection for web analytics, most crawlers or vulnerability scanners might not be a big problem 
as they are usually not using JavaScript to render your website and so they won't trigger any of your web analytics trackers.
But nonetheless there will be a big part of traffic —if you have interesting data— that will try to scan your site using 
tools like Playwright or Puppeteer. Those scripts will often be running in a simple AWS lambda function or cloud container 
on GCP. Luckily that makes them easily identifiable for us. 

The following script gives us a range of IP addresses from the different cloud providers, in the CIDR format. I won't go
into details of how CIDR notation works, but in short the first part will give you a starting point of the range, while the 
part after the `/` will give you the number of IPs in range by identifying the number of bits in the IP address that are masked.
In other words, for the range `192.168.0.0` to `192.168.255.255` the first 16 bits (`192.168`) are the same. This range can 
be written in CIDR notation as `192.168.0.0/16` and will give you more than 65000 IP addresses. On the other hand `10.0.0.1/32` 
will give you one specific IP address only since all 32 bits are masked.

Every cloud provider has their own way of (sometimes not really) providing their IP ranges. So we try to bring these all 
together by accessing their URLs and then matching our IP address of choice to this range using the ipaddress library.

```Python
import requests, json, ipaddress

def get_cidr_ranges(provider):
    if provider == 'aws':
        url = 'https://ip-ranges.amazonaws.com/ip-ranges.json'
        base_key = 'prefixes'
        key = 'ip_prefix'
    elif provider == 'gcp':
        url = 'https://www.gstatic.com/ipranges/cloud.json'
        base_key = 'prefixes'
        key = 'ipv4Prefix'
    elif provider == 'github':
        url = 'https://api.github.com/meta'
        base_key = 'actions'
        key = None
    elif provider == 'cloudflare':
        url = 'https://www.cloudflare.com/ips-v4'
        base_key = None
        key = None
    elif provider == 'azure':
        url = 'https://download.microsoft.com/download/7/1/D/71D86715-5596-4529-9B13-DA13A5DE5B63/ServiceTags_Public_20230522.json'
        base_key = None
        key = None
    

    if base_key is not None:
        data = requests.get(url).json()
        if key is not None:
            return [x[key] for x in data[base_key] if key in x]
        else:
            return [x for x in data[base_key] if ":" not in x]

    elif provider == 'cloudflare':
        return [line.strip() for line in requests.get(url).text.splitlines()]

    elif provider == 'azure':
        data = requests.get(url).json()
        ipv4_list = []
        for region in data["values"]:
            ipv4_list.extend([x for x in region["properties"]["addressPrefixes"] if ":" not in x])
        return list(set(ipv4_list))

# Write all to file for later use
with open("ipv4-cidrs.txt", "w") as f:
	f.write(",".join(get_cidr_ranges('aws')))
	f.write(",".join(get_cidr_ranges('gcp')))
	f.write(",".join(get_cidr_ranges('cloudflare')))
	f.write(",".join(get_cidr_ranges('azure')))


# Alternatively get all the CIDRs and check if our IP matches any CIDR
cidr_list = []
cidr_list.extend(get_cidr_ranges('aws'))
cidr_list.extend(get_cidr_ranges('gcp'))
cidr_list.extend(get_cidr_ranges('cloudflare'))
cidr_list.extend(get_cidr_ranges('azure'))

my_ip = '3.2.34.1' # AWS
is_bot = any([ipaddress.ip_address(my_ip) in ipaddress.ip_network(cidr) for cidr in cidr_list])

print(f"Is {my_ip} a bot? {is_bot}")


```