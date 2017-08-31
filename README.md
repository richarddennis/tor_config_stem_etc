# tor_config_stem_etc

Auto config to deploy tor to a vps and configure it to use the stem control and other things

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
