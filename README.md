grpcbox
=====

Library for creating [grpc](https://grpc.io) servers in Erlang, based on the [chatterbox](https://github.com/joedevivo/chatterbox) http2 server.

Very much still alpha quality.

Implementing a Service
----

The easiest way to get started is using the plugin, [grpcbox_plugin](https://github.com/tsloughter/grpcbox_plugin):

```erlang
{deps, [grpcbox]}.

{grpc, [{protos, "priv/protos"},
        {gpb_opts, [{module_name_suffix, "_pb"}]}]}.

{plugins, [grpcbox_plugin]}.
```

Currently `grpcbox` and the plugin are a bit picky and the `gpb` options will always include `[use_packages, maps, {i, "."}, {o, "src"}]`.

Assuming the `priv/protos` directory of your application has the `route_guide.proto` found in this repo, `priv/protos/route_guide.proto`, the output from running the plugin will be:

```shell
$ rebar3 grpc gen
===> Writing src/route_guide_pb.erl
===> Writing src/grpcbox_route_guide_bhvr.erl
```

A behaviour is used because it provides a way to generate the interface and types without being where the actual implementation is also done. This way if a change happens to the proto you can regenerate the interface without any issues with the implementation of the service, simply then update the implemntation callbacks to match the changed interface.

#### Unary RPC

Unary RPCs receive a single request and return a single response. The RPC `GetFeature` takes a single `Point` and returns the `Feature` at that point:

```protobuf
rpc GetFeature(Point) returns (Feature) {}
```

The callback generated by the `grpcbox_plugin` will look like:

```erlang
-callback get_feature(ctx:ctx(), route_guide_pb:'grpcbox.Point'()) ->
    {ok, route_guide_pb:'grpcbox.Feature'()} | {error, term()}.
```

And the implementation is as simple as an Erlang function that takes the arguments `Ctx`, the context of this current request, and a `Point` map, returning a `Feature` map:

```erlang
get_feature(Ctx, Point) ->
    Feature = #{name => find_point(Point, data()),
                location => Point},
    {ok, Feature, Ctx}.
```

#### Streaming Output

Instead of returning a single feature the server can stream a response of multiple features by defining the RPC to have a `stream Feature` return:

```protobuf
rpc ListFeatures(Rectangle) returns (stream Feature) {}
```

In this case the callback still receives a map argument but also a `grpcbox_stream` argument:

```erlang
-callback list_features(route_guide_pb:rectangle(), grpcbox_stream:t()) ->
    ok | {error, term()}.
```

The `GrpcStream` variable is passed to `grpcbox_stream:send/2` for returning an individual feature over the stream to the client. The stream is ended by the server when the function completes.

```erlang
list_features(_Message, GrpcStream) ->
    grpcbox_stream:send(#{name => <<"Tour Eiffel">>,
                                        location => #{latitude => 3,
                                                      longitude => 5}}, GrpcStream),
    grpcbox_stream:send(#{name => <<"Louvre">>,
                          location => #{latitude => 4,
                                        longitude => 5}}, GrpcStream),
    ok.
```

#### Streaming Input

The client can also stream a sequence of messages:

```protobuf
rpc RecordRoute(stream Point) returns (RouteSummary) {}
```

In this case the callback receives a `reference()` instead of a direct value from the client:

```erlang
-callback record_route(reference(), grpcbox_stream:t()) ->
    {ok, route_guide_pb:route_summary()} | {error, term()}.
```

The process the callback is running in will receive the individual messages on the stream as tuples `{reference(), route_guide_pb:point()}`. The end of the stream is sent as the message `{reference(), eos}` at which point the function can return the response:

```erlang
record_route(Ref, GrpcStream) ->
    record_route(Ref, #{t_start => erlang:system_time(1),
                            acc => []}, GrpcStream).

record_route(Ref, Data=#{t_start := T0, acc := Points}, GrpcStream) ->
    receive
        {Ref, eos} ->
            {ok, #{elapsed_time => erlang:system_time(1) - T0,
                   point_count => length(Points),
                   feature_count => count_features(Points),
                   distance => distance(Points)}, GrpcStream};
        {Ref, Point} ->
            record_route(Ref, Data#{acc => [Point | Points]}, GrpcStream)
    end.
```

#### Streaming In and Out

A bidrectional streaming RPC is defined when both input and output are streams:
 
```protobuf
rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
```

```erlang
-callback route_chat(reference(), grpcbox_stream:t()) ->
    ok | {error, term()}.
```

The sequence of input messages will again be sent to the callback's process as Erlang messages and any output messages are sent to the client with `grpcbox_stream`:

```erlang
route_chat(Ref, GrpcStream) ->
    route_chat(Ref, [], GrpcStream).

route_chat(Ref, Data, GrpcStream) ->
    receive
        {Ref, eos} ->
            ok;
        {Ref, #{location := Location} = P} ->
            Messages = proplists:get_all_values(Location, Data),
            [grpcbox_stream:send(Message, GrpcStream) || Message <- Messages],
            route_chat(Ref, [{Location, P} | Data], GrpcStream)
    end.
```

#### Interceptors

##### Unary Interceptor

A unary interceptor can be any function that accepts a context, decoded request body, server info map and the method function:

```erlang
some_unary_interceptor(Ctx, Request, ServerInfo, Fun) ->
    %% do some interception stuff
    Fun(Ctx, Request).
```

The interceptor is configured in the `grpc_opts` set in the environment or passed to the supervisor `start_child` function. An example from the test suite sets `grpc_opts` in the application environment:

```erlang
#{service_protos => [route_guide_pb],
  unary_interceptor => fun(Ctx, Req, _, Method) ->
                         Method(Ctx, #{latitude => 30,
                                       longitude => 90})
                       end}
```

##### Streaming Interceptor

##### Middleware

There is a provided interceptor `grpcbox_chain_interceptor` which accepts a list of interceptors to apply in order, with the final interceptor calling the method handler. An example from the test suite adds a trailer in each interceptor to show the chain working:

```erlang
#{service_protos => [route_guide_pb],
  unary_interceptor =>
    grpcbox_chain_interceptor:unary([fun ?MODULE:one/4,
                                     fun ?MODULE:two/4,
                                     fun ?MODULE:three/4])}
```

#### Tracing and Statistics

The provided interceptor `grpcbox_trace` supports the [OpenCensus](http://opencensus.io/) wire protocol using [opencensus-erlang](https://github.com/census-instrumentation/opencensus-erlang). It will use the `trace_id`, `span_id` and any options or tags from the trace context.

Configure as an interceptor:

```erlang
#{service_protos => [route_guide_pb],
  unary_interceptor => {grpcbox_trace, unary}}
```

Or as a middleware in the chain interceptor:

```erlang
#{service_protos => [route_guide_pb],
  unary_interceptor =>
    grpcbox_chain_interceptor:unary([..., 
                                     fun grpcbox_trace:unary/4, 
                                     ...])}
```

See [opencensus-erlang](https://github.com/census-instrumentation/opencensus-erlang) for details on configuring reporters.

#### Metadata

Metadata is sent in headers and trailers.

CT Tests
---

To run the Common Test suite:

```
$ rebar3 ct
```

Interop Tests
---

The `interop` rebar3 profile builds with an implementation of the `test.proto` for grpc interop testing:

```
$ rebar3 as interop shell

> grpcbox_sup:start_child().
```

With the shell running the tests can then be run from a script:

```
$ interop/run_tests.sh
```

The script by default uses the Go test client that can be installed with the following:

```
$ go get -u github.com/grpc/grpc-go/interop
$ go build -o $GOPATH/bin/go-grpc-interop-client github.com/grpc/grpc-go/interop/client
```
