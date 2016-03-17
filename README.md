# Apollo Metrics

This module integrates the [semantic-metrics](https://github.com/spotify/semantic-metrics) library
with the Apollo request/response handling facilities.

Including this this module in your assemly means that Apollo will add some metrics tracking
requests per endpoint.

The metrics will be tagged with the following tags:

| tag         | value                      | comment                                              |
|-------------|----------------------------|------------------------------------------------------|
| service     | name of Apollo application |                                                      |
| endpoint    | method:uri-path-segment    | For instance, GET:/android/v1/search-artists/<query> |
| component   | "service-request"          |                                                      |

The following metrics are recorded:

### Endpoint request rate

A Meter metric, per status-code, tagged with:

| tag         | value                      | comment                                              |
|-------------|----------------------------|------------------------------------------------------|
| what        | "endpoint-request-rate"    |                                                      |
| status-code | *                          | "200", "404", "418", etc.                            |
| unit        | "request"                  |                                                      |

### Endpoint request duration

A Timer metric, tagged with:

| tag         | value                      | comment                                              |
|-------------|----------------------------|------------------------------------------------------|
| what        | "endpoint-request-duration"|                                                      |

### Fan-out

A Histogram metric, tagged with:

| tag         | value                      | comment                                              |
|-------------|----------------------------|------------------------------------------------------|
| what        | "request-fanout-factor"    |                                                      |
| unit        | "request/request"          |                                                      |

This metric will show you how many downstream requests each endpoint tends to make.


## Custom Metrics

Previous versions of Apollo have been reporting some metrics at a finer level of granularity,
for instance, adding tags for 'sending service' and 'content type' to metrics like the reply 
payload size histogram. This is a feature that is valuable for some teams, but not all, and that 
makes it harder to correctly get an idea of the totals, as well as increasing the load on our
monitoring system. Therefore, that level of granularity has been removed as a standard feature.

If you do want this, a sketch of a solution for adding server-side metrics is to create a 
```Middleware``` that you wrap your endpoint handlers with. In that middleware, you can create
```MetricId```s with the tags you want to track, and ensure that the correct metrics are generated.

For client-side metrics, wrap the ```Client``` you get from the ```RequestContext``` in a similar
way to how ```DecoratingClient``` is implemented. In your wrapper, ensure that the right metrics
are tracked.

To get a `SemanticMetricRegistry`, use:
```java
  environment.resolve(SemanticMetricRegistry.class);
```

You can then construct `MetricId`:s and emit metrics in handlers and middleware.

To tag custom metrics with endpoint information, you could pass in the endpoint name, or transform
whole routes. The example below shows both approaches.

Alternatively, you could set up an `EndpointRunnableFactoryDecorator` and register
it, similar to how the metrics module does:
```java
    Multibinder.newSetBinder(binder(), EndpointRunnableFactoryDecorator.class)
        .addBinding().to(StatsCollectingEndpointRunnableFactoryDecorator.class);
```
See [Extending incoming/outgoing request handling]
(https://github.com/spotify/apollo/tree/master/apollo-environment#extending-incomingoutgoing-request-handling)
for some description of how to do request handling decorations.


### Custom per-endpoint response payload size histogram example

```java
  /**
   * Use this method to transform a route to one that tracks response payload sizes in a Histogram,
   * tagged with an endpoint tag set to method:uri of the route.
   */
  public Route<AsyncHandler<Response<ByteString>>> withResponsePayloadSizeHistogram(
      Route<AsyncHandler<Response<ByteString>>> route) {

    String endpointName = route.method() + ":" + route.uri();

    return route.withMiddleware(responsePayloadSizeHistogram(endpointName));
  }

  /**
   * Middleware to track response payload size in a Histogram,
   * tagged with an endpoint tag set to the given endpoint name.
   */
  public Middleware<AsyncHandler<Response<ByteString>>, AsyncHandler<Response<ByteString>>>
      responsePayloadSizeHistogram(String endpointName) {

    final MetricId histogramId = MetricId.build()
        .tagged("service", serviceName)
        .tagged("endpoint", endpointName)
        .tagged("what", "endpoint-response-size");

    final Histogram histogram = registry.histogram(histogramId);

    return (inner) -> (requestContext) ->
        inner.invoke(requestContext).whenComplete(
            (response, t) -> {
              if (response != null) {
                histogram.update(response.payload().map(ByteString::size).orElse(0));
              }
            }
        );
  }
```
