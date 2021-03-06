# tor_config_stem_etc


BROKEN 



Auto config to deploy tor to a vps and configure it to use the stem control and other things

-------------------------------------------------------------------------------------------------------------------------------------


Run as root

```sh
apt-get install -y git
git clone https://github.com/richarddennis/tor_config_stem_etc.git
cd tor_config_stem_etc
chmod 755 bootstrap.sh
./bootstrap.sh
sudo apt-get install python-stem
sudo apt-get install privoxy
sudo apt-get install -y gedit 
sudo gedit /etc/privoxy/config
forward-socks5 / localhost:9050
sudo /etc/init.d/privoxy restart
python test_tor_stem_privoxy.py
reboot
```




------------------------------------------------------------------------------------------------------------------------------------

Differences from the other setup scripts


- Enable the ControlPort listener for Tor to listen on port 9051, as this is the port to which Tor will listen for any communication from applications talking to the Tor controller.
- Hash a new password that prevents random access to the port by outside agents.
- Implement cookie authentication as well.

You can create a hashed password out of your password using:

```shell
tor --hash-password my_password
```



```shell
ControlPort 9051
# hashed password below is obtained via `tor --hash-password my_password`
HashedControlPassword 16:4B686E879FB96E1460637C8FCB6FEBAEF9C88CCDA997027AE52F2B22C8
CookieAuthentication 1
```

##privoxy##

Tor itself is not a http proxy. So in order to get access to the Tor Network, use `privoxy` as an http-proxy though socks5.

Install `privoxy` via the following command:
	
```shell
sudo apt-get install privoxy
```

Now, tell `privoxy` to use TOR by routing all traffic through the SOCKS servers at localhost port 9050.

```shell
sudo gedit /etc/privoxy/config
```

and enable `forward-socks5` as follows:
	
```shell
forward-socks5 / localhost:9050
```

Restart `privoxy` after making the change to the configuration file.
	
```shell
sudo /etc/init.d/privoxy restart
```

##Python Script##

In the script below, `urllib2` is using the proxy. `privoxy` listens on port 8118 by default, and forwards the traffic to port 9050 upon which the Tor socks is listening.

Additionally, in the `renew_connection()` function,  a signal is being sent to the Tor controller to change the identity, so you get new identities without restarting Tor. Doing such comes in handy when crawling a web site and one doesn’t wanted to be blocked based on IP address.

**test_tor_stem_privoxy.py**

```python

import stem
import stem.connection

import time
import urllib2

from stem import Signal
from stem.control import Controller

# initialize some HTTP headers
# for later usage in URL requests
user_agent = 'Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.9.0.7) Gecko/2009021910 Firefox/3.0.7'
headers={'User-Agent':user_agent}

# initialize some
# holding variables
oldIP = "0.0.0.0"
newIP = "0.0.0.0"

# how many IP addresses
# through which to iterate?
nbrOfIpAddresses = 3

# seconds between
# IP address checks
secondsBetweenChecks = 2

# request a URL 
def request(url):
    # communicate with TOR via a local proxy (privoxy)
    def _set_urlproxy():
        proxy_support = urllib2.ProxyHandler({"http" : "127.0.0.1:8118"})
        opener = urllib2.build_opener(proxy_support)
        urllib2.install_opener(opener)

    # request a URL
    # via the proxy
    _set_urlproxy()
    request=urllib2.Request(url, None, headers)
    return urllib2.urlopen(request).read()

# signal TOR for a new connection 
def renew_connection():
    with Controller.from_port(port = 9051) as controller:
        controller.authenticate(password = 'my_password')
        controller.signal(Signal.NEWNYM)
        controller.close()

# cycle through
# the specified number
# of IP addresses via TOR 
for i in range(0, nbrOfIpAddresses):

    # if it's the first pass
    if newIP == "0.0.0.0":
        # renew the TOR connection
        renew_connection()
        # obtain the "new" IP address
        newIP = request("http://icanhazip.com/")
    # otherwise
    else:
        # remember the
        # "new" IP address
        # as the "old" IP address
        oldIP = newIP
        # refresh the TOR connection
        renew_connection()
        # obtain the "new" IP address
        newIP = request("http://icanhazip.com/")

    # zero the 
    # elapsed seconds    
    seconds = 0

    # loop until the "new" IP address
    # is different than the "old" IP address,
    # as it may take the TOR network some
    # time to effect a different IP address
    while oldIP == newIP:
        # sleep this thread
        # for the specified duration
        time.sleep(secondsBetweenChecks)
        # track the elapsed seconds
        seconds += secondsBetweenChecks
        # obtain the current IP address
        newIP = request("http://icanhazip.com/")
        # signal that the program is still awaiting a different IP address
        print ("%d seconds elapsed awaiting a different IP address." % seconds)
    # output the
    # new IP address
    print ("")
    print ("newIP: %s" % newIP)

```

Execute the Python 2.7 script above via the following command:
	
```shell
python test_tor_stem_privoxy.py
```

When the above script is executed, one should see that the IP address is changing every few seconds.
