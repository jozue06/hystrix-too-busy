# hystrix-too-busy

Provides a back-pressure management based on hystrix command and too-busy module logic, utilizing hystrix metrics accumulation and circuit breaking capabilities to avoid false positives generated by plain toobusy-js module.

[![codecov](https://codecov.io/gh/trooba/hystrix-too-busy/branch/master/graph/badge.svg)](https://codecov.io/gh/trooba/hystrix-too-busy)
[![Build Status](https://travis-ci.org/trooba/hystrix-too-busy.svg?branch=master)](https://travis-ci.org/trooba/hystrix-too-busy) [![NPM](https://img.shields.io/npm/v/hystrix-too-busy.svg)](https://www.npmjs.com/package/hystrix-too-busy)
[![Downloads](https://img.shields.io/npm/dm/hystrix-too-busy.svg)](http://npm-stat.com/charts.html?package=hystrix-too-busy)
[![Known Vulnerabilities](https://snyk.io/test/github/trooba/hystrix-too-busy/badge.svg)](https://snyk.io/test/github/trooba/hystrix-too-busy)

### What is it?

The importance of maintaining the stability of the system is hard to argue. There are many different techniques on how to achieve this. One of them is based on determining how busy the system based on a latency related to event loop by measuring between expected and actual when measure event happens.

One of the popular modules is [toobusy-js](https://www.npmjs.com/package/toobusy-js), but it does not work as expected and can generate false positives (busy signal) under relatively small load (20% CPU) due to GC or some other events. What was missing is a floating window of measurements over specific period of time which would provide better data to decide if the system is really busy.

[Hystrix](https://www.npmjs.com/package/hystrixjs) component provides implements a fail fast pattern on top of the statistics it collects and that makes it a perfect candidate to add to toobusy-js module to close the gap.

The module defines a hystrix command that executes toobusy check and emits back an error which is used by hystrix to calculate number of errors towards circuit opening.

Once the circuit is open it will stay open till the next sleep window, which will lead to generating a constant signal to the client that the system is busy.

Since it is based on hystrix, it makes all the statistic available to [hystrix dashboard](https://github.com/dimichgh/hystrix-dashboard) and can be integrated into the same node app or plugged into standalone hystrix dashboard.

### Install

```
$ npm install hystrix-too-busy -S
```

### Usage

```js
require('hystrix-too-busy').getState(busy => {
    console.log('I am', busy ? 'busy' : 'free');
})
```

### Configuration

Since this module is based on both too-busy and hystrixjs you need to understand how different parameters affect the outcome, which in our case is a signal that the system is in stress mode and need to shed some load.

* toobusy settings:
    * latencyThreshold (default 70) is a number if milliseconds that defines a threshold beyond which toobusy module would generate positive signal that the system is busy.
    * interval (default 500) is a number in milliseconds that defines intervals at which to calculate the system state.
    * smoothingFactor (default 0.33) is smoothing factor used by toobusy module to calculate how system is busy.

* [hystrix settings](https://github.com/Netflix/Hystrix/wiki/Configuration#CommandCircuitBreaker):
    * circuitBreakerErrorThresholdPercentage (default 50) defines an a threshold for toobusy signal
    * circuitBreakerForceClosed (default false) forces to always keep the circuit closed, which is equal to disabling the module functionality.
    * circuitBreakerForceOpened (default false) forces to always keep the circuit open, which is equal to always busy.
    * circuitBreakerRequestVolumeThreshold (default 20) is a number of requests to the module after which it should start checking if the system is busy
    * circuitBreakerSleepWindowInMilliseconds (default 5000) is an interval after which it should attempt to close the circuit or in other words, check again if the system is busy after being marked as busy.
    * requestVolumeRejectionThreshold (default 0 (off)) defines a number of request to the module after which the requests will be immediately rejected, i.e. marked busy disregarding the actual business of the system. Note by default it is off.
    * statisticalWindowNumberOfBuckets (default 10) is number of buckets used to calculate the stats.
    * statisticalWindowLength (default 10000) defines statistical window in milliseconds.
    * percentileWindowNumberOfBuckets (default 6) defines a number of buckets to calculate percentile stats.
    * percentileWindowLength (default 60000) defines percentile window length in milliseconds.
