# Aperture (docker-compose)
Run Aperture—an IndieWeb feed aggregator—using Docker and docker-compose.

## Installation
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

This little README assumes your main site lives at `https://example.org` and that Aperture will be set up at `https://aperture.example.org`, Watchtower at `https://watchtower.example.org` and Camo at `https://camo.example.org`. You'll want to replace those four URLs with your own.

Make sure, when editing `.env`:
- the Camo URL ends in a trailing slash (and the Watchtower and Aperture URLs don't)
- to fill out an actual Redis password (the way `docker-compose.yml` is set up, `null` won't do)

Launch the different containers:
```
docker-compose build
docker-compose up -d
```
All files and dependencies are fetched the first time the containers are brought up, and configs are automatically set. This takes a while, so, er, wait a minute or two before moving on.

Now, set up and prepare both databases (replace `1234` with your token of choice—see `.env`). First Watchtower's:
```
docker exec -i watchtower_db_1 mysql -u watchtower -psome-random-password watchtower < ./watchtower/html/schema/schema.sql
docker exec watchtower_db_1 mysql -u watchtower -psome-random-password watchtower -e "INSERT INTO users (url, token, created_at) values ('https://aperture.example.org', '1234', NOW());"
```

And then Aperture's.
```
docker exec aperture_aperture_1 php /var/www/html/aperture/artisan migrate
docker exec aperture_aperture_1 php /var/www/html/aperture/artisan key:generate
```

Install the Watchtower service (on the host):
```
sudo cp ~/www/watchtower/build/watchtower.service /etc/systemd/watchtower.service
sudo systemctl enable watchtower
sudo systemctl start watchtower
```

This last step, right now, takes 15 seconds because of a pre-start delay. Reason for this rather random number was the time it takes for the files and dependencies and settings to be downloaded after the containers are first brought up. (Anyway, don't just terminate it if nothing seems to happen.)

And add the following Watchtower cron job (again, on the host), using `crontab -e`:
```
*/5 * * * * docker exec watchtower_watchtower_1 php /var/www/html/scripts/cron.php > /dev/null 2>&1
```

To make sure Aperture and its dependencies automatically start after a reboot, you may also want to add:
```
@reboot cd ~/www && docker-compose up -d
```

Once you've set up a token endpoint for your site, you can finally run:
```
docker exec aperture_aperture_1 php aperture/artisan create:user https://example.org/
```

Now all that's left is making sure `https://aperture.example.org`, `https://watchtower.example.org` and `https://camo.example.org` can actually be reached. (Via a web server, and DNS and things.)

Finally, refer to Aperture from your main site, like so:
```
<link rel="microsub" href="https://aperture.example.org/microsub/1">
```

## Updates
There are no official update instructions, but ...

Technically, you should be able to just `cd` into `~/www/aperture/html` or `~/www/watchtower/html` and execute `sudo git pull` and then `cd`, if needed, into whatever folder holds `composer.json` and run `docker run --rm --it -v $PWD:/app composer install`, and finally  `sudo chown -R 82:82 .`. This last step ensures the PHP-FPM containers have proper access to all PHP files. (The `82` refers to the `www-data` user inside 'em.)

It's probably much easier, however, to just run `docker-compose down` and remove the `html` folders, and then bring everything back up again. If the code didn't change too drastically and the main `.env` file was left untouched, all of the necessary config files should be automatically recreated, and all that's left to do is generate a new application key—`docker exec aperture_aperture_1 php /var/www/html/aperture/artisan key:generate`.
