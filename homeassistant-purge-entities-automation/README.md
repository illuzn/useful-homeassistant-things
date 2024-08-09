# Home Assistant Purge Entities Automation Blueprint

A set of Blueprints for Home Assistant to Purge Entities from the Database periodically.

## Time Trigger
This runs daily at a set time.
[![Import blueprint to Home Assistant](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Filluzn%2Fhomeassistant-purge-entities-automation%2Fblob%2Fmain%2Fpurge_entities_time_trigger_blueprint.yaml)

## Time Pattern Trigger
This runs every X hours as set by you.
[![Import blueprint to Home Assistant](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Filluzn%2Fhomeassistant-purge-entities-automation%2Fblob%2Fmain%2Fpurge_entities_time_pattern_trigger_blueprint.yaml)

Configuration:

Trigger Time: This is a set time each day (e.g. 3:22AM) or a time pattern (e.g. /6 for every 6 hours) depending on which version of the blueprint you have. Generally speaking, once per day is sufficient.

Purge Entities: The entities to be cleaned up.

Days to Keep: The minimum days to keep of your purged entities.

Repack: This will repack and optimise your database (hopefully shrinking it). Not repacking will leave the file size the same with spaces in your database that will get refilled by new data. As long as you are running this once per day I would recommend repacking because the extra disk writes are likely offset by the performance gains. If you are running multiple times a day I would disable this setting (and repack only once per day).