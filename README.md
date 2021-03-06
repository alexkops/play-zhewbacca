# Play-security library

![Build Status - Master](https://travis-ci.org/zalando-incubator/play-zhewbacca.svg?branch=master)
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/org.zalando/play-zhewbacca_2.11/badge.svg)](https://maven-badges.herokuapp.com/maven-central/org.zalando/play-zhewbacca_2.11)
[![codecov.io](https://codecov.io/github/zalando-incubator/play-zhewbacca/coverage.svg?branch=master)](https://codecov.io/github/zalando-incubator/play-zhewbacca?branch=master)
[![Codacy Badge](https://api.codacy.com/project/badge/Grade/fa1fb822bc1246508d343880c0b1868c)](https://www.codacy.com/app/dmitrykrivaltsevich/play-zhewbacca?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=zalando-incubator/play-zhewbacca&amp;utm_campaign=Badge_Grade)
[![MIT licensed](https://img.shields.io/badge/license-MIT-blue.svg)](https://raw.githubusercontent.com/zalando-incubator/play-zhewbacca/master/LICENSE)


**[Table of Contents](http://tableofcontent.eu)**
<!-- Table of contents generated generated by http://tableofcontent.eu -->
- [Play-security library](#play-security-library)
  - [Core Technical Concepts](#core-technical-concepts)
  - [Known alternatives](#known-alternatives)
  - [Getting Started](#getting-started)
  - [Contributing](#contributing)
  - [Contact](#contact)
  - [License](#license)
  
Play! (v2.5 - 2.6) library to protect RESTful resource servers using OAuth2. According to [RFC 6749](https://tools.ietf.org/html/rfc6749#section-7):

> The client accesses protected resources by presenting the _access
  token_ to the _resource server_. The resource server MUST _validate_ the
  access token and ensure that it has not expired and that its scope
  covers the requested resource. _The methods_ used by the resource
  server _to validate the access token_ (as well as any error responses)
  _are beyond the scope_ of this specification but generally involve an
  interaction or coordination between the resource server and the
  authorization server.

However, the specification [does not define](https://tools.ietf.org/html/rfc6749#section-1.1) the interaction between the authorization server and resource server:

> The interaction between the authorization server and resource server
  is _beyond the scope_ of this specification.  The authorization server
  _may be the same server_ as the resource server _or a separate entity_.

The library assumes an existence of external server which implements [API](http://planb.readthedocs.io/en/latest/intro.html#token-info) of [Plan B Token Info service](https://github.com/zalando/planb-tokeninfo) and which is responsible for the validation of access tokens. The Plan B Token Info service API is not compatible with [OAuth 2.0 Token Introspection](https://tools.ietf.org/html/rfc7662), so the library does not cover an interaction with authorization servers which might implement token introspection. However, support of token introspection can be easily added by providing an implementation of `org.zalando.zhewbacca.AuthProvider` interface. We want to remove coupling to Plan B before the end of Q3 2017 and provide better way to integrate authorization servers (see [issue-33](https://github.com/zalando-incubator/play-zhewbacca/issues/33)).

**Note: the library does not validate access tokens on its own**.

After successful validation of access token it evaluates access rules defined to access protected resources. For example, it can compare scopes required to access protected resource with scopes associated with the token. So the main goal of this library is to reduces amount of infrastructure code needed to implement proper interaction between resource server and authorization server (in this case [Plan B Token Info service](https://github.com/zalando/planb-tokeninfo)) and evaluation of access rules.

Currently the library supports only `Bearer` access token type. In order to access a protected endpoint clients should pass an `Authorization` header with the `Bearer` token in every request. More details you can find in the [RFC 6749](https://tools.ietf.org/html/rfc6749#section-7.1).

Developers who want to use this library don't need to change their code in order to protect endpoints. All necessary security configurations happen in a separate configuration file. The main difference between this library and similar libraries is that as a user of this library you don't need to couple it with any custom annotations or "actions". This library also provides you kind of a safety net because the access to all endpoints is denied by default regardless of whether they are existing or planned.

The library is used in production in several projects, so you may consider it as reliable and stable.

## Core Technical Concepts
- Non-blocking from top to bottom.
- Non-intrusive approach for the integration with existing play applications. You don't need to change your code.
- Declarative security configuration via configuration file.
- Minimalistic. The library does only one thing, but does it good.
- Developers friendly. Just plug it in and you are done.
- Easy to use in tests with `AlwaysPassAuthProvider` provider.
- Opinionated design choice: the library relies on `Play`'s toolbox like `WS`-client from the beginning because it is designed to work _exclusively_ within Play-applications.

We mainly decided to release this library because we saw the needs of OAuth2 protection in almost every microservice we built and because we are not happy to couple our code with any forms of custom security actions or annotations.

## Known alternatives
- [Hutmann](https://github.com/zalando-incubator/hutmann)
- [Deadbolt 2](https://github.com/schaloner/deadbolt-2/)
- [SecureSocial](https://github.com/jaliss/securesocial/)

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
authorisation.iam.endpoint = "https://info.services.auth.example.com/oauth2/tokeninfo"

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
