# Basic "Hello world" modal
Ported from [quickstart guide](https://developers.mattermost.com/integrate/apps/quick-start-go/).

# To run

To run with Matterless:

```bash
$ mls run modal-example.md
```

Install in Mattermost:
```
/apps debug-add-manifest --url http://localhost:8222/modal-example/manifest.json
/apps install --app-id matterless-modal-example
```

# events
```yaml
http:GET:/manifest.json:
  - ManifestHTTP
http:POST:/bindings:
  - BindingsHTTP
http:POST:/send-modal/submit:
  - SendFormHTTP
http:POST:/send/submit:
  - SendSubmitHTTP
```

# function ManifestHTTP
```javascript
import {events} from "./matterless.ts";

function handle(event) {
    events.respond(event, {
        status: 200,
        body: {
            "app_id": "matterless-modal-example",
            "display_name": "An app with a modal built with Matterless",
            "app_type": "http",
            "root_url": "http://localhost:8222/modal-example",
            "requested_locations": [
                "/channel_header"
            ]
        }
    });
}
```

# function BindingsHTTP
```javascript
import {events, store} from "./matterless.ts";

function handle(event) {
    events.respond(event, {
        status: 200,
        body: {
            "type": "ok",
            "data": [
                {
                    "location": "/channel_header",
                    "bindings": [
                        {
                            "location": "send-button",
                            "icon": "https://raw.githubusercontent.com/mattermost/mattermost-plugin-apps/master/examples/go/helloworld/icon.png",
                            "label":"send hello message",
                            "call": {
                                "path": "/send-modal"
                            }
                        }
                    ]
                },
            ]
        }
    });
}
```


# function SendFormHTTP
```javascript
import {events} from "./matterless.ts";

function handle(event) {
    events.respond(event, {
        status: 200,
        body: {
            "type": "form",
            "form": {
                "title": "Hello, world!",
                "icon": "https://raw.githubusercontent.com/mattermost/mattermost-plugin-apps/master/examples/go/helloworld/icon.png",
                "fields": [
                    {
                        "type": "text",
                        "name": "message",
                        "label": "message"
                    }
                ],
                "call": {
                    "path": "/send"
                }
            }
        }
    });
}
```

# function SendSubmitHTTP
```javascript
import {events} from "./matterless.ts";

async function handle(event) {
    let context = event.json_body.context;
    console.log("Got this data", event.json_body.values);
    events.respond(event, {
        status: 200,
        body: {
            status: "ok"
        }
    });
}
```