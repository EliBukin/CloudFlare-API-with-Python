# CloudFlare-API-with-Python
Update CloudFlare DNS A record Using API

In this tutorial I will demonstrate how to automatically update an A record of a DNS name that is handled by CloudFlare.
the use case is if you have a DNS pointing to your IP but the IP is not static and can be changed every now and then.

For this we will use python3.
https://github.com/EliBukin/CloudFlare-API-with-Python.git

First step is to create our configuration file that will be addressed by our script, open a terminal and create a yaml file that will store the changing and secret data,
that approach will prevent us hardcoding the data in to the script.

touch configuration.yaml

open the file and paste the following content:

cfauth:
    AuthEmail: AUTH-EMAIL
    AuthKey: AUTH-KEY
    ContentType: application/json
    getcookie: GET-COOKIE
    putcookie: PUT-COOKIE
    zone: ZONE-ID
    dnsrecordid: DNS-RECORD-ID
    payloadname: DOMAIN-NAME

the next thing to do is create our python file:

import json
import yaml
import requests

after importing the modules lets create a variable block and point our variables to where they are stored (in the config file):

with open("configuration.yaml", "r") as ymlfile:
    configfile = yaml.safe_load(ymlfile)

AuthEmail = configfile['cfauth']['AuthEmail']
AuthKey = configfile['cfauth']['AuthKey']
ContentType = configfile['cfauth']['ContentType']
getcookie = configfile['cfauth']['getcookie']
putcookie = configfile['cfauth']['putcookie']
zone = configfile['cfauth']['zone']
dnsrecordid = configfile['cfauth']['dnsrecordid']
payloadname = configfile['cfauth']['payloadname']

when that done we will create our update function that will be used only if our IP was changed:

### update function to update CloudFlare with new IP
def updatedns():

    url = "https://api.cloudflare.com/client/v4/zones/"+zone+"/dns_records/"+dnsrecordid+""

    payload = "{\n \"type\":\"A\",\n \"name\":\""+payloadname+"\",\n \"content\":\""+newip.strip()+"\",\n \n \"ttl\":120,\n \"proxied\":true\n}"
    headers = {
    'X-Auth-Email': AuthEmail,
    'X-Auth-Key': AuthKey,
    'Content-Type': ContentType,
    'Cookie': putcookie
    }
    response = requests.request("PUT", url, headers=headers, data = payload)
    print(response.text.encode('utf8'))

from that point we will create our API request to get currently updated IP in the A record, and that will be represented as ‘oldip’:

### get the old CloudFlare (current updated) IP
url = "https://api.cloudflare.com/client/v4/zones/"+zone+"/dns_records/"+dnsrecordid+""

payload = {}
headers = {
  'X-Auth-Email': AuthEmail,
  'X-Auth-Key': AuthKey,
  'Content-Type': ContentType,
  'Cookie': getcookie
}
response = requests.request("GET", url, headers=headers, data = payload)
#print(response.text.encode('utf8'))
###
result = json.loads(response.text)
#print(result)
oldip = result['result']['content']

once we have our old (current CloudFlare value) IP lets check what is our current IP and compare them, and if they are not identical update function will be executed:

### compare CloudFlare IP with fresh fetched IP, and if not identical then update
newip = requests.get("https://ipinfo.io/ip").text

print(newip.strip())
if newip.strip() == oldip:
    print ("IP unchanged it is still", oldip)
else:
    print ('IP changed to', newip.strip(), 'updating CloudFlare DNS A record with new IP...')
    updatedns()

the complete script should look like this:

#!/usr/bin/python3

import json
import yaml
import requests

with open("configuration.yaml", "r") as ymlfile:
    configfile = yaml.safe_load(ymlfile)

AuthEmail = configfile['cfauth']['AuthEmail']
AuthKey = configfile['cfauth']['AuthKey']
ContentType = configfile['cfauth']['ContentType']
getcookie = configfile['cfauth']['getcookie']
putcookie = configfile['cfauth']['putcookie']
zone = configfile['cfauth']['zone']
dnsrecordid = configfile['cfauth']['dnsrecordid']
payloadname = configfile['cfauth']['payloadname']

### update function to update CloudFlare with new IP
def updatedns():

    url = "https://api.cloudflare.com/client/v4/zones/"+zone+"/dns_records/"+dnsrecordid+""

    payload = "{\n \"type\":\"A\",\n \"name\":\""+payloadname+"\",\n \"content\":\""+newip.strip()+"\",\n \n \"ttl\":120,\n \"proxied\":true\n}"
    headers = {
    'X-Auth-Email': AuthEmail,
    'X-Auth-Key': AuthKey,
    'Content-Type': ContentType,
    'Cookie': putcookie
    }

    response = requests.request("PUT", url, headers=headers, data = payload)

    print(response.text.encode('utf8'))

### get the old CloudFlare (current updated) IP
url = "https://api.cloudflare.com/client/v4/zones/"+zone+"/dns_records/"+dnsrecordid+""

payload = {}
headers = {
  'X-Auth-Email': AuthEmail,
  'X-Auth-Key': AuthKey,
  'Content-Type': ContentType,
  'Cookie': getcookie
}

response = requests.request("GET", url, headers=headers, data = payload)
#print(response.text.encode('utf8'))

###
result = json.loads(response.text)
#print(result)

oldip = result['result']['content']
#print(oldip)

### compare CloudFlare IP with fresh fetched IP, and if not identical then update
newip = requests.get("https://ipinfo.io/ip").text

print(newip.strip())
if newip.strip() == oldip:
    print ("IP unchanged, it is still", oldip)
else:
    print ('IP changed to', newip.strip(), 'updating CloudFlare DNS A record with new IP...')
    updatedns()

### 

OK, now we can run our script from the terminal with the following command:

python3 ./cf-api-dns-udpate.py

But, running this script manually misses the purpose…
Now lets schedule that to run as a cron job and output to a logfile.

cron tab will run a “middleman” tiny little bash script that will call our python script and throw the output to a log file.

create a bash file:

touch cf-api-dns-update.sh

open the file and populate with the following code:

#!/bin/sh
#
touch <PATH-TO-LOGFILE>cf-log.txt
cflog="<PATH-TO-LOGFILE>cf-log.txt"
#
date &>> $cflog
python3 <PATH-TO-PYTHON-SCRIPT>cf-api-dns-udpate.py &>> $cflog
echo "" >> $cflog

touch will create our logfile if not exists.
cflog is a variable that stores the path to logfile.
date will write the date and time of an execution.
python3 will execute our script.
echo “” will write an empty line between the logged sessions.

next, open your crontab:

sudo crontab -e

and add the following line to execute our job every 30 minutes:

*/30 * * * * bash cf-api-dns-update.sh

the last step is to make our files executable, that we will do with the following commands:

sudo chmod -x cf-api-dns-update.sh
sudo chmod -x cf-api-dns-update.py

and that’s about it… now your DNS A record will be updated with a new IP in case that will change.
