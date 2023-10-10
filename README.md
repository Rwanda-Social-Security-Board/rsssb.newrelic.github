

# Integrating New Relic with NestJS: A Guide

**Objective**: This document offers a step-by-step guide on integrating your NestJS applications with New Relic for enhanced monitoring and error reporting.

## Table of Contents:

1. Prerequisites
2. Setting Up New Relic Account
3. Installing the New Relic Agent
4. NestJS Custom Logging Service
5. NestJS Exception Filter
6. Docker Integration

## 1. Prerequisites:

- A New Relic account.
- A NestJS application.

## 2. Setting Up Your New Relic Account:

1. [Sign up for New Relic](https://newrelic.com/signup) if you haven’t.
2. Login and select New Relic APM product.
3. Create a new application instance and note your license key.

## 3. Installing the New Relic Agent:

### Using yarn:

1. Navigate to your NestJS project directory and run:
   ```bash
   yarn add newrelic
   ```

2. In the project root, create a configuration file named `newrelic.js`. Populate this with the default configurations from `node_modules/newrelic/newrelic.js`.


```
'use strict';
/**
 * New Relic agent configuration.
 *
 * See lib/config/default.js in the agent distribution for a more complete
 * description of configuration variables and their potential values.
 */

exports.config = {
  /**
   * Array of application names.
   */
  app_name: 'ltss_backend',
  /**
   * Your New Relic license key.
   */
  license_key: 'XXXXXXXX',

  /**
   * This setting controls distributed tracing.
   * Distributed tracing lets you see the path that a request takes through your
   * distributed system. Enabling distributed tracing changes the behavior of some
   * New Relic features, so carefully consult the transition guide before you enable
   * this feature: https://docs.newrelic.com/docs/transition-guide-distributed-tracing
   * Default is true.
   */
  distributed_tracing: {
    /**
     * Enables/disables distributed tracing.
     *
     * @env NEW_RELIC_DISTRIBUTED_TRACING_ENABLED
     */
    enabled: true,
  },
  logging: {
    /**
     * Level at which to log. 'trace' is most useful to New Relic when diagnosing
     * issues with the agent, 'info' and higher will impose the least overhead on
     * production applications.
     */
    level: 'info',
  },

  application_logging: {
    forwarding: {
      enabled: true,
      max_samples_stored: 10000
    }
  },

  /**
   * When true, all request headers except for those listed in attributes.exclude
   * will be captured for all traces, unless otherwise specified in a destination's
   * attributes include/exclude lists.
   */
  allow_all_headers: true,
  attributes: {
    /**
     * Prefix of attributes to exclude from all destinations. Allows * as wildcard
     * at end.
     *
     * NOTE: If excluding headers, they must be in camelCase form to be filtered.
     *
     * @env NEW_RELIC_ATTRIBUTES_EXCLUDE
     */
    exclude: [
      'request.headers.cookie',
      'request.headers.authorization',
      'request.headers.proxyAuthorization',
      'request.headers.setCookie*',
      'request.headers.x*',
      'response.headers.cookie',
      'response.headers.authorization',
      'response.headers.proxyAuthorization',
      'response.headers.setCookie*',
      'response.headers.x*',
    ],
  },
};


```

3. Update the `license_key` and `app_name` fields in `newrelic.js`.

## 4. NestJS Custom Logging Service:

Assuming you have a custom logging service named `LoggerService`:

```typescript
import { Injectable } from '@nestjs/common';

@Injectable()
export class LoggerService {
  log(message: any) {
    console.log(message);  // Replace with your logging mechanism
  }
  
  error(message: any, trace?: string) {
    console.error(message);
    if (trace) {
      console.error(trace);
    }
  }
  
  // ... other log levels like warn, debug, etc.
}
```

## 5. NestJS Exception Filter:

Here is an example of an exception filter that uses the custom `LoggerService` and logs exceptions to New Relic:

```typescript
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { LoggerService } from './logger.service';
import { Response } from 'express';

const newrelic = require('newrelic');

@Catch()
export class AppExceptionFilter<T> implements ExceptionFilter {
  constructor(private readonly logger: LoggerService) {}

  catch(exception: T, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const status = exception instanceof HttpException ? exception.getStatus() : 500;

    const message = exception instanceof Error ? exception.message : 'Unexpected error';

    this.logger.error(message, exception.stack);

    // Log the error to New Relic
    newrelic.noticeError(exception);

    response.status(status).json({
      statusCode: status,
      message: message,
    });
  }
}
```

## 6. Docker Integration:

For a Docker integration:



2. **docker-compose.yml**:

```yaml
version: '3'

services:
  nestjs-app:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      - NEW_RELIC_LICENSE_KEY=your_license_key_here
      - NEW_RELIC_APP_NAME=Your NestJS App
    ports:
      - "3000:3000"
```

Make sure to adjust the `newrelic.js` to pick up environment variables for `license_key` and `app_name`.

3. **Run your Dockerized application**:
   ```bash
   docker-compose up --build
   ```

---


Note: In your run of the mill docker-compose application , the new relic agent will crawl and capture basic telemetry of each container . 
