# Aperture (docker-compose)
Run Aperture—an IndieWeb feed aggregator—using Docker and docker-compose.

## Installation
This little README assumes your main site lives at `https://example.org`, and that Aperture will be set up at `https://aperture.example.org` and Watchtower at `https://watchtower.example.org`. You'll want to replace these three URLs with your own.

It also assumes you're running all of this under the user `aperture`. This mostly affects the provided [NGINX configuration](https://github.com/janboddez/aperture-docker-compose/tree/master/nginx-examples) files, which serve as examples only, and the Watchtower [user service](https://github.com/janboddez/watchtower-docker/blob/master/build/watchtower.service-user) file.

Use something like this:
```
mkdir ~/www
cd ~/www

git clone https://github.com/janboddez/aperture-docker-compose .

git submodule init
git submodule update

cp .env.example .env
```
Then, before kicking off the build step, update the values inside `.env`—and, if needed, `docker-compose.yml`.

Make sure, when editing `.env`, to fill out an actual Redis password (the way `docker-compose.yml` is set up, `null` won't do). Or modify `docker-compose.yml` according to your preferred setup.

If you're going to go with the prebuilt images in Docker's repository, comment out the `build` entries for `aperture` and `watchtower` in `docker-compose.yml`. (This'll save you some time.)

Launch the different containers:
```
docker-compose build
docker-compose up -d
```
All files and dependencies are fetched the first time the containers are brought up, and configs are automatically set. This takes a while, so, er, maybe wait a minute or two before moving on?

Now, set up and run migrations for both databases. First up is Watchtower's. (Replace `1234` with your token of choice—see `.env`. The same goes for your Watchtower database password: use the one you've specified earlier, in `.env`.)
```
docker exec -i watchtower_db_1 mysql -u watchtower -p<some-random-password> watchtower < ./watchtower/html/schema/schema.sql
docker exec watchtower_db_1 mysql -u watchtower -p<some-random-password> watchtower -e "INSERT INTO users (url, token, created_at) values ('https://aperture.example.org', '1234', NOW());"
```
(I know, you _shouldn't_ just type passwords into a terminal like that, but doing so allows us to set everything up with a single command. Feel free to run `history -c` afterward to make your system forget.)

And then Aperture's.
```
docker exec -ti aperture_aperture_1 php aperture/artisan migrate
docker exec aperture_aperture_1 php aperture/artisan key:generate
```
Note: The `-ti` bit in the first command _is_ important; it'll lead Artisan to ask for your permission running the migrations in a "production" environment—tell it "yes"!

Install the Watchtower service (on the host). At least on Ubuntu, this'll make sure Watchtower (inside its container) keeps doing its thing:
```
sudo cp ~/www/watchtower/build/watchtower.service /etc/systemd/watchtower.service
sudo systemctl enable watchtower
sudo systemctl start watchtower
```
Note: If you're on [Rootless Docker](https://docs.docker.com/engine/security/rootless/), this isn't entirely true. Provided you've correctly got Rootless Docker to run, this should get you on your way:
```
cp ~/www/watchtower/build/watchtower.service-user ~/.config/systemd/user/watchtower.service
```
Then edit the new `watchtower.service` file. Add in your actual user ID, and replace `aperture` with your username—your Linux UID and name, that is.
```
systemctl --user enable watchtower
systemctl --user start watchtower
```
This last step, right now, takes 15 seconds because of a pre-start delay that may or may not make sense. (Anyway, _don't just terminate it_ if nothing seems to happen.)

And add the following Watchtower cron job (again, on the host), using `crontab -e`:
```
*/5 * * * * docker exec watchtower_watchtower_1 php /var/www/html/scripts/cron.php > /dev/null 2>&1
```

To make sure Aperture and its dependencies automatically start after a reboot, you may also want to add:
```
@reboot cd ~/www && docker-compose up -d
```

Almost there! Once you've set up a token endpoint for your site, run:
```
docker exec aperture_aperture_1 php aperture/artisan create:user https://example.org/
```

Now all that's left is making sure `https://aperture.example.org` and `https://watchtower.example.org` can actually be reached. Two example NGINX config files can be found in `nginx-examples`. Feel free to drop these in your host system's `/etc/nginx/sites-available` and create the needed symlinks in `/etc/nginx/sites-enabled`. Or configure everything your way.

The examples, by the way, assume that the user all of the above was run under is named `aperture`. (E.g., the `root` directives point to `/home/aperture/www/<and-so-on>`.) And that Watchtower runs on port 9001, and Aperture on 9002. (You'd of course use the values in the `.env` file. And adapt all URLs. And make sure all of the necessary certificates are in place.)

Finally, refer to Aperture from your main site, like so:
```
<link rel="microsub" href="https://aperture.example.org/microsub/1">
```

## Caveat
When these containers are first set up, they don't actually contain any app code. There's a real possibilty that future Aperture updates are incompatible with the `docker-entrypoint.sh` scripts in the images. If that's the case, I'm going to have to start versioning the containers. (Aperture itself is still being developed and not versioned.)

This also somewhat complicates updating. On the one hand, it's probably possible bring all containers down, delete the `html` folders that are created in `www/watchtower` and `www/aperture` the first time the containers run, and bring everything back up. As long as the `db` folders aren't touched, you'd only have to run `docker exec -ti aperture_aperture_1 php aperture/artisan key:generate` after `www/aperture/.env` is recreated. (Regenerating an app key for existing apps isn't actually recommended, but I don't think it hurts in this case.)

Alternatively, you could enter the Watchtower and Aperture containers and, once inside, run `git pull` and `composer install --no-dev`. In Aperture's case, `cd` into the `aperture` subfolder first before running `composer`. Make sure you do this as the user with uid 82 (i.e., `www-data` inside the containers), though, or properly chown everything in `html` (on the host) afterward. Otherwise, the containers' respective `www-data` users won't be able to move or write to files.
