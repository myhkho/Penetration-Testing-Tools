## malleable-redirector - a proxy2 plugin

**Let's raise the bar in C2 redirectors IR resiliency, shall we?**

Red Teaming business has seen [several](https://bluescreenofjeff.com/2016-04-12-combatting-incident-responders-with-apache-mod_rewrite/) [different](https://posts.specterops.io/automating-apache-mod-rewrite-and-cobalt-strike-malleable-c2-profiles-d45266ca642) [great](https://gist.github.com/curi0usJack/971385e8334e189d93a6cb4671238b10) ideas on how to combat incident responders and misdirect them while offering resistant C2 redirectors network at the same time.  

This piece of code tries to combine many of these great ideas into a one, lightweight utility, mimicking Apache2 in it's roots of being a simple HTTP(S) reverse-proxy. Combining Malleable C2 profiles understanding, knowledge of bad IP addresses pool and a flexibility of easily adding new inspection and misrouting logc - resulted in having a crafty repellent for IR evasion. 

### Abstract

This program acts as a HTTP/HTTPS reverse-proxy with several restrictions imposed upon which requests and from whom it should process, similarly to the .htaccess file in Apache2's mod_rewrite.

`malleable_redirector` was created to resolve the problem of effective IR/AV/EDRs/Sandboxes evasion on the C2 redirector's backyard. It comes in a form of a plugin for other project of mine called [proxy2](https://github.com/mgeeky/proxy2), that is a lightweight forward & reverse HTTP/HTTPS proxy.

** Features:**

- Malleable C2 Profile parser able to validate inbound HTTP/S requests strictly according to malleable's contract and drop in case of violation (Malleable Profiles 4.0+ with variants covered)
- Integrated curated massive blacklist of IPv4 pools and ranges known to be associated with IT Security vendors
- Ability to query connecting peer's IPv4 address against IP Geolocation/whois information and confront that with predefined regular expressions to rule out peers connecting outside of trusted organizations/countries/cities etc.
- Built-in Replay attacks mitigation enforced by logging accepted requests' MD5 hashsums into locally stored SQLite database and preventing requests previously accepted.
- Functionality to ProxyPass requests matching specific URL onto other Hosts
- Support for multiple Teamservers
- Support for many reverse-proxying Hosts/redirection sites giving in a randomized order - which lets load-balance traffic or build more versatile infrastructures

The proxy2 in companion with this plugin can act as a CobaltStrike Teamserver C2 redirector, given Malleable C2 profile used during the campaign and teamserver's hostname:port. The plugin will parse supplied malleable profile in order to understand which inbound requests may possibly come from the compatible Beacon or are not compliant with the profile and therefore should be misdirected. Sections such as http-stager, http-get, http-post and their corresponding uris, headers, prepend/append patterns, User-Agent are all used to distinguish between legitimate beacon's request and some Internet noise or IR/AV/EDRs out of bound inquiries. 

The plugin was also equipped with marvelous known bad IP ranges coming from:
  curi0usJack and the others:
  [https://gist.github.com/curi0usJack/971385e8334e189d93a6cb4671238b10](https://gist.github.com/curi0usJack/971385e8334e189d93a6cb4671238b10)

Using an IP addresses blacklisting along with known bad keywords lookup through Reverse-IP DNS queries and HTTP headers, the reliability of this tool results considerably increased redirector's resiliency to the unauthorized peers wanting to examine protected infrastructure.

Use wisely, stay safe.

### Example usage

All settings were moved to the external file:
```
$ python3 proxy2.py --config example-config.yaml

  [INFO] 19:21:42: Loading 1 plugin...
  [INFO] 19:21:42: Plugin "malleable_redirector" has been installed.
  [INFO] 19:21:42: Preparing SSL certificates and keys for https traffic interception...
  [INFO] 19:21:42: Using provided CA key file: ca-cert/ca.key
  [INFO] 19:21:42: Using provided CA certificate file: ca-cert/ca.crt
  [INFO] 19:21:42: Using provided Certificate key: ca-cert/cert.key
  [INFO] 19:21:42: Serving http proxy on: 0.0.0.0, port: 80...
  [INFO] 19:21:42: Serving https proxy on: 0.0.0.0, port: 443...
  [INFO] 19:21:42: [REQUEST] GET /jquery-3.3.1.min.js
  [INFO] 19:21:42: == Valid malleable http-get request inbound.
  [INFO] 19:21:42: Plugin redirected request from [code.jquery.com] to [1.2.3.4:8080]
  [INFO] 19:21:42: [RESPONSE] HTTP 200 OK, length: 5543
  [INFO] 19:21:45: [REQUEST] GET /jquery-3.3.1.min.js
  [INFO] 19:21:45: == Valid malleable http-get request inbound.
  [INFO] 19:21:45: Plugin redirected request from [code.jquery.com] to [1.2.3.4:8080]
  [INFO] 19:21:45: [RESPONSE] HTTP 200 OK, length: 5543
  [INFO] 19:21:46: [REQUEST] GET /
  [...]
  [ERROR] 19:24:46: [DROP, reason:1] inbound User-Agent differs from the one defined in C2 profile.
  [...]
  [INFO] 19:24:46: [RESPONSE] HTTP 301 Moved Permanently, length: 212
  [INFO] 19:24:48: [REQUEST] GET /jquery-3.3.1.min.js
  [INFO] 19:24:48: == Valid malleable http-get request inbound.
  [INFO] 19:24:48: Plugin redirected request from [code.jquery.com] to [1.2.3.4:8080]
  [...]
```

Where **example-config.yaml** contains:

```
plugin: malleable_redirector
verbose: True

port:
  - 80/http
  - 443/https

profile: jquery-c2.3.14.profile

# Let's Encrypt certificates
ssl_cacert: /etc/letsencrypt/live/attacker.com/fullchain.pem
ssl_cakey: /etc/letsencrypt/live/attacker.com/privkey.pem

teamserver_url:
  - 1.2.3.4:8080
```

The above output contains a line pointing out that there has been an unauthorized, not compliant with our C2 profile inbound request, which got dropped due to incompatible User-Agent string presented:
```
  [...]
  [DROP, reason:1] inbound User-Agent differs from the one defined in C2 profile.
  [...]
```


### Plugin options

Following options/settings are supported:

```
#
# This is a sample config file for Malleable-Redirector plugin of proxy2 utility.
#


#
# ====================================================
# General proxy related settings
# ====================================================
#

plugin: malleable_redirector

trace: True
debug: True

port:
  - 80/http
  - 443/https

# Let's Encrypt certificates
ssl_cacert: /etc/letsencrypt/live/attacker.com/fullchain.pem
ssl_cakey: /etc/letsencrypt/live/attacker.com/privkey.pem


#
# ====================================================
# malleable_redirector plugin related settings
# ====================================================
#

#
# Path to the Malleable C2 profile file. 
# If not given, most of the request-validation logic won't be used.
#
profile: malleable.profile

#
# (Required) Address where to redirect legitimate inbound beacon requests.
# A.k.a. TeamServer's Listener bind address, in a form of:
#       [inport:][http(s)://]host:port
#
# If proxy2 was configured to listen on more than one port, specifying "inport" will 
# help the plugin decide to which teamserver's listener redirect inbound request. 
#
# If 'inport' values are not specified in the below option (teamserver_url) the script
# will pick destination teamserver at random.
#
# Having proxy2 listening on only one port does not mandate to include the "inport" part.
# This field can be either string or list of strings.
#
teamserver_url: 
  - 1.2.3.4:5555


#
# Report only instead of actually dropping/blocking/proxying bad/invalid requests.
# If this is true, will notify that the request would be block if that option wouldn't be
# set. 
#
# Default: False
#
report_only: False

#
# Log full bodies of dropped requests.
#
# Default: False
#
log_dropped: False

#
# What to do with the request originating not conforming to Beacon, whitelisting or 
# ProxyPass inclusive statements: 
#   - 'redirect' it to another host with (HTTP 301), 
#   - 'reset' a TCP connection with connecting client
#   - 'proxy' the request, acting as a reverse-proxy against specified action_url 
#       (may be dangerous if client fetches something it shouldn't supposed to see!)
#
# Valid values: 'reset', 'redirect', 'proxy'. 
#
# Default: redirect
#
drop_action: redirect

#
# If someone who is not a beacon hits the proxy, or the inbound proxy does not meet 
# malleable profile's requirements - where we should proxy/redirect his requests. 
# The protocol HTTP/HTTPS used for proxying will be the same as originating
# requests' protocol. Redirection in turn respects protocol given in action_url.
#
# This value may be a comma-separated list of hosts, or a YAML array to specify that
# target action_url should be picked at random:
#   action_url: https://google.com, https://gmail.com, https://calendar.google.com
#
# Default: https://google.com
#
action_url: 
  - https://google.com

#
# ProxyPass alike functionality known from mod_proxy.
#
# If inbound request matches given conditions, proxy that request to specified host,
# fetch response from target host and return to the client. Useful when you want to 
# pass some requests targeting for instance attacker-hosted files onto another host, but
# through the one protected with malleable_redirector.
#
# Protocol used for ProxyPass will match the one from originating request. Therefore
# there is no point in prepending host part with http/https schema, as it's going to be
# ignored anyway.
# 
# Syntax:
#   proxy_pass:
#     - /url_to_be_passed example.com
#
# The first parameter 'url' is a regex (case-insensitive). Must start with '/'.
# The begin/end regex operands are implicit and will constitute following regex with URL:
#     '^' + url + '$'
#
# Default: No proxy pass rules.
#
proxy_pass:
  - /foobar\d* bing.com


#
# Every time malleable_redirector decides to pass request to the Teamserver, as it conformed
# malleable profile's contract, a MD5 sum may be computed against that request and saved in sqlite
# file. Should there be any subsequent request evaluating to a hash value that was seen & stored
# previously, that request is considered as Replay-Attack attempt and thus should be banned.
#
# CobaltStrike's Teamserver has built measures aginst replay-attacks, however malleable_redirector may
# assist in that activity.
#
# Default: False
#
mitigate_replay_attack: False


#
# List of whitelisted IP addresses/CIDR ranges.
# Inbound packets from these IP address/ranges will always be passed towards specified TeamServer without
# any sort of verification or validation.
#
whitelisted_ip_addresses:
  - 127.0.0.0/24

#
# Ban peers based on their IPv4 address. The blacklist with IP address to check against is specified
# in 'ip_addresses_blacklist_file' option.
#
# Default: True
#
ban_blacklisted_ip_addresses: True

#
# Specifies external list of CIDRs with IPv4 addresses to ban. Each entry in that file
# can contain a single IPv4, a CIDR or a line with commentary in following format:
#     1.2.3.4/24 # Super Security System
#
# Default: plugins/malleable_banned_ips.txt
#
ip_addresses_blacklist_file: plugins/malleable_banned_ips.txt

#
# Ban peers based on their IPv4 address' resolved ISP/Organization value or other details. 
# Whenever a peer connects to our proxy, we'll take its IPv4 address and use one of the specified
# APIs to collect all the available details about the address. Whenever a banned word 
# (of a security product) is found in those details - peer will be banned.
# List of API keys for supported platforms are specified in ''. If there are no keys specified, 
# only providers that don't require API keys will be used (e.g. ip-api.com, ipapi.co)
#
# Default: True
#
verify_peer_ip_details: True

#
# Specifies a list of API keys for supported API details collection platforms. 
# If 'verify_peer_ip_details' is set to True and there is at least one API key given in this option, the
# proxy will collect details of inbound peer's IPv4 address and verify them for occurences of banned words
# known from various security vendors. Do take a note that various API details platforms have their own
# thresholds for amount of lookups per month. By giving more than one API keys, the script will
# utilize them in a random order.
#
# To minimize number of IP lookups against each platform, the script will cache performed lookups in an
# external file named 'ip-lookups-cache.json'
#
# Supported IP Lookup providers:
#   - ip-api.com: No API key needed, free plan: 45 requests / minute
#   - ipapi.co: No API key needed, free plan: up to 30000 IP lookups/month and up to 1000/day.
#   - ipgeolocation.io: requires an API key, up to 30000 IP lookups/month and up to 1000/day.
#
# Default: empty dictionary
#
ip_details_api_keys: 
  #ipgeolocation_io: 0123456789abcdef0123456789abcdef
  ipgeolocation_io:


#
# Restrict incoming peers based on their IP Geolocation information. 
# Available only if 'verify_peer_ip_details' was set to True. 
# IP Geolocation determination may happen based on the following supported characteristics:
#   - organization, 
#   - continent, 
#   - continent_code, 
#   - country, 
#   - country_code, 
#   - city, 
#   - timezone
#
# The Peer will be served if at least one geolocation condition holds true for him 
# (inclusive/alternative arithmetics).
#
# If no determinants are specified, IP Geolocation will not be taken into consideration while accepting peers.
# If determinants are specified, only those peers whose IP address matched geolocation determinants will be accepted.
#
# Each of the requirement values may be regular expression. Matching is case-insensitive.
#
# Following (continents_code, continent) pairs are supported:
#    ('AF', 'Africa'),
#    ('AN', 'Antarctica'),
#    ('AS', 'Asia'),
#    ('EU', 'Europe'),
#    ('NA', 'North america'),
#    ('OC', 'Oceania'),
#    ('SA', 'South america)' 
#
# Proper IP Lookup details values can be established by issuing one of the following API calls:
#   $ curl -s 'https://ipapi.co/TARGET-IP-ADDRESS/json/' 
#   $ curl -s 'http://ip-api.com/json/TARGET-IP-ADDRESS'
#
# The organization/isp/as/asn/org fields will be merged into a common organization list of values.
#
ip_geolocation_requirements:
  organization:
    #- My\s+Target\+Company(?: Inc.)?
  continent:
  continent_code:
  country:
  country_code:
  city:
  timezone:

#
# Fine-grained requests dropping policy - lets you decide which checks
# you want to have enforced and which to skip by setting them to False
#
# Default: all checks enabled
#
policy:
  # [IP: ALLOW, reason:0] Request conforms ProxyPass entry (url="..." host="..."). Passing request to specified host
  allow_proxy_pass: True
  # [IP: DROP, reason:1] inbound User-Agent differs from the one defined in C2 profile.
  drop_invalid_useragent: True
  # [IP: DROP, reason:2] HTTP header name contained banned word
  drop_http_banned_header_names: True
  # [IP: DROP, reason:3] HTTP header value contained banned word:
  drop_http_banned_header_value: True
  # [IP: DROP, reason:4b] peer's reverse-IP lookup contained banned word
  drop_dangerous_ip_reverse_lookup: True
  # [IP: DROP, reason:5] HTTP request did not contain expected header
  drop_malleable_without_expected_header: True
  # [IP: DROP, reason:6] HTTP request did not contain expected header value:
  drop_malleable_without_expected_header_value: True
  # [IP: DROP, reason:7] HTTP request did not contain expected (metadata|id|output) section header:
  drop_malleable_without_expected_request_section: True
  # [IP: DROP, reason:8] HTTP request was expected to contain (metadata|id|output) section with parameter in URI:
  drop_malleable_without_request_section_in_uri: True
  # [IP: DROP, reason:9] Did not found append pattern:
  drop_malleable_without_prepend_pattern: True
  # [IP: DROP, reason:10] Did not found append pattern:
  drop_malleable_without_apppend_pattern: True
```


### TODO:

- Implement support for JA3 signatures in both detection & blocking and impersonation to fake nginx/Apache2/custom setups.
- Add some unique beacons tracking logic to offer flexilibity of refusing staging and communication processes at the proxy's own discretion
- Introduce day of time constraint when offering redirection capabilities
- Keep track of metadata/ID payloads to better distinguish connecting peers and avoid replay attack consequences
- Test it thoroughly with several enterprise-grade EDRs, Sandboxes and others 
- Add Proxy authentication and authorization logic on CONNECT/relay.
- Add Mobile users targeted redirection

### Author

Mariusz B. / mgeeky, '20
<mb@binary-offensive.com>

