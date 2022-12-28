# API Service

The Watchlist API is a relatively simple component. It exposes a public-facing REST API with the following endpoints:

* `GET /progress`
* `GET /progress/{assetId}`
* `PUT /progress/{assetId}`
* `DELETE /progress/{assetId}`
* `GET /watchlist`
* `PUT /watchlist/{assetId}`
* `DELETE /watchlist/{assetId}`

Details of the expected HTTP requests/responses can be found in the [OpenAPI spec](https://editor.swagger.io/?url=https://raw.githubusercontent.com/cweagans/plutotv-watchlist/main/api.yaml) (or view the [raw YAML file](https://github.com/cweagans/plutotv-watchlist/blob/main/api.yaml))

## Authentication

All requests to the Watchlist API should be authenticated by some other infrastructure component. The current spec assumes that the JWT in the `Authorization` header has been verified while en route to the service.

## Implementation notes

### Versioning

While it is included in the OpenAPI spec, it's worth mentioning again that the endpoints should all be prefaced with `/v1`. This way, changes to the service can be rolled out on some future `/v2` set of endpoints without disrupting the consumers of the current `/v1` endpoints.

### Endpoint behaviors

* The semantics of the `DELETE` endpoints in this API are a little different than you might expect. They do not check that the provided `assetId` actually exists in the platform catalog/media database. Instead, those endpoints should be viewed as a way to say "Ensure that this asset ID doesn't have any associated view progress" or "Ensure that this asset ID isn't included on the user's watchlist".
* The `PUT /watchlist/{assetId}` and `DELETE /watchlist/{assetId}` endpoints are idempotent.
* The `PUT /progress/{assetId}` endpoint does not check for existing progress data for the provided asset ID. If a user is logged in on two client devices and is watching the same asset on both devices, the last one to make the request "wins". Progress for a title will be overwritten with whatever is included in this request.

### Consumers

* It is expected that a client will send requests to `PUT /progress/{assetId}` once every few seconds while the asset is being played on the device. If a video is paused by the viewer, an additional request to the same endpoint should be sent. While this does leave the possibility of not resuming at the _exact_ same point if asset playback is resumed on another device, it will generally be close enough (and conveniently give the viewer a couple seconds of context before they start seeing content that they haven't watched yet).
* When a user has reached the end of the video content for a given asset (for instance, the credits are rolling), the client should clear the viewing progress for that asset via the `DELETE /progress/{assetId}` end point. That way, the next time the viewer watches that asset, they'll start from the beginning instead of somewhere in the middle of the credits.
* Because the Watchlist API primarily deals with asset IDs, it is recommended that the client maintain a local cache of asset data which can be queried by asset ID. It is also possible to request a full copy of the user's watchlist and viewing progress (through the `GET /watchlist` and `GET /progress` endpoints respectively). This data can be maintained in the same cache and updated periodically and/or using the `GET /progress/{assetId}` endpoint when the info page for a particular asset is viewed on the client.

### Data storage

* The API should store data in DynamoDB. This will ensure that the performance characteristics remain predictable as the service scales up and down with usage. The specific structure for the data is detailed elsewhere in this document.
* Data from DynamoDB should be read through a [DAX](https://aws.amazon.com/dynamodb/dax/) cluster to reduce latency (and potentially infrastructure cost).

## Deployment

While designing the service, my intent was that it would be built as a simple Go API, packaged as a tiny container, and run on a Kubernetes cluster with the Horizontal Pod Autoscaler enabled. However, the service could run on e.g. Lambda + API Gateway just as well (with the caveat that the JWT validation may not be as straightforward as simply relying on Istio). Nothing about the service necessarily requires a particular method of deployment. Generally speaking, this service should be deployed in a similar way to other services for the platform.

## External service dependencies

The primary dependency of the Watchlist API is some service that provides validation of asset IDs -- that it's a valid ID, that the asset is accessible given the user's location, etc. I'm assuming that there is some kind of catalog service that provides that kind of information, but without specific details about how that might work, further detail about the implementation is not possible. Broadly, given an asset ID and information about the user, I need to make a determination about whether or not an asset can be included in a user's watchlist (or have progress recorded by a particular user).

# Data store

All of the information for the Watchlist API should be stored in DynamoDB. There are two primary types of records that the system needs to keep track of: a watchlist entry and a progress entry. All of this data can be neatly packaged into a single DynamoDB table like so:

| PK | SK | AssetID | Progress |
|-|-|-|-|
| `12345` | `watchlist` | `37FC12F8-5C7F-4A0D-BCDB-32890B3911DE` |  |
| `12345` | `watchlist` | `CCB380B2-115B-4197-8C6C-977582F08D99` |  |
| `12345` | `watchlist` | `C7A859BD-603C-415C-8F47-DCAC1AAED5BF` |  |
| `12345` | `progress` | `37FC12F8-5C7F-4A0D-BCDB-32890B3911DE` | 1135 |

In this example, the partition key is the user ID (in whatever format that may be) and the sort key is the identifier for the type of data in the record. `watchlist` records only have an asset ID and `progress` records have both an asset ID and a Progress value -- the number of seconds a viewer watched of the asset.

Retrieving the data is relatively straightforward in this configuration. It's a simple matter of requesting `PK = "12345" and SK = "watchlist"` or `PK = "12345" and SK = "progress"`, optionally filtering the progress query by asset ID.

Because of the segmentation of watchlist vs progress records, updating data should be extremely quick as well - requiring a minimal number of records to be traversed to e.g. update progress for one specific asset.

# Analytics

There are a handful of events interesting from a BI perspective in this service. To expose those, the API service should send messages to an SQS queue when the following events occur:

* An asset is added to or removed from a watchlist (answering the question "how popular is this title?")
* Asset view progress is created or updated (answering the question "how long are viewers watching things?")

To ensure that the BI team is getting the best data, an message should only be sent to the SQS queue when it's an actual occurrence of that event. That is, if an asset is already on a watchlist and another request is received to add it to the watchlist, the request can complete successfully, but no SQS message should be sent.

Similarly, if asset view progress is gradually increasing, messages can be sent to SQS. If progress for an asset is some high number and reverts to a low number, a message shouldn't be sent.

Suggested payloads for each message are shown below, but should be validated with the BI team prior to implementation:

```json
{
	"event": "added-to-watchlist",
	"assetId": "37FC12F8-5C7F-4A0D-BCDB-32890B3911DE"
}
```

```json
{
	"event": "removed-from-watchlist",
	"assetId": "37FC12F8-5C7F-4A0D-BCDB-32890B3911DE"
}
```

```json
{
	"event": "viewed",
	"assetId": "37FC12F8-5C7F-4A0D-BCDB-32890B3911DE",
	"oldProgress": 1234,
	"newProgress": 3456
}
```

# Other information

(specific to the design scenario PDF)

* Decision process:
	* Generally speaking, I need something lightweight and easy to scale. Go is a great fit for that.
	* The data store is likely to be the main bottleneck in this service (since the application service can be scaled out nearly infinitely given unlimited computing resources). DynamoDB is a good choice for this kind of scenario because a) it effectively "outsources" a primary scaling concern and b) it is _made_ for this.
	* I've found that SQS can often act as a clear point of delineation between teams (or even subsystems of a larger system) -- Conway's Law at work. It's easy to send messages to, easy to consume, easy to monitor, and scaling that particular aspect of a system is (generally) somebody else's problem to solve.
* Strengths/weaknesses:
	* Strength: it is unlikely that this service will ever run into the practical limits of DynamoDB.
	* Weakness: it is extremely difficult to manually view/understand the data being passed to and from the service, as well as the data stored in the DynamoDB table.
	* Strength: using an SQS queue for a BI event stream is a simple interface to use to emit events for consumption by another team.
	* Weakness: the BI team has to consume that queue to get the data that they need, which may be additional work on their part.
