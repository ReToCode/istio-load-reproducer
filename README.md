# Service Mesh with vegeta - reproducer

## Setup
```bash
# Install Service Mesh and dependant operators
oc apply -f yaml/operators.yaml
sleep 30
oc wait --for=condition=ready pod -l name=istio-operator -n openshift-operators --timeout=300s
oc wait --for=condition=ready pod -l name=jaeger-operator -A --timeout=300s
oc wait --for=condition=ready pod -l name=kiali-operator -n openshift-operators --timeout=300s

# Create an istio instance
oc apply -f yaml/namespace.yaml
oc apply -f yaml/smcp.yaml
sleep 15
oc wait --for=condition=ready pod -l app=istiod -n istio-system --timeout=300s
oc wait --for=condition=ready pod -l app=istio-ingressgateway -n istio-system --timeout=300s
oc wait --for=condition=ready pod -l app=istio-egressgateway -n istio-system --timeout=300s

oc apply -f yaml/smmr.yaml
```


## Run the load tests

Note: you can change the load test parameters (like RPS, see full list [here](https://github.com/tsenart/vegeta)) in [vegeta-attacker.yaml](yaml/vegeta-attacker.yaml).

```bash
# Without proxies injected
sed "s|INJECT|false|g" yaml/backend.yaml | oc apply -f -
sed "s|INJECT|false|g" yaml/vegeta-attacker.yaml | oc apply -f -

# With proxies injected
sed "s|INJECT|true|g" yaml/backend.yaml | oc apply -f -
sed "s|INJECT|true|g" yaml/vegeta-attacker.yaml | oc apply -f -
```

## Results
Full logs, see [log](./log)

### 1000 RPS - with proxies

✅ This is fine:

```text
load-test-7c7pm vegeta Requests      [total, rate, throughput]         60000, 1000.01, 998.31
load-test-7c7pm vegeta Duration      [total, attack, wait]             1m0s, 1m0s, 101.926ms
load-test-7c7pm vegeta Latencies     [min, mean, 50, 90, 95, 99, max]  100.715ms, 101.984ms, 101.814ms, 102.372ms, 102.498ms, 107.589ms, 147.343ms
load-test-7c7pm vegeta Bytes In      [total, mean]                     1860000, 31.00
load-test-7c7pm vegeta Bytes Out     [total, mean]                     0, 0.00
load-test-7c7pm vegeta Success       [ratio]                           100.00%
load-test-7c7pm vegeta Status Codes  [code:count]                      200:60000  
```

### 1000 RPS - without proxies

✅ This is fine:

```text
load-test-7wjw2 Requests      [total, rate, throughput]         60000, 1000.01, 998.33
load-test-7wjw2 Duration      [total, attack, wait]             1m0s, 59.999s, 100.909ms
load-test-7wjw2 Latencies     [min, mean, 50, 90, 95, 99, max]  100.109ms, 101.221ms, 101.033ms, 101.594ms, 101.689ms, 102.721ms, 255.94ms
load-test-7wjw2 Bytes In      [total, mean]                     1860000, 31.00
load-test-7wjw2 Bytes Out     [total, mean]                     0, 0.00
load-test-7wjw2 Success       [ratio]                           100.00%
load-test-7wjw2 Status Codes  [code:count]                      200:60000  
```

### 3000 RPS - with proxies

⛔️ This fails, throughput can not be reached:

```text
load-test-fff8r vegeta Get "http://backend.demo.svc.cluster.local?sleep=100": dial tcp 0.0.0.0:0->172.30.180.70:80: bind: address already in use
load-test-fff8r vegeta Get "http://backend.demo.svc.cluster.local?sleep=100": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
load-test-fff8r vegeta Get "http://backend.demo.svc.cluster.local?sleep=100": dial tcp 0.0.0.0:0->172.30.180.70:80: bind: address already in use (Client.Timeout exceeded while awaiting headers)
load-test-fff8r vegeta Get "http://backend.demo.svc.cluster.local?sleep=100": dial tcp 0.0.0.0:0->172.30.180.70:80: i/o timeout (Client.Timeout exceeded while awaiting headers)
```

while the throughput is worse than with 1000 RPS:

```text
load-test-fff8r vegeta Requests      [total, rate, throughput]         179997, 2854.61, 353.93
load-test-fff8r vegeta Duration      [total, attack, wait]             1m32s, 1m3s, 29.427s
load-test-fff8r vegeta Latencies     [min, mean, 50, 90, 95, 99, max]  1.556ms, 45.61s, 49.305s, 1m0s, 1m1s, 1m21s, 1m28s
load-test-fff8r vegeta Bytes In      [total, mean]                     1014692, 5.64
load-test-fff8r vegeta Bytes Out     [total, mean]                     0, 0.00
load-test-fff8r vegeta Success       [ratio]                           18.18%
load-test-fff8r vegeta Status Codes  [code:count]                      0:147265  200:32732
```

### 3000 RPS - without proxies

✅ This is fine:

```text
load-test-dzwn9 Requests      [total, rate, throughput]         180000, 3000.03, 2995.03
load-test-dzwn9 Duration      [total, attack, wait]             1m0s, 59.999s, 100.183ms
load-test-dzwn9 Latencies     [min, mean, 50, 90, 95, 99, max]  100.097ms, 101.372ms, 101.04ms, 101.618ms, 101.734ms, 106.11ms, 276.156ms
load-test-dzwn9 Bytes In      [total, mean]                     5580000, 31.00
load-test-dzwn9 Bytes Out     [total, mean]                     0, 0.00
load-test-dzwn9 Success       [ratio]                           100.00%
load-test-dzwn9 Status Codes  [code:count]                      200:180000  
```

### 6000 RPS - without proxies

✅ This is fine:

```text
load-test-2z5cm Requests      [total, rate, throughput]         360001, 6000.09, 5989.99
load-test-2z5cm Duration      [total, attack, wait]             1m0s, 59.999s, 101.093ms
load-test-2z5cm Latencies     [min, mean, 50, 90, 95, 99, max]  100.09ms, 102.262ms, 101.062ms, 101.803ms, 108.415ms, 123.687ms, 351.142ms
load-test-2z5cm Bytes In      [total, mean]                     11160031, 31.00
load-test-2z5cm Bytes Out     [total, mean]                     0, 0.00
load-test-2z5cm Success       [ratio]                           100.00%
load-test-2z5cm Status Codes  [code:count]                      200:360001  
```
