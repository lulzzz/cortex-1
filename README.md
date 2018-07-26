
# WIP

**Cortex** is a fault-tolerant(raft) alerts correlation engine. 

Alerts are accepted as a standard cloudevents.io event(https://github.com/cloudevents/spec/blob/master/json-format.md). Site 24x7 and Icinga integration converters are also provided.

It collects similar events in a bucket over a time window using a regex matcher and then executes a JS(ES6) script. The script contains the correlation logic which can further create incidents or alerts. The JS environment is limited and is achieved by embedding k6.io javascript interpreter(https://docs.k6.io/docs/modules). This is an excellent module built on top of https://github.com/dop251/goja.


## Event Flow:

Steps: 

1. **Match** : alert --> (convert from site 24x7/icinga ) --> (match rule) --> **Collect**
2. **Collect** --> (add to the rule bucket which *dwells* around until the configured time) -->  **Execute**
3. **Execute** --> (flush after Dwell period) --> (execute configured script) --> *Post*
4. **Post** --> (if result is set from script, post the result to the HookEndPoint or post the bucket itself if result is nil)


The collection of events in a bucket is done by writing a rule: 

```json
{
	"title": "a test rule",
	"id": "test-rule-id-1",
	"eventTypes": ["acme.prod.icinga.check_disk", "acme.prod.site247.*"],
	"scriptID": "myscript.js",
	"dwell": 4000,
	"dwellDeadline": 3800,
	"maxDwell": 8000,
	"hookEndpoint": "http://localhost:3000/testrule",
	"hookRetry": 2
}
```


where:

*EventTypes* is the pattern of events to put in a bucket(collection of cloudevents) associated with the rule.

*Dwell* is the wait duration since the first matched event.


For this rule, incoming events with `eventType` matching one of `eventTypes` will be put in the same bucket:

```json
{
	"rule": {},
	"events": [{
		"cloudEventsVersion": "0.1",
		"eventType": "acme.prod.site247.search_down",
		"source": "site247",
		"eventID": "C234-1234-1234",
		"eventTime": "2018-04-05T17:31:00Z",
		"extensions": {
			"comExampleExtension": "value"
		},
		"contentType": "application/json",
		"data": {
			"appinfoA": "abc",
			"appinfoB": 123,
			"appinfoC": true
		}
	}]
}
```

After the `dwell` period, the configured `myscript.js` will be invoked and the bucket will be passed along:

```js
import http from "k6/http";
// result is a special variable
let result = null
// the entry function called by default
export default function(bucket) {
    bucket.events.foreach((event) => {
        // create incident or alert or do nothing
        http.Post("http://acme.com/incident")
        // if result is set. it will picked up the engine    and posted to hookEndPoint
    })
}`
```

If `result` is set, it will be posted to the hookEndPoint. The `bucket` itself will be reset and evicted from the `collect` loop. The execution `record` will then be stored and can be fetched later.

A new `bucket` will be created when an event matches the rule again.







