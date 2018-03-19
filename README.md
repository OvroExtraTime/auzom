# [auzom](https://legacy.auzom.gg/)

Esports tournament platform by Auzom Entertainment.

### Prerequisites

1. [Install
   Docker](https://docs.docker.com/engine/installation/linux/ubuntulinux/).
1. [Install Docker Compose](https://docs.docker.com/compose/install/).
1. Make sure port `3000` isn't taken.

### Getting started

1. Run `./setup.sh` to do an initial setup.
1. **Optional.** Adjust configuration. See [Configuration](#configuration) for
  more.
1. Run `docker-compose up -d` to launch the app.
1. Run `docker-compose logs -f` and wait until it stops printing stuff rapidly.
   Exit with `ctrl + c`.
1. **Optional.** Run `./dev-load-prod.sh <dumpfile>` to import a database dump.
1. Run `./dev-migrate.sh`, even if you've imported a dump.
1. The app should be available at `https://<VIRTUAL_HOST>`, see
  `nginx/.env.local`.

### Configuration

##### `nginx/.env.local`

Contains the following:

* `VIRTUAL_HOST`: your hostname, can also be an IP
* `LETSENCRYPT_HOST`: should be the same as `VIRTUAL_HOST`
* `LETSENCRYPT_EMAIL`: your main email address

Consumed by `nginx-proxy` and `letsencrypt-nginx-proxy-companion`.

---

##### `backend/.env.local`

Contains only one variable - `STATIC_HOST` - which should be the same as
`VIRTUAL_HOST` from `nginx/.env.local`. Used for CORS and mailing inside the
backend, nothing else.

---

##### `.env.local`

Contains the following:

* Imports `POSTGRES_VERSION` from `.env`, which is tracked by git
* `POSTGRES_URL_TEMPLATE`: connection string, with `{pwd}` in place of password
* `DOCKER_REGISTRY`: the registry to push images to

This is utilized by `prod-*.sh` scripts and `Makefile`. The password itself
isn't stored intentionally, for security purposes. Of course, if an attacker
gains access to the project tree of a developer who has production backups, then
they'll have the whole database dump, but at least they won't be able to mess
with the production database.

### Script overview

##### `bashin.sh`

The most used of all. It does what it says: gets you a `bash` shell `in` any of
the running services listed in `docker-compose`.

If you run `./bashin.sh <service>`, then everything inside the `<service>`
container gets killed, and you get a fresh shell to do whatever. You can also
run `./bashin.sh <service> nokill`, to get a secondary shell inside the same
container, without killing anything, which can be useful for parallel dependency
installation.

Once you've bashed in, you might want to restart whatever you've killed, which
is `gulp` for `backend` and `yarn start` for `frontend`.

Haven't seen anybody else [mis]use Docker in such way, but I personally don't
see anything wrong with this approach (trust me, I've debated with myself about
this *really* hard).

---

##### `dev-load-prod.sh`

Replace the development database with a backup made by `prod-backup.sh`.

---

##### `dev-migrate.sh`

Migrate the development database. Not really convenient, since building a
migration image takes quite a bit of time, but is useful for testing, before
applying new migrations to the production database. For development, use a
database client of your preference (for me it's DBeaver), by connecting to
`127.0.0.1:5432`.

---

##### `dockerception.sh`

Not for direct use (though feel free to).

---

##### `ecr-login.sh`

Runs `aws ecr get-login`, and then runs its output, which does a `docker login`.

---

##### `prod-backup.sh`

Does a dump of the production database to `.backup/production/*.sql`.

---

##### `prod-fix-privileges.sh`

There's an annoying problem with new tables / other types of new entities: for
some odd reason Postgres doesn't apply custom default privileges to them, even
though it was explicitly told to in the past. Surely it's my fault, but new
tables are made so rarely, that solving this just isn't worth many hours of
research (I've already done a few, haven't found a solution). This is just a
quick prophylactic measure, to avoid post-migration issues.

---

##### `prod-migrate.sh`

Migrate the production database. Can optionally build a new migration image.

### Wrapping up

1. Run `docker-compose kill` to shut down the app.
* **Optional.** Run `docker-compose rm` to clean up (data isn't preserved).
* **Optional.** You can also kill and remove the proxy stuff, see `proxy.sh`.

### Legal

The original screenshots, which background images on the landing page are
derived from, aren't made by me, but rather
[ShadowSix](https://twitter.com/Shadow6ix) and
[Berdu](https://twitter.com/Berduu), who were kind enough to allow us to use
them for Battlefield Conquest League, Aces High and Auzom itself.

All SVG files in `frontend/src/assets`, besides `logo.svg`, belong to the
respective brands/creators.

Auzom's logo is owned by Olegs Jeremejevs and has no license. You can get
permission to use it [via email](mailto:olegs@jeremejevs.com).

The rest of the repository is covered by [MIT License
(Expat)](https://tldrlegal.com/license/mit-license), as seen in `LICENSE.md`.

### Special thanks

... to Michiel De Backker AKA Wieweet, who has helped me out with the workload
and implemented a few important API endpoints / some key model logic in the
early days of Auzom.

### Random acknowledgements and pieces of information

##### State of the private repo

Development timeline:

![](http://i.imgur.com/W6QQYjp.png)

37 KLOC. 80K additions and 45K deletions total. The issue tracker has 87 open
and 164 closed issues + 10 milestones + there are 155 unchecked to-dos on my
personal brain dump list and 105 TODO comments scattered across the project,
which adds up to ~350 open issues.

##### Tests

There was API black-box testing in the beginning, and some UI testing was
planned as well, but then I had decided that they were objectively not worth the
time investment (the project was taking long enough as it is), since they can
barely help with security, and other issues couldn't possibly be severe enough
to damage our image. Since the tests were dropped, 4 tournament seasons have
been completed successfully, without a single problem encountered. Maybe we were
just lucky, but we'll never know.

##### Continuous integration

There is none, since I'm the only developer, and my local copy has always been
acting as the "staging" environment (thanks to Docker, differences between my
dev setup and the production are *almost* negligible). And, well, there are no
tests to run anyway.

##### Deployment

It's semi-automatic. Docker images are built and pushed with `make` locally, and
deployment is done by re-deploying these images in AWS Beanstalk. The procedure
is simple and speedy enough to not warrant any additional automation (and I
always like to take a look at the status of all deployments in AWS console
anyway). There's one issue worth considering - Node modules of the frontend not
being re-downloaded with every build - but me being the only developer it
doesn't matter much, and it's much faster to reuse existing `node_modules`.

##### Transactions

As-is, all transactions in the code are currently almost useless, except for the
rollback aspect of them. This was meant to be addressed at a later date, once
our userbase were to start growing by itself.

##### Migrations

There are no down migrations because they just don't make sense. If something
goes wrong and some data is destroyed in the result, then the API must be
shutdown immediately, the fatal flaws must be fixed with a custom-made SQL
script, and the destroyed data must be recovered from the latest backup. Making
these "fix stuff" SQL scripts in advance is a waste of time, since they are
quite likely to end up destroying other data without proper modifications.

##### Mobile optimization

There is none, since it would introduce too much overhead for barely any
advantage in the early stages. 91% of our traffic comes from desktop, so no
issue there (though, of course, there's some correlation between the usability
of a website on mobile and the amount of mobile visitors...).

##### Page titles / meta

Page titles would be handy, but there were no complaints from our users, so it
was a low priority task. Meta-wise, embedability and search engine optimization
didn't concern us either, we weren't looking for traction quite yet.

##### Server rendering

Was planned, but again, this didn't matter much at our stage.

##### Performance

Both the API and the client are far from being fully optimized (much more so for
the client), but it was a conscious choice to not invest any time into shaving
off milliseconds/megabytes, since there was a plan to switch to GraphQL and
[Apollo Client](https://github.com/apollostack/apollo-client) in the near
future, resulting in many performance issues rendering themselves obsolete.

##### Comments

Well, there aren't many. I believe in self-documenting code. Only the most
complicated parts have more than a one-liner about them, but, let's be honest
here, most of the website is plain old self-explanatory
[CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete).

##### Browser compatibility

We care only about the latest browsers. Thankfully, gaming crowd tends to be
up-to-date. Out of 40K sessions, tracked since June '16, only 8 were made from
IE 10 and below.

##### Et cetera

As mentioned, there are ~350 open issues, some about button title typos, some
about multi-week refactors. Most noteworthy shortcomings are outlined above, but
there are many more, which I intend to move to this repository in the next few
weeks. However, do report bugs and ways to improve anyway, worst case it's going
to be a duplicate.
