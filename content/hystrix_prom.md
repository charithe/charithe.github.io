Title: Integrating Hystrix And Exposing Metrics To Prometheus
Date: 2017-01-09 18:08
Category: Programming
Tags: Java, Hystrix, Prometheus, Monitoring, Metrics, Rx

[Hystrix](https://github.com/Netflix/Hystrix) is a well-known fault tolerance library for distributed systems. If your 
application needs to interact with remote systems (or even micro-services running in the same data center), Hystrix will
provide latency-sensitive routing, retries, request coalescing, circuit breakers and many more fault-tolerance features out of the box.
I cannot do much justice to Hystrix in this short brain dump but there are many great articles and presentations around the web that
are well worth perusing.



Integrating Hystrix into an application can be a bit cumbersome as every external interaction needs to be defined as a Hystrix command.
One method I have successfully used to reduce the amount of integration code is to write a wrapper class.

```java
import com.netflix.hystrix.HystrixCommand;
import com.netflix.hystrix.HystrixCommandGroupKey;
import com.netflix.hystrix.HystrixCommandKey;
import rx.Single;

import java.util.function.Supplier;

// This assumes that your code is already written using RxJava. Hystrix also provides methods for wrapping the result
// as an Observable as well
public class CommandWrapper<T> extends HystrixCommand<Single<T>> {
    // if there are multiple logically related commands, this should be made into a parameter as well 
    private static final String COMMAND_GROUP_KEY = "MyCommandGroup";
    private final Supplier<Single<T>> wrappedFn;
    private final String commandKey;

    public static <T> CommandWrapper<T> of(String commandKey, Supplier<Single<T>> wrappedFn) {
        return new CommandWrapper<>(commandKey, wrappedFn);
    }

    CommandWrapper(String commandKey, Supplier<Single<T>> wrappedFn) {
        super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey(COMMAND_GROUP_KEY))
                  .andCommandKey(HystrixCommandKey.Factory.asKey(commandKey)));
        this.commandKey = commandKey;
        this.wrappedFn = wrappedFn;
    }

    @Override
    protected Single<T> run() throws Exception {
        return wrappedFn.get();
    }

    @Override
    public HystrixCommandKey getCommandKey() {
        return HystrixCommandKey.Factory.asKey(commandKey);
    }
}
```

The above wrapper allows your core logic to be written without worrying about Hystrix and yet reap its' benefits  as follows:

```java
public Single<ApiToken> login() {
    return CommandWrapper.of("LoginToSystemX", client::login).execute();
}
```



Prometheus Reporting And Hystrix Dashboard
------------------------------------------

Hystrix provides a comprehensive set of metrics that can be viewed live using the [Hystrix Dashboard](https://github.com/Netflix/Hystrix/tree/master/hystrix-dashboard) or collected using metrics backends such as Netflix Servo. The Hystrix distribution provides a Servlet handler for these integrations but what if the application is not running inside a servlet container and the preferred metrics backend is [Prometheus](https://prometheus.io/). 

Two open-source projects come to our rescue here.

- [Hystrix RxNetty Metrics Stream](https://github.com/Netflix/Hystrix/tree/master/hystrix-contrib/hystrix-rx-netty-metrics-stream)
- [SoundCloud's Prometheus-Hystrix](https://github.com/soundcloud/prometheus-hystrix)

Unfortunately, SoundCloud `prometheus-hystrix` library is not released on Maven Central at the time of writing. You can instead make use of 
[jitpack.io](https://jitpack.io/) to obtain a build direct from the source release itself. Refer to [https://jitpack.io/#soundcloud/prometheus-hystrix](https://jitpack.io/#soundcloud/prometheus-hystrix) for more information. 


In Gradle, the dependency requirements are:

```
compile group: 'com.github.soundcloud', name: 'prometheus-hystrix', version: '3.1.0'
compile group: 'com.netflix.hystrix', name: 'hystrix-rx-netty-metrics-stream', version: '1.5.8'
```

In order to expose Hystrix metrics in the Prometheus format, we need to write a little bit of code:

```java
import io.netty.buffer.ByteBuf;
import io.netty.buffer.ByteBufOutputStream;
import io.netty.buffer.Unpooled;
import io.netty.handler.codec.http.HttpHeaderNames;
import io.netty.handler.codec.http.HttpResponseStatus;
import io.prometheus.client.CollectorRegistry;
import io.prometheus.client.exporter.common.TextFormat;
import io.reactivex.netty.protocol.http.server.HttpServerResponse;
import rx.Observable;

import java.io.BufferedWriter;
import java.io.IOException;
import java.io.OutputStreamWriter;

public class PrometheusMetrics {
    public static Observable<Void> dumpMetrics(HttpServerResponse<ByteBuf> response) {
        ByteBuf buffer = Unpooled.buffer();
        try (BufferedWriter bufWriter = new BufferedWriter(new OutputStreamWriter(new ByteBufOutputStream(buffer)))) {
            TextFormat.write004(bufWriter, CollectorRegistry.defaultRegistry.metricFamilySamples());
        } catch (IOException e) {
            response.setStatus(HttpResponseStatus.INTERNAL_SERVER_ERROR);
            return response.writeStringAndFlush("ERROR");
        }

        response.setStatus(HttpResponseStatus.OK);
        response.getHeaders().add(HttpHeaderNames.CONTENT_TYPE, TextFormat.CONTENT_TYPE_004);
        return response.writeAndFlush(buffer);
    }
}
```


Then, we can write a RxNetty server handler to expose both the Hystrix Dashboard stream and the Prometheus stream as follows:

```java
public class MetricsEndpoint {
	private static final int METRICS_PORT = 5555;

    public static HttpServer<ByteBuf, ByteBuf> start() {
        return RxNetty.createHttpServer(METRICS_PORT,
                                        new HystrixMetricsStreamHandler<>(new MonitorHandler())).start();
    }

	static class MonitorHandler implements RequestHandler<ByteBuf, ByteBuf> {
    	private static final String METRICS_PATH = "/metrics";

    	@Override
    	public Observable<Void> handle(HttpServerRequest<ByteBuf> request, HttpServerResponse<ByteBuf> response) {
        	if (request.getPath().startsWith(METRICS_PATH)) {
            	return PrometheusMetrics.dumpMetrics(response);
        	}

        	response.setStatus(HttpResponseStatus.BAD_REQUEST);
        	return response.writeStringAndFlush("BAD REQUEST");
    	}
	}
}
```

Now once the server is started, the Hystrix Dashboard can be pointed to `<server_address>:5555` for the live graphs and
the Prometheus scraper can obtain the metrics via `<server_address>:5555/metrics` as well.
