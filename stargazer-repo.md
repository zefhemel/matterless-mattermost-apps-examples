# Mattermost Star Gazer App 🌟

[Demo video](https://cln.sh/0KPiSc)


# To run
```bash
$ mls run stargazer-repo.md
```

This will deploy and run it to `http://localhost:8222/stargazer-repo`.

# To install
Into Mattermost:

```
/apps debug-add-manifest --url http://localhost:8222/stargazer-repo/manifest.json
/apps install --app-id stargazer-repo
```

To uninstall:

```
/apps uninstall stargazer-repo
```

# How to use
Either use the "stargazer" button in the header, or the `/stargazer` command.

# imports

* https://raw.githubusercontent.com/zefhemel/matterless/master/lib/cron.md

# events

```yaml
http:GET:/manifest.json:
  - ManifestHTTP
http:POST:/install:
  - InstallHTTP
http:POST:/bindings:
  - BindingsHTTP
http:POST:/stargazer-modal/submit:
  - StargazerModalFormHTTP
http:POST:/stargazer-add-repo/submit:
  - WatchHTTP
http:POST:/list/submit:
  - ListHTTP
http:POST:/unwatch/lookup:
  - UnwatchLookupHTTP
http:POST:/unwatch/submit:
  - UnwatchHTTP
http:POST:/watch/submit:
  - StargazerAddHTTP
```

# Core watch functionality

## cron StarChecker
```yaml
schedule: "0 0 * * * *"
function: CheckStars
```

## function CheckStars
```javascript
import {store} from "./matterless.ts";
import {Mattermost} from "https://raw.githubusercontent.com/zefhemel/matterless/master/lib/mattermost_client.js";

async function handle() {
    const token = await store.get("bot:token"),
          url = await store.get("mattermost:url"),
          botUserId = await store.get("bot:user_id");

    if (!token) {
        // Not installed yet
        return;
    }
    
    let mmClient = new Mattermost(url, token);
    console.log("Checking the stars");
    for (let [key, currentCount] of await store.queryPrefix("watch:")) {
        let [_, channelId, repo] = key.split(':');
        
        // Call Github API
        let result = await fetch(`https://api.github.com/repos/${repo}`);
        let json = await result.json();
        let newCount = json.stargazers_count;
        
        console.log("Old count", currentCount, "new count", newCount);

        if (currentCount !== newCount) {
            await mmClient.createPost({
                channel_id: channelId,
                message: `Stars for repo [${repo}](https://github.com/${repo}): ${newCount} :rocket:`
            });
            await store.put(key, newCount);
        }
    }
}
```

# Mattermost Apps manifests

## function ManifestHTTP

```javascript
import {events} from "./matterless.ts";

function handle(event) {
    events.respond(event, {
        status: 200,
        body: {
            app_id: "stargazer-repo",
            display_name: "Stargazer app",
            app_type: "http",
            root_url: "http://localhost:8222/stargazer-repo",
            requested_permissions: [
                "act_as_bot"
            ],
            requested_locations: [
                "/channel_header",
                "/command"
            ]
        }
    });
}
```

## function BindingsHTTP

```javascript
import {events} from "./matterless.ts";

async function handle(event) {
    events.respond(event, {
        status: 200,
        body: {
            type: "ok",
            data: [
                {
                    location: "/channel_header",
                    bindings: [
                        {
                            location: "send-button",
                            icon: "https://github.blog/wp-content/uploads/2020/09/github-stars-logo_Color.png",
                            label: "Watch the stars for a new repo",
                            call: {
                                path: "/stargazer-modal"
                            }
                        }
                    ]
                },
                {
                    location: "/command",
                    bindings: [
                        {
                            icon: "https://github.blog/wp-content/uploads/2020/09/github-stars-logo_Color.png",
                            label: "stargazer",
                            description: "Github star checker",
                            bindings: [
                                {
                                    location: "list",
                                    label: "list",
                                    form: {
                                        fields: [],
                                        call: {
                                            path: "/list"
                                        }
                                    }
                                },
                                {
                                    location: "watch",
                                    label: "watch",
                                    form: {
                                        title: "Watch a repo",
                                        icon: "https://github.blog/wp-content/uploads/2020/09/github-stars-logo_Color.png",
                                        fields: [
                                            {
                                                name: "repo",
                                                label: "repo",
                                                type: "text",
                                            }
                                        ],
                                        call: {
                                            path: "/watch"
                                        }
                                    }
                                },
                                {
                                    location: "unwatch",
                                    label: "unwatch",
                                    form: {
                                        title: "Stop watching a repository",
                                        icon: "https://github.blog/wp-content/uploads/2020/09/github-stars-logo_Color.png",
                                        fields: [
                                            {
                                                name: "repo",
                                                label: "repo",
                                                type: "dynamic_select",
                                            }
                                        ],
                                        call: {
                                            path: "/unwatch"
                                        }
                                    }
                                }
                            ]
                        }
                    ]
                }
            ]
        }
    });
}
```

# Lifecycle

## function InstallHTTP
```javascript
import {events, store} from "./matterless.ts";

async function handle(event) {
    console.log("Just got an install", event);
    let ctx = event.json_body.context;
    await store.put("bot:token", ctx.bot_access_token);
    await store.put("bot:user_id", ctx.bot_user_id);
    await store.put("mattermost:url", ctx.mattermost_site_url)
    await store.put("mattermost:team_id", ctx.team_id)
    
    events.respond(event, {
        status: 200,
        body: "OK"
    });
}
```

# Modal implementation

## function StargazerModalFormHTTP

```javascript
import {events} from "./matterless.ts";

function handle(event) {
    console.log("Hello", event.json_body.context)
    events.respond(event, {
        status: 200,
        body: {
            type: "form",
            form: {
                title: "Watch a new repository",
                icon: "https://github.blog/wp-content/uploads/2020/09/github-stars-logo_Color.png",
                fields: [
                    {
                        type: "text",
                        name: "repo",
                        label: "Github repo"
                    }
                ],
                call: {
                    path: "/stargazer-add-repo"
                }
            }
        }
    });
}
```

## function WatchHTTP

```javascript
import {events, store} from "./matterless.ts";

async function handle(event) {
    let channelId = event.json_body.context.channel_id;
    let repo = event.json_body.values.repo;
    await store.put(`watch:${channelId}:${repo}`, 0);
    events.respond(event, {
        status: 200,
        body: {
            status: "ok",
            markdown: "Done!"
        }
    });
}
```

# Slash command implementation

## Listing

### function ListHTTP
```javascript
import {events, store} from "./matterless.ts";

async function handle(event) {
    let channelId = event.json_body.context.channel_id;

    events.respond(event, {
        status: 200,
        body: {
            type: "ok",
            markdown: (await store.queryPrefix(`watch:${channelId}:`)).map(([key, val]) => {
                const [_, channelId, repo] = key.split(':');
                return `* ${repo}`;
            }).join("\n") || "Not watching anything!"
        }
    });
}
```

## Unwatch

### function UnwatchLookupHTTP
For autocompletion of the `unwatch` command.

```javascript
import {events, store} from "./matterless.ts";

async function handle(event) {
    let channelId = event.json_body.context.channel_id;

    events.respond(event, {
        status: 200,
        body: {
            type: "ok",
            data: {
                items: (await store.queryPrefix(`watch:${channelId}:`)).map(([key, val]) => {
                    const [_, channelId, repo] = key.split(':');
                    return {
                        label: repo,
                        value: repo
                    };
                })
            }
        }
    });
}
```

### function UnwatchHTTP
The actual unwatch logic.

```javascript
import {events, store} from "./matterless.ts";

async function handle(event) {
    let channelId = event.json_body.context.channel_id;
    let repo = event.json_body.values.repo.value;
    await store.del(`watch:${channelId}:${repo}`);
    events.respond(event, {
        status: 200,
        body: {
            type: "ok",
            markdown: "Done!"
        }
    });
}
```