# WARP: Workspace Autonomous Reservation Program

The story of this project begins when, due to COVID-19, we have converted our regular office into a hybrid of regular and hot-desk assignments. We needed to find a solution for desk reservations, transparency of that, and detailed logging of who was in the office for epidemic purposes.

I've quickly evaluated a couple of existing solutions, but they were either too big and complicated and/or too expensive. As I assumed that other people would have the same challenge I had, I decided to spend my after-hours time making an open-source tailored system for the need. Yes - it is free as speech, not as beer.

## What WARP can do
- It allows people to book / change / unbook desks (or even parking stalls) in the office.
- It allows people to check who else will be in the office.
- It works on mobile.
- All is done in an easy, visual way.
- Generate a report of past bookings and export it to Excel file

## More advanced features
- Seats can be limited to certain people, so other people cannot book them (it is called assigned seats).
- Seats can be disabled, so people don't see them at all.
- Multiple zones (maps) can be created, for example, floors or parking.
- Zones can be grouped. One person can have only one seat booked simultaneously in a zone group (so you can have one group for floors and another group for parking stalls).
- Admin(s) can book / modify / unbook seat for any user.
- Full admin interface to add/remove/edit maps, zones, groups, and users.
- SAML2.0 support - via Apache [mod_auth_mellon](https://github.com/latchset/mod_auth_mellon) module.
- LDAP and Active Directory - via LDAP3 library.
- Translations - currently, English and Polish are supported.

## What I'm not even planning to do
- Approvals - the main goal of the system was to make it autonomous and management-free. So I don't intend to implement approval flows.
- Timezone support - the selected time is always in the same timezone as a zone. It works well and is simple. But in case someone would like to have a couple of zones in different timezones and keep the `one person one seat at a given time` rule across these timezones, this will fail.

## What browsers are supported
To be honest, I was not paying much attention to browser compatibility, nor was I extensively testing it on other browsers than Chrome and Firefox. Nevertheless, all modern browsers should be supported (definitely not IE).

## Is there any demo?

![demo animation](res/demo.gif)

It is so easy to run it via docker compose that I have removed the demo, which was available some time ago.

# Deployment

During the first run on an empty database, WARP will populate the database schema and create an admin user.

Default admin credentials are: `admin:noneshallpass`

## Demo quickstart

The preferred way to deploy is to run it via Docker. You need a working docker, and I won't cover it here.

### docker compose

From the command line:
```
# clone the repository
$ git clone https://github.com/sebo-b/warp.git
$ cd warp

$ docker compose -f demo_compose.yaml up
```

After that, open http://127.0.0.1:8080 in your browser and log in as `admin` with password `noneshallpass`.

### without docker compose (but why?)

From the command line:
```
# clone the repository
$ git clone https://github.com/sebo-b/warp.git
$ cd warp

# build docker image (you can skip hash if you don't want to track it)
$ export GIT_HASH=`git log -1 --format=%h`
$ docker build -t warp:latest -t warp:$GIT_HASH .

# install postrgres (what I cover here is a simplistic way just to run a demo)
$ docker pull postgres
$ docker run --name warp-demo-db -e POSTGRES_PASSWORD=postgres_password -d postgres
$ export WARP_DEMO_DB_IP=`docker inspect  -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' warp-demo-db`

# start warp
$ docker run --name warp-demo-wsgi \
> --env 'WARP_DATABASE=postgresql://postgres:postgres_password@warp-demo-db:5432/postgres' \
> --env WARP_SECRET_KEY=mysecretkey \
> --env WARP_DATABASE_INIT_SCRIPT='["sql/schema.sql","sql/sample_data.sql"]' \
> --add-host=warp-demo-db:${WARP_DEMO_DB_IP} -d warp:latest
$ export WARP_DEMO_WSGI_IP=`docker inspect  -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' warp-demo-wsgi`

# install nginx as wsgi rewerse proxy
$ docker pull nginx
$ docker run --add-host=warp-demo-wsgi:${WARP_DEMO_WSGI_IP} --mount type=bind,source="$(pwd)"/res/nginx.conf,target=/etc/nginx/conf.d/default.conf,readonly -d -p 127.0.0.1:8080:80 nginx
```

After that, open http://127.0.0.1:8080 in your browser and log in as `admin` with password `noneshallpass`.

### without Docker - the old way

You need a working Python3 environment, Node.js, and PostgreSQL, and I won't cover it here. This is not a preferred way, use it only for debugging or development purposes. Things may change, and this section can be outdated - but I assume that you know what you are doing.

From the command line:

```
# clone repo
$ git clone https://github.com/sebo-b/warp.git
$ cd warp

# create virtual envirnoment and activate it
$ python3 -m venv --prompt warp .venv
$ source .venv/bin/activate

# install python requirements
# if this raises an error in psycopg2, either install its all build dependencies
# or change psycopg2 to psycopg2-binary in requirements.txt
$ pip install -r requirements.txt

# compile JavaScript files
$ pushd js
$ npm ci
$ npm run build
$ popd

# setup Flask and database URL
$ export FLASK_APP=warp
$ export FLASK_ENV=development
$ export WARP_DATABASE=postgresql://warp:warp@localhost:5432/warp

# run the app
$ flash run
```

After that, open http://127.0.0.1:5000 in your browser and log in as `admin` with password `noneshallpass`.

## Production environment

For the production envirnoment, I recommend running Nginx and PostgreSQL on separate VMs. Then (even multiple) WARP image can be simply started via Docker and rev-proxed from Nginx.

Each configuration parameter (check config.py) can be passed via the envirnoment as `WARP_varname`.

### SECRET_KEY

For the production environment, **make sure** that you have generated `SECRET_KEY` used for signing cookies. It is defined in `config.py.`

Flask documentation mentions this method to generate it:
```
$ python -c 'import os; print(os.urandom(16))'
```

Alternatively, you can use OpenSSL and Sed:
```
$ openssl rand -hex 16 | sed 's/\(..\)/\\x\1/g;s/^/b"/;s/$/"/'
```
or wrap it into Python:
```
$ python -c 'from subprocess import run; print(run(["openssl","rand","16"],capture_output=True).stdout)'
```

### Language

Change `LANGUAGE_FILE` variable in `config.py` or set `WARP_LANGUAGE_FILE` environment variable. Currently, language is global for the instance.

### *Active Directory* (or any other LDAP) authentication
Authentication via LDAP server like Active Directory, it is an easy way to allow your LDAP directory users login on your WARP instalation. 

Supported configurations to connect LDAP server are:
- **LDAP protocol:** *LDAP* or *LDAPS*. It is not recommend to use LDAP plain connections, on production enviroments.
- **Authentication mechanims:** *SIMPLE* or *NTLM v2*.

To enable LDAP auth set *WARP_AUTH_LDAP* env variable to *True*. WARP will check user login and password via LDAP bind action and the list of authorithed groups (see *WARP_LDAP_GROUP_MAP* env variable). If Bind acction succed and user belongs to authorized groups login is allowed. 

Authorization is managed by adding user, on first login, to WARP auth database and assign the WARP group defined by *WARP_LDAP_GROUP_MAP*. 

It is possible to exclude some users from using LDAP auth by adding them to *WARP_LDAP_EXCLUDED_USERS* list. Users on this list will check credentials via warp auth database ignoring LDAP AUTH configuration.

 **Configuration enviroment variables:**
- WARP_AUTH_LDAP: True
- LDAP_MATCHING_RULE_IN_CHAIN: Set to True for Active Directory or any LDAP supporting "LDAP_MATCHING_RULE_IN_CHAIN" (default is false). When enabled nested groups are considered when looking for group mapping.
- WARP_LDAP_EXCLUDED_USERS: array of users login that are excluded form LDAP login (password stored on warp DB).
- WARP_LDAP_GROUP_MAP: mapping between LDAP group and default asigned group on WARP.
- WARP_LDAP_USER_CLASS: LDAP user objectclass. Order is important as only one group is assigned. First match is used.
- WARP_LDAP_USER_ID_ATTRIBUTE: Attribute to compare with login
- WARP_LDAP_USER_NAME_ATTRIBUTE: Attribute used to obtain name of the automatically created user on warp. 
- WARP_LDAP_USER_GROUPS_ATTRIBUTE: attribute of the LDAP user contaning group list.
- WARP_LDAP_SEARCH_BASE: Base domain name to locate user and groups
- WARP_LDAP_AUTH_SERVER: yourServerName/Ip    
- WARP_LDAP_AUTH_SERVER_PORT: LDAP/LDAPS server port 
- WARP_LDAP_AUTH_USE_LDAPS: True form LDAPS connection not defined or False for LDAP plain connection
- WARP_LDAP_AUTH_TLS_VERSION: *'1.2'* for TLS1.2 not defined for TLS1
- WARP_LDAP_AUTH_CIPHER: Set to a valid cipher for LDAPS server not defined *ECDHE-RSA-AES256-SHA384* is used.
- WARP_LDAP_AUTH_VALIDATE_CERT: *False* for non certificate validation not defined or True for Certificate validation.
- WARP_LDAP_AUTH_TYPE: NTLM/SIMPLE 
- WARP_LDAP_AUTH_NTLM_DOMAIN: NTLM domain name is the prefix used for the login name when NTLM AUTH is enabled *DomainName\loginname*
  
  **Sample values:**
    ```
    WARP_AUTH_LDAP: 'true'
    LDAP_MATCHING_RULE_IN_CHAIN: 'true'
    WARP_LDAP_AUTH_TYPE: 'NTLM'  
    WARP_LDAP_AUTH_NTLM_DOMAIN: "Domain1"
    WARP_LDAP_EXCLUDED_USERS: '["admin"]'
    WARP_LDAP_GROUP_MAP: '[{"ldapGroup": "CN=LDAP-GROUP-NAME,OU=XXXX,DC=yourDomain,dc=com", "warpGroup": "AsignedGroupOnWarpApp"}]'
    WARP_LDAP_USER_CLASS: "user"
    WARP_LDAP_USER_ID_ATTRIBUTE: "sAMAccountName"
    WARP_LDAP_USER_NAME_ATTRIBUTE: "name"       
    WARP_LDAP_USER_GROUPS_ATTRIBUTE: "memberOf"
    WARP_LDAP_SEARCH_BASE: 'DC=yourDomain,DC=com'
    WARP_LDAP_AUTH_SERVER: 'ldapServerName'   

    # LDAP  
    WARP_LDAP_AUTH_SERVER_PORT: 389

    # LDAPS 
    WARP_LDAP_AUTH_SERVER_PORT: 636 
    WARP_LDAP_AUTH_USE_LDAPS: 'true'                            # When not pressent or False plaintext LDAP connection used
    WARP_LDAP_AUTH_TLS_VERSION:  '1.2'                          # 1.2 for TLS1.2 none for TLS1
    WARP_LDAP_AUTH_CIPHER: 'ECDHE-RSA-AES256-SHA384'
    WARP_LDAP_AUTH_VALIDATE_CERT: 'true'                        # False to disable certificate validation
    ```

### How to import users

You can add them manually one by one via the users' management tab or import them directly to the database. Basically, insert users to `user` table, look at the table definition in `warp/sql/schema.sql.`

The role is one of:
```
10 - admin
20 - regular user
90 - account blocked
```

Password is a hash used by `werkzeug.security.check_password_hash` (more documentation can be [found here](https://werkzeug.palletsprojects.com/en/2.0.x/utils/#werkzeug.security.generate_password_hash)), by default (in my configuration) it is pbkdf2:sha256 with 16 bytes salt and 260,000 iterations.

You can generate it with Python (just make sure you have activated the environment where Flask is installed):
```
python -c 'from getpass import getpass; from werkzeug.security import generate_password_hash; print(generate_password_hash(getpass()))'

```

# Other

## How can I support you

Oh.. I was not expecting that, but you can send a beer via PayPal: https://paypal.me/sebo271

### Can I pay for a feature or support

Reach me out on my mail (git log is your friend), and we can discuss.
