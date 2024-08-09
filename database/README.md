# Why is this Important?

Home Assistant uses an SQLite database. Essentially, this is a database stored in a single file on your computer. While leaps and bounds have been made optimising how this database is handled and database crashes are much less frequent, large databases are still the cause of significant issues.

This can affect you because:

- If you are using "spinning rust" (or a hard disk drive), a large database will cause significant slow downs/ loading times.
- If you are using a solid state drive, a large database will decimate your drives lifespan (in 2.5 years of use my own SSD had 90% of its reserve blocks used).
- A large SQL database is likely not backing up properly. You should periodically attempt restoring a backup. The precise mechanism of this is unclear, but with a large database (> 2GB) and using an SSD my backups would take a few minutes - it is my speculation that the database will be written to during this time and corruption can occur to the database due to those writes that occur during the backup process. 

Given the cheap cost of storage of late, I do not recommend using an SD card as your primary storage medium unless you are using Home Assistant for a limited and specific purpose (e.g. it only controls a few lights).

# Steps to Take

1. View the size of your database: Under 1GB should be relatively healthy on even anemic hardware like a Rasberry Pi 3.
2. Determine which entities are taking up the most space in your database: Usually, this is only a subset of your devices e.g. my solar inverter outputs household electricity consumption every second thats 86,400 entries in your database per day (now multiply that by the fact that it also does the same for solar production, grid power import, grid power export, etc and the numbers start getting very large very quickly)..
3. Determine useful limits and apply them: This involves making some hard choices about how much of a data hoarder you are.

# 1. Viewing the Total size of the Database

The first step is to measure the size of your database. This way you can see if you have a problem with inflated database, and how large is the problem.

One way to achieve that is by looking at the file size manually, using either [SSH](https://www.home-assistant.io/common-tasks/os/#installing-and-using-the-ssh-add-on-requires-enabling-advanced-mode-for-the-ha-user) (run `du -h /config/home-assistant_v2.db`) or by [Samba folder share](https://www.home-assistant.io/common-tasks/os/#installing-and-using-the-samba-add-on).

However, a much simpler and easier way is to configure a [sensor that shows the size of the database file](https://www.home-assistant.io/integrations/filesize/). Just make sure you understand the implications of [`allowlist_external_dirs`](https://www.home-assistant.io/docs/configuration/basic/#allowlist_external_dirs). Edit your `/config/configuration.yaml` file:

```yaml
homeassistant:
  allowlist_external_dirs:
    - /config
```

[and add the file size sensor](https://www.home-assistant.io/integrations/filesize/) using /config/home-assistant_v2.db as the file to monitor.

This sensor will be updated every half minute - which is far too often. [Disable polling](https://www.home-assistant.io/blog/2021/06/02/release-20216/#disable-polling-updates-on-any-integration) and use an automation to call the `homeassistant.update_entity` action once per day.

# 2. Finding the Heaviest Entities

To find which entities are using too much space, you will have to dig into the database itself and run a SQL query. Fortunately, there is already an add-on to make it easier for you.

1. Go to the Add-on store: [![Open your Home Assistant instance and show the Supervisor add-on store.](https://my.home-assistant.io/badges/supervisor_addon_store.svg)](https://my.home-assistant.io/redirect/supervisor_addon_store/)
2. Install the **SQLite Web** add-on: [![Open your Home Assistant instance and show the dashboard of a Supervisor add-on.](https://my.home-assistant.io/badges/supervisor_addon.svg)](https://my.home-assistant.io/redirect/supervisor_addon/?addon=a0d7b954_sqlite-web)
3. If you want, you can *Show in sidebar* and also *Start on boot*. The choice is yours.
4. Make sure this add-on is *started* before you continue.

When you open the *SQLite Web* interface, you can see a few tables, of which only two are relevant:

* `states` contains a log of all the states for all the entities.
* `events` contains a log of all the events. Some events include `event_data` (such as parameters in `call_service` events), but `state_changed` events have empty `event_data`, because the state data is stored in the other table.

If you are curious to view the size of each table, try this SQL query:

```sql
SELECT
  SUM(pgsize) bytes,
  name
FROM dbstat
GROUP BY name
ORDER BY bytes DESC
```

It will also show `ix_*` entries, which are not real tables, but just indexes to keys in the actual tables.

## Viewing Entity Usage

1. Click on the `states` table in *SQLite Web*.
2. Click on **Query**.
3. Type the following SQL query:

```sql
SELECT
  COUNT(*) AS cnt,
  COUNT(*) * 100 / (SELECT COUNT(*) FROM states) AS cnt_pct,
  states_meta.entity_id
FROM states
INNER JOIN states_meta ON states.metadata_id=states_meta.metadata_id
GROUP BY states_meta.entity_id
ORDER BY cnt DESC
```

This will output as a percentage the entities which are contributing the most entries to your `states` table (in descending order i.e. heaviest entities at the top).

Analyze the results. Most likely, the top few entities will contribute to a quarter (or maybe half) of the table size (either as number of rows, or as size of the rows). If you have very many entities which all take up the same amount of space, you most likely don't have an issue with particular entities causing issues (but read the caveats below).

### Caveats

1. This does not consider the size of entry. States has a maximum length of 255 (or approximately 0.25kB) and attributes has an infinite length. You may have certain entities which are huge but do not update that frequently. For example, my hourly weather forecast sensor publishes the entire forecast as an attribute  (comprising several kilobytes of text) and is updated hourly.
2. This does not consider your `events` table. Generally speaking, filtering events is far less granular and you will probably find that `state_changed` is the heaviest event and I would not recommend filtering this event.

# 3. Work Out Your Limits

Answer these useful questions:

1. How long do I want to-the-second data for? This drives entity history, the logbook, etc. Be realistic, are you going to care in 1 years time, whether the light was on at 4:31PM? For example, I have a very reasonably data base size (~600MB) and I use 30 days for this - long enough to usefully diagnose any issues.
2. Which entities do I literally not care about the history for? For example, past weather forecast data is useless to me (I'm not going back to check how accurate the weather forecast was and I don't care what the weather forecast was for today a week ago).
3. Which of those heavy entities I found in step 2 do I want history for but can live with statistical averages? For example, you won't know how much solar you generated at precisely 12:31:45PM but you will get 5 minute/ hourly/ daily statistical summaries (maximum, minimum, mean).

### Configure Your Recorder

Add the following to your `configuration.yaml`:

```yaml
recorder:
    purge_keep_days: 30 # This is your answer to question 1 above
    exclude: !include recorderexcludes.yaml # We will put the answer to question 2 in this seperate file for neatness.
```

### Configure Your Excludes

Create `recorderexcludes.yaml` in your /config/ folder. This is your answer from question 2. Here is an example:

```yaml
domains:
# This section excludes history for entire domains i.e. the domain.entity_name. I've provided some rationale for my examples below.
  - device_tracker # I don't care whether certain devices are home or not. The person domain still provides location history of people.
  - media_player # I don't care what was playing 1 hour ago (or 1 week ago).
  - sun # I don't care about history sunrise etc. You can calculate this online if really needed.
  - update # I don't care if there was an update a week ago for a certain addon.
  - calendar # If I want this history I'll look at my calendar rather than relying on history.
  - remote # I don't care what buttons I'm remote pressing using Home Assistant
  - todo # I don't care what my historic todos were
entity_globs:
# This section allows you to use wildcards * and ? to filter more than one entity that has a pattern in its entity name
  - sensor.*uptime* # I have a monitor of the uptime of my VMs/ docker containers. I don't care about the history of this because it will obviously just be increasing every second.
  - sensor.sun* # Same reason that the sun domain is excluded.
  - sensor.useless_entity_? # This is a hypothetical example only but useful where you have multiple sensors with this name pattern e.g. useless_entity_1, useless_entity_2, etc.
entities:
# This section allows you to filter individual entities
  - image.robot_vacuum_map # I don't need the image of my robot vacuum's history clogging up my database
  - sensor.router_receive_errors # This sensor is of very little use to me
  - sensor.weather_forecast_daily # Reasons for this outlined above
  - sensor.weather_forecast_hourly # Reasons for this outlined above
```

Note that these excludes mean:

1. No history will be shown for that entity.
2. No logbook will be kept for that entity.
3. No statistics will be kept for that entity.

Once you have made these changes restart Home Assistant and they will apply immediately.

### Use my Automation

My [cleanup database automation](/homeassistant-purge-entities-automation/README.md) is designed to for heavy entities which you still would like some history for (5 minute/ hourly/ daily statistical summaries of maximum, minimum, mean).

For most purposes a time trigger is sufficient (running once a day to cleanup your database); however, if you have particularly problematic entities you may want to use the time pattern trigger which allows you to run multiple times a day (e.g. every 6th hour).

Using this, my 1400 entities (after excludes) with a 30 day history causes my database to fluctuate between ~550MB immediately after a cleanup and ~650MB immediately before a cleanup.

# Other Suggestions

If you absolutely must have a very large database for whatever reason, I would suggest that you:

- use a real external database engine (e.g. MariaDB)
- ensure your hardware has enough RAM to hold the entire database in memory
- use the following for storage (in this order of preference):
  - enterprise SSDs (these have much higher TBW and MTBF ratings)
  - consumer NVME SSDs
  - portable/ USB SSDs
  - hard disk drives
  - high endurance eMMC/ SD Cards
- if using SSDs use drives which have oversized capacity compare to what you are using (these will have much larger reserve blocks compared to your database size)
- use some form of redundant storage e.g. RAID 0
- monitor the SMART statistics coming from your drive.

Your hard drive will fail (and likely fail in less than 3 years with a large database), it is not a question of if but when you will wake up to the realisation your instance has died.

# References

[Recorder Docs](https://www.home-assistant.io/integrations/recorder/)

[How to keep your recorder database size under control](https://community.home-assistant.io/t/how-to-keep-your-recorder-database-size-under-control)