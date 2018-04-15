# Hod: A container for your containers

A *hod* is a set of containers that work as a unit (it also means
[other things][hod-definition]).  It's a simple
system, using a reverse proxy ([Traefik][traefik]) to route requests
to a set of containers managed by [Docker Compose][docker-compose].
If you need something that's less simple, look at [Nomad][nomad].  If
you need something that's *not* simple, look at Kubernetes or Mesos or
etc....

This isn't revolutionary, it's barely even evolutionary.  It's based
on various Internet resources, such as the [Digital Ocean Ubuntu
example][do-example] and the [configuration
fragments][traefik-examples] in the Traefik docs.

It *is*, however, a template for a variety of different setups that I
run into and as such seemed worth pulling out.  Think of this repo as
a template or a working starting point for a solution to your problem.

## To get it going


1. Perhaps, update docker-compose (docker-compose version 1.13.0 is
   known to work, as is 1.20.1).  Follow [their
   directions][install-compose] (update the version number in the URLs
   as appropriate).

   ```
   sudo curl -L https://github.com/docker/compose/releases/download/1.20.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
   sudo chmod +x /usr/local/bin/docker-compose
   sudo curl -L https://raw.githubusercontent.com/docker/compose/1.20.1/contrib/completion/bash/docker-compose -o /etc/bash_completion.d/docker-compose
   ```

2. Read through the `docker-compose.yml` and `traefik.toml` files and
   update the hostname references and other variable bits (e.g. the
   paths to the files that are mounted into the Docker containers).

   Use a reasonable email address in the `acme` section of
   `traefik.toml`.

   By default the `acme.json` and `traefik.toml` files are mounted from
   the directory in which the `docker-compose.yml` file resides.  Adjust
   to suit.

   If none of your frontend rules (explicity in `traefik.toml` or as
   labels in `docker-compose.yml`) use a `Host` rule then uncomment
   and touch up the `acme.domains` bit at the bottom of `traefik.toml`
   to ensure that a certificate is generated.

   The password for the admin user can be generated like so (using
   `htpasswd` from the Apache project) if you're using basic auth:

   ```
   htpasswd -nb admin secure_password
   ```

   or like so for digest auth:

   ```
   htdigest -c password-file traefik api
   cat password-file
   ```

   and then set in the `traefik.toml` file.  You might want to use a
   different password....

   Adjust the `external` attribute of Docker network you're using
   (`web`); set it to `true` if you want to create the network
   yourself (or already have), `false` if you want `docker-compose` to
   handle it for you.

   If you add new services, remember that they're not exposed by
   default, you'll need to add a label that exposes them (or change
   `traefik.toml`).

3. Create the acme.json file, it must be owned by root and chmod'ed 600.

   ```
   sudo touch acme.json
   sudo chmod 600 acme.json
   ```

3. Create the Docker network if necessary.  The default configuration
   uses one named `web`, adjust it if necessary

4. Run docker-compose, e.g.

   ```
   docker-compose up -d
   ```

   Wait a few moments before trying it out so that Traefik has time to
   set up the certificates.  If you rush it you'll be served a
   self-signed certificate (probably displeasing your browser).  Take
   a deep breath, count [up to 11][11] and try again.

5. Cocktail.

6. You changed the admin password, right?

### Managing secrets

It's nice to be able to pass "secrets" into the containers without
having them committed to the central source control.  You can do this
by maintaining a separate secrets file and merging it in when you run
`docker-compose`, e.g.:

```
docker-compose -f docker-compose.yml -f ~/.cimr-secrets/secrets.yml up
```

Here's an example of a set of secrets for a cimr container:

```yaml
version: '2'

services:
  ft-cimr:
    environment:
      JENKINS_ADMIN_PASSWORD: bloop
      JENKINS_GIT_PRIVATE_KEY: |-
        -----BEGIN RSA PRIVATE KEY-----
        MIIEpQIBAAKCAQEA7ZMUhGe8O7rK5cimlIGia3Ze1+R5JfS8uk4Yovc3YiupW6PY
        qY+UDin5ewBaZVfUaPygnTrssuDf5HvtOkSnOvPA7hyUiHtLtkEuMww/w/O2OEK4
        XEOCQaiJGVp4ykkLvCJ4Gu4euKB1zwKb2oPg5BrYpqCJuvjX5Eu3Yp4+0gCbZv8I
        [... many lines deleted]
        AR5UJF4Edd8WmyUTiRbopzbxJnxDrZipY1Qfsk4d5bNLo1Wo06jxyhE=
        -----END RSA PRIVATE KEY-----
      JENKINS_GITHUB_TOKEN: 5efac3158d4523505383f16e8867d754fa5z32fd
      JENKINS_SLAVES_PRIVATE_KEY: |-
        -----BEGIN RSA PRIVATE KEY-----
        MIIEpQIBAAKCAQEA7ZMUhGe8O7rK5cimlIGia3Ze1+R5JfS8uk4Yovc3YiupW6PY
        qY+UDin5ewBaZVfUaPygnTrssuDf5HvtOkSnOvPA7hyUiHtLtkEuMww/w/O2OEK4
        XEOCQaiJGVp4ykkLvCJ4Gu4euKB1zwKb2oPg5BrYpqCJuvjX5Eu3Yp4+0gCbZv8I
        [... many lines deleted]
        AR5UJF4Edd8WmyUTiRbopzbxJnxDrZipY1Qfsk4d5bNLo1Wo06jxyhE=
        -----END RSA PRIVATE KEY-----
```

### What should work

At this point (assuming no radical changes to the configuration),
these things should work (don't be silly, use *your* hostname in the
URL...):

| URL                     | result                                                                                                                                      |
|-------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| http://hostname/ping    | routed to Traefik's ping backend, responds OK (or not).                                                                                     |
| http://hostname/api     | redirects to Traefik's https port then to the the API/dashboard backend, which requires auth.                                               |
| https://hostname/api    | hits Traefik's https port then to the API/dashboard backend, which requires auth.                                                           |
| http://hostname/whoami  | redirects to Traefik's https port, which requires auth, then to a container running the "whoami" image, responds with some non-random data. |
| https://hostname/whoami | hits Traefik's https port, which requires auth, then to a container running the "whoami" image, responds with some non-random data.         |

## Notes

The routing of traffic from ports 80 and 443 to the "API/health"
backend on port 8080 is a bit magical.  On a good day, with a tail
wind, I believe I understand what's going on.  I'm not sure I can
explain it though (so perhaps I don't really understand it...).  You
can read about more about it in [Traefik issue #2833][2833], from
whence I stole it....


[11]: https://en.wikipedia.org/wiki/Up_to_eleven
[2833]: https://github.com/containous/traefik/issues/2833
[do-example]: https://www.digitalocean.com/community/tutorials/how-to-use-traefik-as-a-reverse-proxy-for-docker-containers-on-ubuntu-16-04
[docker-compose]: https://docs.docker.com/compose/
[hod-definition]: https://en.wikipedia.org/wiki/Hod
[install-compose]: https://docs.docker.com/compose/install/#install-compose
[nomad]: https://www.nomadproject.io/
[traefik-examples]: https://docs.traefik.io/user-guide/examples/
[traefik]: https://traefik.io/
