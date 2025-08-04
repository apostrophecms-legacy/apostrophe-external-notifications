# ⛔️ **DEPRECATED** — do not use for new projects

See [our current docs](https://docs.apostrophecms.org/)

# apostrophe-external-notifications

A simple way to get notifications via Slack and other external systems when various events occur in [ApostropheCMS](https://apostrophecms.org).

## Installation

```
# In the root dir of your existing apostrophe project
npm install apostrophe-external-notifications
```

## Configuration

```javascript
  // in app.js
  modules: {
    'apostrophe-external-notifications': {
      // OPTIONAL: an alias to make it easier to send your own notifications,
      // see example below
      alias: 'external',
      platforms: {
        slack: {
          // See below for a more nuanced way to do this
          channel: [ '#apostrophe-edits' ],
          webhooks: {
            '#apostrophe-edits': 'https://hooks.slack.com/services/GO/GET-YOUR-OWN'
          }
        }
      }
    }
  }
```

> To set this up in Slack, you need to [register a Slack "app" here](https://api.slack.com/apps?new_app=1). After copying that information, click "Incoming Webhooks," then be sure to turn them on. Now click "Add New Webhook to Workspace" and select the desired channel. Repeat for each channel you wish to notify. Finally, copy and paste the resulting webhook URLs into the `webhooks` configuration as shown above.

### Sending different events to different channels

The simple configuration above sends everything to the `#apostrophe-edits` channel in Slack. You can also break down which events go to which channels:

```javascript
  modules: {
    'apostrophe-external-notifications': {
      alias: 'external',
      platforms: {
        slack: {
          events: {
            'apostrophe-workflow:afterCommit': '#apostrophe-commits',
            'apostrophe-workflow:afterExport': '#apostrophe-exports',
            'apostrophe-workflow:afterForceExport': '#apostrophe-exports'
          },
          webhooks: {
            '#apostrophe-commits': 'https://hooks.slack.com/services/GO/GET-YOUR-OWN-1',
            '#apostrophe-exports': 'https://hooks.slack.com/services/GO/GET-YOUR-OWN-2'
          }
        }
      }
    }
  }
```

If you do not configure the shared `channel` option, then **only the events you individually configure are sent to Slack at all.** You may also do it both ways.

> Anywhere you see channels configured above, you can specify **either an array of channels or a single channel**.
> 
> If you configure more than one channel, you must also create and paste in the "webhook" URLs for each of them.

## Limitations

There must be an Apostrophe promise event associated with what you want notifications for, and an external notification handler must be registered for that event. `apostrophe-external-notifications` has handlers for some popular cases, but not all.

## Built-in event listeners: what you can get without writing any code

Currently the following event handlers have listeners built into this module:

```
apostrophe-workflow:afterCommit
apostrophe-workflow:afterExport
apostrophe-workflow:afterForceExport
```

More event handlers are coming. In the meantime, you can add support for more events yourself, as shown below. We suggest doing so as a PR on the module in question so that the community benefits and you are aware when we add a handler that would otherwise duplicate yours.

## Adding support for more events

Here's how you might add support for the `afterCommit` event, if we didn't already have it. **This code assumes you gave the module an alias in your project,** as seen above.

```
self.apos.external.notifyOn('apostrophe-workflow:afterCommit', (req, commit) => 
  [ '{user} committed the {type} {title} which has these tags: {string}.', commit.from, commit.from, commit.from.tags ]
);
```

"What's going on in this code?" `apostrophe-workflow:afterCommit` is the event we want to listen for. `(req, commit)` are the arguments that the `afterCommit` event provides. We then return an array containing a template string, and arguments to replace parts of the template string. The template string can contain the following optional placeholders:

* `{user}` displays the current user's name, or falls back gracefully if there is no user. Powered by the `req` argument from the event, so we do not need to pass anything else. Not all events have `req`. If `req` is not present or contains no username `Anonymous` is sent.
* `{type}` displays the type of a document in a user-friendly way, or falls back to the `type` property. Expects a matching `doc` argument as shown above. (The `commit` object emitted by `afterCommit` has a `from` property containing the doc that was committed.)
* `{title}` displays the `title` a document in a user-friendly way, or falls back to the `slug` property. Expects a matching `doc` argument. (The `commit` object emitted by `afterCommit` has a `from` property containing the doc that was committed.)
* `{string}` simply expects and sends a string argument. If it receives an array argument, it will send it as a comma-separted string, with spaces.

> While you could do everything with `{string}`, the other placeholders save time and prevent frequent causes of crashing bugs due to missing sanity checks.

## Adding support for your own events in a published npm module

Thanks for doing that! Follow the above technique. However, **in a public npm module, you MUST NOT assume** that `apostrophe-external-notifications` has a handy alias (do NOT write `apos.external`). You also should NOT assume that the module is present at all. Instead, write:

```javascript
const external = self.apos.modules['apostrophe-external-notifications'];
if (external) {
  external.notifyOn(/* ... as seen above */);
}
```

## Adding support for more platforms

Support for Slack ships with this module by default. You can add handlers for other platforms.

Here is a simplified Slack platform handler:

```
const rp = require('request-promise');
self.apos.externals.addPlatform('slack', async (req, options, channels, message) => {
  for (const channel of channels) {
    await rp({
      method: 'POST',
      uri: options.webhooks[channel],
      form: {
        text: message.formatted
      }
    });
  }
});
```

> The above example assumes you gave the module the alias `externals`. If you want to ship support for a platform as an npm module, refer to our module as `self.apos.modules['apostrophe-externals']` to be safe.

Note that `req` **may be undefined** in cases where an event is global and not concerned with an individual request. `req` is provided in case you want to handle the message differently depending on the sender's identity. Here we do not.

`channels` contains the array of channel names the event should be sent to. **It may be empty and you should do nothing if it is empty**, unless channels are not a relevant concept for your platform. If you don't care about channels, you may wish to look at `message.event`, which contains the original ApostropheCMS promise event name.

`message.formatted` contains the message to be sent, as a string. Placeholders have already been resolved, the message is complete and ready to send.

> Although our platform handler function is `async` and `await`s each channel's message delivery to Slack, for the sake of performance `apostrophe-external-notifications` will not wait for the handler to finish before allowing the original Apostrophe event handler to return. However, for the sake of consistency the module does guarantee that notifications sent for a specific `req` will be delivered in order relative to their peers. Those with no `req` are also sent in order.
