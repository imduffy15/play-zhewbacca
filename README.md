# Play-security library

![Build Status - Master](https://travis-ci.org/zalando-incubator/play-zhewbacca.svg?branch=master)
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/org.zalando/play-zhewbacca_2.11/badge.svg)](https://maven-badges.herokuapp.com/maven-central/org.zalando/play-zhewbacca_2.11)
[![codecov.io](https://codecov.io/github/zalando-incubator/play-zhewbacca/coverage.svg?branch=master)](https://codecov.io/github/zalando-incubator/play-zhewbacca?branch=master)
[![Codacy Badge](https://api.codacy.com/project/badge/Grade/fa1fb822bc1246508d343880c0b1868c)](https://www.codacy.com/app/dmitrykrivaltsevich/play-zhewbacca?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=zalando-incubator/play-zhewbacca&amp;utm_campaign=Badge_Grade)
[![MIT licensed](https://img.shields.io/badge/license-MIT-blue.svg)](https://raw.githubusercontent.com/zalando-incubator/play-zhewbacca/master/LICENSE)

Play! (v2.5) library to protect REST endpoint using OAuth2 token verification. In order to access a protected endpoint clients should pass an `Authorization` header with the `Bearer` token in every request.
This library is implemented in a non-blocking way and provides two implementations of token verification.

- `AlwaysPassAuthProvider` implementation is useful for Development environment to bypass a security mechanism. This implementation assumes that every token is valid.
- `OAuth2AuthProvider` implementation acquires token information from a 3rd-party endpoint and then verifies this info. Also it applies a [Circuit Breaker](http://doc.akka.io/docs/akka/snapshot/common/circuitbreaker.html) pattern.

Clients of this library don't need to change their code in order to protect endpoints. All necessary security configurations happen in a separate configuration file. The main difference between this library and similar libraries is that as a user of this library you don't need to change your code or somehow couple it any custom annotations or "actions". This library also provides you kind of a safety net because the access to all endpoints regardless of whether they are existing or planned is denied by default.

The library is used in production in several projects, so you may consider it as reliable and stable.

## Core Technical Concepts
- Non-intrusive approach for the integration with existing play applications. You don't need to change your code.
- Declarative security configuration via configuration file.
- Minimalistic. The library does only one thing, but does it good.
- Developers friendly. Just plug it in and you are done.
- Non-blocking from top to bottom.
- Opinionated design choice: the library relies on `Play`'s toolbox like `WS`-client from the beginning because it is designed to work _exclusively_ within Play-applications.

We mainly decided to release this library because we saw the needs of OAuth2 protection in almost every microservice we built and because we are not happy to couple our code with any forms of custom security actions or annotations.

## Getting Started

Configure libraries dependencies in your `build.sbt`:

```scala
libraryDependencies += "org.zalando" %% "play-zhewbacca" % "0.2.2"
```

To configure Development environment:

```scala
package modules

import com.google.inject.AbstractModule
import org.zalando.zhewbacca._
import org.zalando.zhewbacca.metrics.{NoOpPlugableMetrics, PlugableMetrics}

class DevModule extends AbstractModule {
  val TestTokenInfo = TokenInfo("", Scope.Default, "token type", "user uid")
  
  override def configure(): Unit = {
    bind(classOf[PlugableMetrics]).to(classOf[NoOpPlugableMetrics])
    bind(classOf[AuthProvider]).toInstance(new AlwaysPassAuthProvider(TestTokenInfo))
  }
  
}
```

For Production environment use:

```scala
package modules

import com.google.inject.{ TypeLiteral, AbstractModule }
import org.zalando.zhewbacca._
import org.zalando.zhewbacca.metrics.{NoOpPlugableMetrics, PlugableMetrics}

import scala.concurrent.Future

class ProdModule extends AbstractModule {

  override def configure(): Unit = {
    bind(classOf[AuthProvider]).to(classOf[OAuth2AuthProvider])
    bind(classOf[PlugableMetrics]).to(classOf[NoOpPlugableMetrics])
    bind(new TypeLiteral[(OAuth2Token) => Future[Option[TokenInfo]]]() {}).to(classOf[IAMClient])
  }

}
```

By default no metrics mechanism is used. User can implement ```PlugableMetrics``` to gather some simple metrics.
See ```org.zalando.zhewbacca.IAMClient``` to learn what can be measured.

You need to include `org.zalando.zhewbacca.SecurityFilter` into your applications' filters:

```scala
package filters

import javax.inject.Inject
import org.zalando.zhewbacca.SecurityFilter
import play.api.http.HttpFilters
import play.api.mvc.EssentialFilter

class MyFilters @Inject() (securityFilter: SecurityFilter) extends HttpFilters {
  val filters: Seq[EssentialFilter] = Seq(securityFilter)
}
```

and then add `play.http.filters = filters.MyFilters` line to your `application.conf`. `SecurityFilter` rejects any requests to any endpoint which does not have a corresponding rule in the `security_rules.conf` file.

Example of configuration in `application.conf` file:

```
# Full URL for authorization endpoint
authorisation.iam.endpoint = "https://info.services.auth.zalando.com/oauth2/tokeninfo"

# Maximum number of failures before opening the circuit
authorisation.iam.cb.maxFailures = 4

# Duration in milliseconds after which to consider a call a failure
authorisation.iam.cb.callTimeout = 2000

# Duration in milliseconds after which to attempt to close the circuit
authorisation.iam.cb.resetTimeout = 60000

# Maximum number of retries
authorisation.iam.maxRetries = 3

# Duration in milliseconds of the exponential backoff
authorisation.iam.retry.backoff.duration = 100

# IAMClient depends on Play internal WS client so it also has to be configured.
# The maximum time to wait when connecting to the remote host.
# Play's default is 120 seconds
play.ws.timeout.connection = 2000

# The maximum time the request can stay idle when connetion is established but waiting for more data
# Play's default is 120 seconds
play.ws.timeout.idle = 2000

# The total time you accept a request to take. It will be interrupted, whatever if the remote host is still sending data.
# Play's default is none, to allow stream consuming.
play.ws.timeout.request = 2000

play.http.filters = filters.MyFilters

play.modules.enabled += "modules.ProdModule"
```

By default this library reads the security configuration from the `conf/security_rules.conf` file. You can change the file name by specifying a value for the key `authorisation.rules.file` in your `application.conf` file.

```
# This is an example of production-ready configuration security configuration.
# You can copy it from here and paste right into your `conf/security_rules.conf` file.
rules = [
    # All GET requests to /api/my-resource has to have a valid OAuth2 token for scopes: uid, scop1, scope2, scope3
    {
        method: GET
        pathRegex: "/api/my-resource/.*"
        scopes: ["uid", "scope1", "scope2", "scope3"]
    }

    # POST requests to /api/my-resource require only scope2.write scope
    {
        method: POST
        pathRegex: "/api/my-resource/.*"
        scopes: ["scope2.write"]
    }

    # GET requests to /bar resources allowed to be without OAuth2 token
    {
        method: GET
        pathRegex: /bar
        allowed: true
    }

    # 'Catch All' rule will immidiately reject all requests for all other endpoints
    {
        method: GET
        pathRegex: "/.*"
        allowed: false      // this is an example of inline comment
    }
]
```

The following example demonstrates how you can get access to the Token Info object inside your controller:

```scala
package controllers

import javax.inject.Inject

import org.zalando.zhewbacca.TokenInfoConverter._
import org.zalando.zhewbacca.TokenInfo

class SeoDescriptionController @Inject() extends Controller {

  def create(uid: String): Action[AnyContent] = Action { request =>
    val tokenInfo: TokenInfo = request.tokenInfo
    // do something with token info. For example, read user's UID: tokenInfo.userUid
  }
}

```

## Contributing
Your contributions are highly welcome! To start please read the [Contributing Guideline](CONTRIBUTING.md).

## Contact
Please drop us an email in case of any doubts. The actual list of maintainers with email addresses you can find in the [MAINTAINERS](MAINTAINERS) file.

## License
The MIT License (MIT)

Copyright (c) 2015 Zalando SE

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
