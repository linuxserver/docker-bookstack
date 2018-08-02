
[linuxserverurl]: https://linuxserver.io
[forumurl]: https://discourse.linuxserver.io
[ircurl]: https://www.linuxserver.io/irc/
[appurl]: https://www.bookstackapp.com
[dockerfileurl]: https://github.com/linuxserver/docker-bookstack/blob/master/Dockerfile
[hub]: https://hub.docker.com/r/<image-name>/

[![linuxserver.io](https://raw.githubusercontent.com/linuxserver/docker-templates/master/linuxserver.io/img/linuxserver_medium.png?v=4&s=4000)][linuxserverurl]

## Contact information:-

| Type | Address/Details |
| :---: | --- |
| Discord | [Discord](https://discord.gg/YWrKVTn) |
| Forum | [Linuserver.io forum][forumurl] |

&nbsp;
&nbsp;

The [LinuxServer.io][linuxserverurl] team brings you another image release featuring :-

 + regular and timely application updates
 + easy user mappings
 + custom base image with s6 overlay
 + weekly base OS updates with common layers across the entire LinuxServer.io ecosystem to minimise space usage, down time and bandwidth
 + security updates

# docker-bookstack

[![Dockerfile-link](https://raw.githubusercontent.com/linuxserver/docker-templates/master/linuxserver.io/img/Dockerfile-Link-green.png)][dockerfileurl]

[BookStack](https://www.bookstackapp.com) is a free and open source Wiki designed for creating beautiful documentation. Feautring a simple, but powerful WYSIWYG editor it allows for teams to create detailed and useful documentation with ease.

Powered by SQL and including a Markdown editor for those who prefer it, BookStack is geared towards making documentation more of a pleasure than a chore.

For more information on BookStack visit their website and check it out: https://www.bookstackapp.com

## Usage

This container depends on an SQL server to provide the storage database. If you have one set up already (Docker or otherwise) then continue but if not then deploy a MariaDB container from [this dockerhub page](https://hub.docker.com/r/linuxserver/mariadb/)
```
docker create \
  --name=bookstackapp \
  -v <path to data>:/config \
  -e PGID=<gid> -e PUID=<uid>  \
  -e DB_HOST=<yourdbhost> \
  -e DB_USER=<yourdbuser> \
  -e DB_PASS=<yourdbuser> \
  -e DB_DATABASE=bookstackapp
  -e "APP_URL=https://your.site.com" \
  -p 6875:80 \
  docker-bookstack
```

It is strongly recommended that this container is used with our LetsEncrypt container so that your BookStack app is served over valid HTTPS.

## Parameters

The parameters are split into two halves, separated by a colon, the left hand side representing the host and the right the container side.
For example with a port -p external:internal - what this shows is the port mapping from internal to external of the container.
So -p 8080:80 would expose port 80 from inside the container to be accessible from the host's IP on port 8080
http://192.168.x.x:8080 would show you what's running INSIDE the container on port 80.

| Parameter | Function |
| :---: | --- |
| `-p 6875:80` | will map the container's port 80 to port 6875 on the host |
| `-v /config` | this will store any uploaded data on the docker host |
| `-e APPURL` | this will set the app url, see below for explanation |
| `-e PGID` | for GroupID, see below for explanation |
| `-e PUID` | for UserID, see below for explanation |
| `-e DB_HOST` | for specifying the database host, see below for further explanation |
| `-e DB_USER ` | for specifying the database user |
| `-e DB_PASS ` | for specifying the database password |
| `-e DB_DATABASE ` | for specifying the database to be used |


## User / Group Identifiers

Sometimes when using volumes (`-v` flags) permissions issues can arise between the host OS and the container, we avoid this issue by allowing you to specify the user `PUID` and group `PGID`.

Ensure any volume directories on the host are owned by the same user you specify and it will "just work" &trade;.

In this instance `PUID=1001` and `PGID=1001`, to find yours use `id user` as below:

```
  $ id <dockeruser>
    uid=1001(dockeruser) gid=1001(dockergroup) groups=1001(dockergroup)
```

## Setting up the application

This application is dependent on an SQL database be it one you already have or a new one. If you do not already have one, set up our MariaDB container.

Once the MariaDB container is deployed, you can enter the following commands into the shell of the MariaDB container to create the user, password and database that the app will then use. Replace myuser/mypassword with your own data.

**Note** this will allow any user with these credentials to connect to the server, it is not limited to localhost

`
from shell: mysql -u root -p
CREATE DATABASE bookstackapp;
GRANT USAGE ON *.* TO 'myuser'@'%' IDENTIFIED BY 'mypassword';
GRANT ALL privileges ON 'bookstackapp'.* TO 'myuser'@localhost;
FLUSH PRIVILEGES;
`

Once you have completed these, you can then use the docker run command to create your BookStack container. Make sure you replace things such as <yourdbuser> with the correct data.

The appurl must be set to tell the application where it will be served from. If you are using https://books.mysite.io then APP_URL will need to be set to `-e "APP_URL=https://books.mysite.io"`

Then docker start bookstackapp to start the container. You should then be able to access the container at http://dockerhost:6875

Default username is admin@admin.com with password of **password**

Documentation can be found at https://www.bookstackapp.com/docs/

## Container access and information.

| Function | Command |
| :--- | :--- |
| Shell access (live container) | `docker exec -it <container-name> /bin/bash` |
| Realtime container logs | `docker logs -f <container-name>` |
| Container version | `docker inspect -f '{{ index .Config.Labels "build_version" }}' <container-name>` |
| Image version |  `docker inspect -f '{{ index .Config.Labels "build_version" }}' <image-name>` |
| Dockerfile | [Dockerfile][dockerfileurl] |

## Changelog

|  Date | Changes |
| :---: | --- |
| 02.07.18 |  Initial Release. |
