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

### 2000 RPS - with proxies

✅ This is fine:

```text
load-test-smqd2 vegeta Requests      [total, rate, throughput]         120000, 2000.01, 1996.63
load-test-smqd2 vegeta Duration      [total, attack, wait]             1m0s, 1m0s, 101.734ms
load-test-smqd2 vegeta Latencies     [min, mean, 50, 90, 95, 99, max]  100.72ms, 103.608ms, 101.882ms, 102.51ms, 102.763ms, 138.335ms, 417.715ms
load-test-smqd2 vegeta Bytes In      [total, mean]                     3720000, 31.00
load-test-smqd2 vegeta Bytes Out     [total, mean]                     0, 0.00
load-test-smqd2 vegeta Success       [ratio]                           100.00%
load-test-smqd2 vegeta Status Codes  [code:count]                      200:120000  
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

### 3000 RPS - with proxies, backend scaled to 20

```bash
 oc scale -n demo deploy/backend --replicas=20
```

✅ This is fine:

```text
load-test-p4qz5 vegeta Requests      [total, rate, throughput]         180000, 2999.99, 2994.87
load-test-p4qz5 vegeta Duration      [total, attack, wait]             1m0s, 1m0s, 102.467ms
load-test-p4qz5 vegeta Latencies     [min, mean, 50, 90, 95, 99, max]  100.745ms, 128.389ms, 102.05ms, 103.101ms, 127.932ms, 974.465ms, 1.719s
load-test-p4qz5 vegeta Bytes In      [total, mean]                     5580000, 31.00
load-test-p4qz5 vegeta Bytes Out     [total, mean]                     0, 0.00
load-test-p4qz5 vegeta Success       [ratio]                           100.00%
load-test-p4qz5 vegeta Status Codes  [code:count]                      200:180000  
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

### 6x 1000 RPS - with proxies (10 backends)

✅ This is fine:

```text
load-test-kgswv vegeta Requests      [total, rate, throughput]         60000, 1000.02, 998.33
load-test-kgswv vegeta Duration      [total, attack, wait]             1m0s, 59.999s, 101.692ms
load-test-kgswv vegeta Latencies     [min, mean, 50, 90, 95, 99, max]  100.703ms, 101.692ms, 101.554ms, 102.127ms, 102.277ms, 104.88ms, 147.502ms
load-test-kgswv vegeta Bytes In      [total, mean]                     1860000, 31.00
load-test-kgswv vegeta Bytes Out     [total, mean]                     0, 0.00
load-test-kgswv vegeta Success       [ratio]                           100.00%
load-test-kgswv vegeta Status Codes  [code:count]                      200:60000  
load-test-kgswv vegeta Error Set:
```


### 6x 1000 RPS - with proxies (20 backends)

✅ This is fine:

```text
load-test-vsj5g vegeta Requests      [total, rate, throughput]         60000, 1000.02, 998.32
load-test-vsj5g vegeta Duration      [total, attack, wait]             1m0s, 59.999s, 101.925ms
load-test-vsj5g vegeta Latencies     [min, mean, 50, 90, 95, 99, max]  100.709ms, 101.754ms, 101.645ms, 102.171ms, 102.291ms, 104.569ms, 137.75ms
load-test-vsj5g vegeta Bytes In      [total, mean]                     1860000, 31.00
load-test-vsj5g vegeta Bytes Out     [total, mean]                     0, 0.00
load-test-vsj5g vegeta Success       [ratio]                           100.00%
load-test-vsj5g vegeta Status Codes  [code:count]                      200:60000  
```

### 6x 2000 RPS - with proxies (10 backends)

✅ This is fine:

```text
  load-test-dc6k9 vegeta Requests      [total, rate, throughput]         119999, 2000.02, 1996.66
  load-test-dc6k9 vegeta Duration      [total, attack, wait]             1m0s, 59.999s, 100.695ms
  load-test-dc6k9 vegeta Latencies     [min, mean, 50, 90, 95, 99, max]  100.695ms, 102.72ms, 101.687ms, 102.352ms, 102.623ms, 116.492ms, 340.705ms
  load-test-dc6k9 vegeta Bytes In      [total, mean]                     3719969, 31.00
  load-test-dc6k9 vegeta Bytes Out     [total, mean]                     0, 0.00
  load-test-dc6k9 vegeta Success       [ratio]                           100.00%
  load-test-dc6k9 vegeta Status Codes  [code:count]                      200:119999
```
