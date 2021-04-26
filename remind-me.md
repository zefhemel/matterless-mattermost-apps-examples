# Remind me later

Adds a "Remind me" option to the post menu, allowing you to remind you about a post in a certain amount of time, using
human readable dates and times.

## To run

To run with Matterless:

```bash
$ mls run remind-me.md
```

Install in Mattermost:

```
/apps debug-add-manifest --url http://localhost:8222/remind-me/manifest.json
/apps install --app-id remind-me
```

Uninstall:

```
/apps uninstall remind-me
```

# Implementation

## events

```yaml
http:GET:/manifest.json:
  - ManifestHTTP
http:POST:/bindings:
  - BindingsHTTP
http:POST:/install:
  - InstallHTTP
http:POST:/remind/submit:
  - RemindFormHTTP
http:POST:/reminder-add/submit:
  - ReminderAddHTTP
```

# Mattermost Apps configuration

## function ManifestHTTP

```javascript
import {events} from "./matterless.ts";

function handle(event) {
    events.respond(event, {
        status: 200,
        body: {
            app_id: "remind-me",
            display_name: "Remind me about a post",
            app_type: "http",
            root_url: "http://localhost:8222/remind-me",
            requested_permissions: [
                "act_as_bot"
            ],
            requested_locations: [
                "/post_menu"
            ]
        }
    });
}
```

## function BindingsHTTP

```javascript
import {events, store} from "./matterless.ts";

function handle(event) {
    console.log("Got bindings call", event);
    events.respond(event, {
        status: 200,
        body: {
            type: "ok",
            data: [
                {
                    location: "/post_menu",
                    bindings: [
                        {
                            location: "remind-button",
                            label: "Remind me",
                            call: {
                                path: "/remind",
                                expand: {
                                    post: "all",
                                    team: "all",
                                    acting_user: "all",
                                    channel: "all"
                                }
                            }
                        }
                    ]
                },
            ]
        }
    });
}
```

## function InstallHTTP

```javascript
import {events, store} from "./matterless.ts";

async function handle(event) {
    let ctx = event.json_body.context;
    console.log("Just got an install", event);

    await store.put("bot:token", ctx.bot_access_token);
    await store.put("mattermost:url", ctx.mattermost_site_url);

    events.respond(event, {
        status: 200,
        body: "OK"
    });
}
```

# UI

## function RemindFormHTTP

```javascript
import {events} from "./matterless.ts";

function handle(event) {
    events.respond(event, {
        status: 200,
        body: {
            type: "form",
            form: {
                title: "When shall I remind you?",
                fields: [
                    {
                        type: "static_select",
                        name: "when",
                        label: "When (pre-defined)",
                        options: [
                            {label: "in 20 min", value: "in 20min"},
                            {label: "in 1 hour", value: "in 1h"},
                            {label: "in 3 hours", value: "in 3h"},
                            {label: "tomorrow", value: "tomorrow 9am"},
                            {label: "next week", value: "next Monday 9am"},
                        ]
                    },
                    {
                        type: "text",
                        name: "custom",
                        label: "When (custom)",
                        description: "e.g. 'Thursday at 2pm'"
                    }
                ],
                call: {
                    path: "/reminder-add"
                }
            }
        }
    });
}
```

## function ReminderAddHTTP

```javascript
import chrono from 'https://cdn.skypack.dev/chrono-node';
import {Mattermost} from "https://raw.githubusercontent.com/zefhemel/matterless/master/lib/mattermost_client.js";
import {events, store} from "./matterless.ts";

async function handle(event) {
    let context = event.json_body.context;
    let postUrl = `${context.mattermost_site_url}/${context.team.name}/pl/${context.post.id}`;
    let values = event.json_body.values;
    let actingUser = context.acting_user;

    // If selected one of the pre-selected option, prefer that, otherwise fall back to the custom option
    let dateString = values.when ? values.when.value : values.custom;

    let parsedDate = chrono.parseDate(dateString, new Date().toLocaleString("en-us", {timeZone: actingUser.timezone.automaticTimezone}));

    let client = new Mattermost(context.mattermost_site_url, context.bot_access_token);
    let directChannel = await client.createDirectChannel(context.bot_user_id, actingUser.id);

    if (!parsedDate) {
        await client.createPost({
            channel_id: directChannel.id,
            message: `Invalid reminder time...`
        });
        return events.respond(event, {
            status: 200
        });
    }

    await client.createPost({
        channel_id: directChannel.id,
        message: `I will remind you about [this post](${postUrl}) at ${parsedDate} :thumbsup:.`
    });

    await store.put(`reminder:${parsedDate.toISOString()}:${context.post.id}`, {
        postUrl: postUrl,
        directChannelId: directChannel.id,
        snippet: context.post.message,
        channelName: context.channel.display_name,
    });

    events.respond(event, {
        status: 200
    });
}
```

# Reminder cron job

## imports

* https://raw.githubusercontent.com/zefhemel/matterless/master/lib/cron.md

## cron ReminderChecker

```yaml
schedule: "*/10 * * * * *"
function: CheckReminders
```

## function CheckReminders

```javascript
import {store} from "./matterless.ts";
import {Mattermost} from "https://raw.githubusercontent.com/zefhemel/matterless/master/lib/mattermost_client.js";

let client;

async function init() {
    client = new Mattermost(await store.get('mattermost:url'), await store.get('bot:token'));
}

async function handle() {
    const now = new Date();
    const todayPrefix = now.toISOString().split(':').slice(0, 1).join(':');

    // Fetch all reminders schedule for today (to be safe)
    for (const [key, value] of await store.queryPrefix(`reminder:${todayPrefix}:`)) {
        // Extract the date/time from the key
        let dateString = key.split(':').slice(1, -1).join(':');
        let d = new Date(dateString);

        // If we're after the time, send the reminder message
        if (d < now) {
            // Send the message
            await client.createPost({
                channel_id: value.directChannelId,
                message: `Reminder for [this post](${value.postUrl}) in "${value.channelName}": ${value.snippet}`
            });
            // And delete it from the store
            await store.del(key);
        }
    }
}
```