## 0.4.0

In this version, we've brought flow control capability for frequent ("hot spot") parameters, which enables detecting top-N visiting parameters and perform fine-grained rate limiting for every "hot" values (or for some specific values). Currently Sentinel Go supports basic numeric types (various int/uint/float types), bool type and string type.

### Features / Enhancements

- Support flow control for frequent ("hot spot") parameters ([#119](https://github.com/alibaba/sentinel-golang/pull/119))
- Improve pooling and time retrieval mechanism to optimize performance ([#155](https://github.com/alibaba/sentinel-golang/pull/155))
- Polish data-source helper (canonical converter and updater) for rules ([#157](https://github.com/alibaba/sentinel-golang/pull/157))
- Add fundamental benchmarks ([#154](https://github.com/alibaba/sentinel-golang/pull/154))

## 0.3.0

In this version, we've brought circuit breaking feature to Sentinel Go, which is used to provide stability and prevent cascading failures in distributed systems. Currently Sentinel Go provides two kinds of strategies: RTT-based (slow request ratio) and error-based (error ratio/error count).

### Features / Enhancements

- Add circuit breaking support ([#18](https://github.com/alibaba/sentinel-golang/pull/18), [#152](https://github.com/alibaba/sentinel-golang/pull/152))
- Refactor the mechanism of recording error in SentinelEntry/StatisticSlot and polish `api.Tracer` ([#143](https://github.com/alibaba/sentinel-golang/pull/143), [#153](https://github.com/alibaba/sentinel-golang/pull/153))
- Improve mechanism of reusing TokenResult to reduce memory footprint ([#149](https://github.com/alibaba/sentinel-golang/pull/149), [#142](https://github.com/alibaba/sentinel-golang/pull/142))
- Add etcd v3 data-source implementation ([#115](https://github.com/alibaba/sentinel-golang/pull/115))
- Add adapter for echo Web framework ([#95](https://github.com/alibaba/sentinel-golang/pull/95))
- Support carrying additional attachments with `sentinel.Entry(options)` ([#124](https://github.com/alibaba/sentinel-golang/pull/124))
- Remove unnecessary division checking for interval of SlidingWindowMetric ([#134](https://github.com/alibaba/sentinel-golang/pull/134))

## 0.2.0

### Features / Enhancements

- Add basic abstraction for data-source extension ([#73](https://github.com/alibaba/sentinel-golang/pull/73))
- Unify general configuration and logging configuration ([#56](https://github.com/alibaba/sentinel-golang/pull/56))
- Add basic error `Tracer` API ([#65](https://github.com/alibaba/sentinel-golang/pull/65), [#96](https://github.com/alibaba/sentinel-golang/pull/96))
- Add integration module for Gin web framework ([#82](https://github.com/alibaba/sentinel-golang/pull/82))
- Add integration module for gRPC-go ([#81](https://github.com/alibaba/sentinel-golang/pull/81))
- Add dubbo-go adapter module ([#60](https://github.com/alibaba/sentinel-golang/pull/60))
- Add refreshable file data-source implementation ([#86](https://github.com/alibaba/sentinel-golang/pull/86))
- Add support for collecting CPU usage for SystemRule and add `stat.system.collectIntervalMs` config item

### Bug fixes

- Fix bugs in reading logic of MetricLogSearcher