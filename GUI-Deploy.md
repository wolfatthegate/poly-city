Here is the full sequence, in order. Everything on the box runs as `ubuntu` from `~/sponsors/deploy`.

**1. Get the code onto the box.** Merge PR #400 first, then on your Mac from the repo root:

```sh
git switch main && git pull
KEY=~/Downloads/shardtown-sponsors.pem
rsync -av -e "ssh -i $KEY" --exclude node_modules --exclude deploy/.env --exclude deploy/backups server/ ubuntu@174.129.247.30:~/sponsors/
ssh -i $KEY ubuntu@174.129.247.30
```

The rsync copies the new admin page, the API, `db/slots.sql` and the two migrations. Your `.env` and backups are left alone.

**2. Back up, then migrate the database.** The box has only run migrations 001 and 002. Both of these must run before the rebuild: 003 lets the slot list apply, and 004 is what the new booking page needs to load at all.

```sh
cd ~/sponsors/deploy
./backup.sh
./sql.sh < ../db/migrations/003-mural-slots.sql
./sql.sh < ../db/migrations/004-profile-pictures.sql
```

**3. Clear the bookings**, as you asked. The 10 there point at slot ids from the October 9 layout anyway. Logos are kept.

```sh
./sql.sh -e "DELETE FROM sponsorships"
```

**4. Rebuild and restart.** The admin container applies the 200-slot list to the database every time it starts, so no separate slot step is needed.

```sh
docker compose up -d --build
docker compose logs admin --tail 5
```

The log should include `slot list applied: 200 slots`. If it says a migration is missing, step 2 did not run. The Caddyfile did not change, so Caddy needs nothing.

**5. Check it.**

```sh
./sql.sh -e "SELECT COUNT(*) AS current_slots FROM ad_slots WHERE is_current = 1; SELECT COUNT(*) AS bookings FROM sponsorships"
```

Expect 200 and 0. Then open https://shardtownsponsors.wolfatthegate.com/admin/ with a hard refresh (Cmd+Shift+R), since the browser may still hold the old page. You should see the current map, diamonds for the mural walls, an empty bookings table, and the new Profile picture field in the form.

If you want to deploy before merging, rsync from the `blank-sponsor-signs` branch instead of main; the server code is identical.