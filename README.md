# Eskom and City of Cape Town Loadshedding alerts using Node-RED, MQTT to Home Assistant

Node-RED workflows to 
- Get current national loadshedding stage from the Eskom website
- Calculate the next loadshedding time for City of Cape Town zone (my office)
- Get the next loadshedding time from Eskom website (my home)
- Publish to MQTT topics

I then use Home Assistant MQTT sensors to read the states, display on my dashboard and send me push notifications.

## Node-RED flow

![Loadshedding Node-RED flow](/images/nodered.png)

**Download [Node-RED flow](nodered.json)**

To import, in your Node-RED click the hamburger menu at top right, Import, and paste or select from file.

### City of Cape Town schedule
My office gets its electricity from the City of Cape Town. The _Calculate Office schedule_ function makes use of [Quantiversal/Cape-Town-Loadshedding-Schedule](https://github.com/Quantiversal/Cape-Town-Loadshedding-Schedule) javascript library which I added to the functions `On Start` method.

`On Message` actually calls the functions in the loadshedding calculator.

### Eskom schedule

Eskom does not make a similar calculator (other than in an Excel sheet), so I chose to scrape the Eskom website for my areas' schedule. This can definitely be improved. (PRs welcome.)

The easiest way to get the URL used in _Get Home schedule from Eskom API_ is to go to the [Eskom loadshedding](https://loadshedding.eskom.co.za/) website, open Chrome dev tools, go to the Network tab and then search for your suburb in the Quicksearch for Direct Eskom Customers box. Select the suburb, and the page that is loaded is the URL you want.

Paste that URL, something like `https://loadshedding.eskom.co.za/LoadShedding/GetScheduleM/64660/2/Western%20Cape/822?_=1622753016064` into the URL box of the _Get Home schedule from Eskom API_ node. Where you see `/2/` in the URL, replace it with `/{{stage}}/` to substitute the current loadshedding stage and pull the correct calendar.


## MQTT messages

Update the MQTT nodes to use your MQTT broker.

Node-RED publishes messages with the following format:

Topic `loadshedding/stage`

The current loadshedding stage.

```
2
```

Topic `loadshedding/stage/updated`

Only publishes when the stage changes

```
3
```

Topic `loadshedding/home/next`

The current `time`, the loadshedding `schedule` for today and tomorrow and the `next` scheduled slot for the `home` area.

```json
{
  "time": "2021-06-03T22:14:30.040+02:00",
  "schedule": [
    "2021-06-03T02:00:00+02:00",
    "2021-06-03T10:00:00+02:00",
    "2021-06-04T10:00:00+02:00",
    "2021-06-04T18:00:00+02:00"
  ],
  "next": "2021-06-04T10:00:00+02:00"
}
```

Topic `loadshedding/office/next`

The current `time`, the loadshedding `schedule` for today and tomorrow and the `next` scheduled slot for the `office` area. (Note for my office I haven't implemented the `schedule` yet since.... covid means work from home. PR's welcome.)

```json
{
  "time": "2021-06-03T22:24:20.305+02:00",
  "schedule": [],
  "next": "2021-06-04T04:00:00.000+02:00"
}
```

Personally, I only every use `next`, but you could display today's and tomorrow's schedule. The timezone (+02:00) is so that Home Assistant correctly interprets the time.


## Home Assistant

In my [configuration.yaml](https://github.com/dalehumby/homeassistant-config/blob/master/configuration.yaml#L77) file I have the following code

```yaml
sensor:
  # Eskom loadshedding
  - platform: mqtt
    unique_id: "loadshedding.stage"
    name: "Loadshedding stage"
    state_topic: "loadshedding/stage"
    value_template: "{{ value | int }}"
    expire_after: 1800
    device_class: energy
    unit_of_measurement: stage
    icon: mdi:flash-off
  - platform: mqtt
    unique_id: "loadshedding.home_next"
    name: "Next loadshedding time"
    state_topic: "loadshedding/home/next"
    value_template: "{{ value_json.next }}"
    expire_after: 1800
    device_class: timestamp
```

Once you've added this to your configuration file, restart Home Assistant and the MQTT sensors should be added.


### Lovelace dashboard entity card
![Loadshedding entity card](/images/entity-card.png)

To add an entity card to your dashboard, click three dots at top right, Edit Dashboard, click Add Card button at botom of page, select Entities, Show Code Editor and paste the following:

```yaml
type: entities
entities:
  - entity: sensor.loadshedding_stage
  - type: conditional
    conditions:
      - entity: sensor.loadshedding_home_next
        state_not: None
    row:
      entity: sensor.loadshedding_home_next
      name: Next loadshed
      format: time
title: Power
state_color: false
```

I only show the next scheduled loadshedding time if there is currently loadshedding. Otherwise I just show the current stage.

### Lovelace dashboard badge
![Stage 2 badge](/images/badge.png)

For the badge I use the following yaml in my Lovelace dashboard:

```yaml
title: Home
views:
  - path: downstairs
    title: Downstairs
    badges:
      - type: entity-filter
        state_filter:
          - operator: '>='
            value: 1
        entities:
          - entity: sensor.loadshedding_stage
            name: Eskom
```

To add it, in your dashboard click three dots at top right, Edit Dashboard, click three dots again, select Raw configuration editor. Paste the yaml from `- type: entity-filter` onwards under the `badges` section. Then Save.

The badge only appears if there is loadshedding.


### Automations
I have a 15 minute reminder to boil the kettle pushed to my phone and watch, and if my TV is on play a voice message.

In [automations.yaml](https://github.com/dalehumby/homeassistant-config/blob/master/automations.yaml#L51) file of Home Assistant, paste the following:

```yaml
- id: loadshedding_update_home_schedule
  alias: "Update loadshedding schedule"
  trigger:
    platform: mqtt
    topic: "loadshedding/home/next"
  action:
    service: input_datetime.set_datetime
    entity_id: input_datetime.loadshedding_home_next
    data:
      datetime: "{{ trigger.payload_json.next }}"

- id: loadshedding_warning
  alias: "Loadshedding 15 min warning"
  trigger:
    platform: template
    value_template: "{{ (as_timestamp(states('sensor.date_time').replace(',', '')) - 2*60*60 + 15*60) == (state_attr('input_datetime.loadshedding_home_next', 'timestamp') | int)  }}"
  condition:
    condition: numeric_state
    entity_id: sensor.loadshedding_stage
    above: 0
  action:
    - service: notify.mobile_app_dales_iphone_xs
      data:
        message: "Loadshedding in 15 min"
    - service: tts.google_cloud_say
      entity_id: media_player.apple_tv
      data:
        message: "Alert... Loadshedding starts in 15 minutes. Time to put on the kettle."
```
