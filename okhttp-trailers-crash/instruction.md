# HTTP Response Trailers Crash in Retrofit Integration

## Problem Description

Our application crashes when trying to read HTTP response trailers after using Retrofit to convert response bodies. The crash occurs with Retrofit 3 and OkHttp 5.0.0.

## Error Details

We're observing this exception:

```
java.lang.IllegalStateException: Cannot read raw response body of a converted body.
	at retrofit2.OkHttpCall$NoContentResponseBody.source(OkHttpCall.java:300)
	at okhttp3.Response.trailers(Response.kt:195)
	at misk.client.ClientMetricsInterceptorNoSummaryTest.responseCodes(ClientMetricsInterceptorNoSummaryTest.kt:107)
```

## Context

The issue happens when:
1. We use Retrofit to make HTTP requests
2. Retrofit wraps the response body with a custom implementation
3. We try to read trailers from the response
4. The custom response body throws an error because it cannot provide access to the raw source

This appears to be a compatibility issue between OkHttp's trailer reading mechanism and custom `ResponseBody` implementations that don't support direct access to the underlying source.

## Expected Behavior

Reading trailers should work regardless of whether the response body is a standard OkHttp implementation or a custom implementation like Retrofit's. The trailers functionality should not require access to the response body's source if the trailers are already available through other means.

## Testing

There is a test in `okhttp/src/jvmTest/kotlin/okhttp3/TrailersTest.kt` called `customTrailersDoNotUseResponseBody()` that reproduces this issue. The test should pass once the fix is applied.

Run the test with:
```bash
./gradlew :okhttp:test --tests "*customTrailersDoNotUseResponseBody*" --no-daemon
```

## Success Criteria

The fix should allow custom `ResponseBody` implementations to work with `Response.trailers()` without requiring access to the body's source. The test `customTrailersDoNotUseResponseBody` should pass.

