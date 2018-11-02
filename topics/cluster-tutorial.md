# Redis Cluster Turorial

본 문서에서는 레디스 클러스터에 대해 가볍게 다룬다. 분산시스템에 대한 복잡한 내용은 다루지 않는다. 레디스 클러스터를 셋업하고 테스트하고 동작하는 방법을 다루며, 유저 입장에서 전체 시스템이 어떻게 연계되어있는지에 대해 간략히 설명한다. 클러스터 시스템의 자세한 내용은 [Redis Cluster Specification](https://redis.io/topics/cluster-spec)에서 알아보도록 하자. 

레디스 클러스터의 가용성 (availability)과 일관성 (consistency) 특징에 대해서는 최종 유저 입장에서 간단한 방식을 통해 기술한다.

단, 본 문서에서 다루는 내용은 레디스 3.0 이상의 버전을 대상으로 함을 명심해야 한다.

실제 프로덕트에 레디스 클러스터를 사용하고자 할 경우에는, Redis Cluster Specification을 읽어보길 권장한다. 그 외의 경우에는 본 문서에 따라서 레디스 클러스터에 익숙해지고, 몇번 테스트 가동을 해본 뒤에 스펙 문서를 읽어도 늦지 않다.

### Redis Cluster 101

레디스 클러스터는 **데이터들이 여러개의 레디스 노드로 자동적으로 분산**되도록 하는 레디스 서버 설치를 제공한다. 또한, 데이터를 파티셔닝 하는 동안 몇가지 설정을 통해 **다양한 단계의 가용성 레벨을 제공**하고 있다. 여기서 말하는 가용성이란 실질 사용상황에서 어떤 노드들이 통신 가동/통신이 불가능한 상황이 될 경우에도, 지속적으로 레디스 서버를 사용할 수 있도록 하는 기능적인 측면을 말한다. 그러나, 일부의 기능적 문제가 아닌 대다수의 마스터 노드가 사용 불가능 상태로 빠질경우에는 수동 복구를 진행해야 한다.

레디스 클러스터 사용을 통해 얻을 수 있는 실질적인 이득을 정리하자면 아래와 같다.

- **다수의 노드로 데이터 셋을 자동적으로 분산 시킬 수 있음**
- **일부의 노드가 동작 불능 상태일 때도 지속적으로 서버를 유지할 수 잇는 가용성을 제공**

### 레디스 클러스터 TCP 포트

모든 레디스 클러스터 노드는 두개의 TCP 연결을 유지한다. 하나는 클라이언트와의 연결을 위한 일반적은 레디스 서버 TCP 포트 이고 (기본값: 6379), 하나는 클러스터 유지를 위한 포트 이다. (기본값: 데이터 포트 + 10000 = 16379)

두번째로 열리는 포트의 경우 클러스터 유지를 위한 클러스터 버스로, 바이너리 프로토콜을 통해 노드간의 통신을 수행하는 채널이다. 클러스터 버스는 고장 감지 (failure detection), 설정 갱신, 장애 극복 프로토콜 승인 및 다양한 작업을 수행하는데 사용이 된다. 클라이언트에서는 절대로 클러스터 버스를 사용하면 안되고, 일반 포트를 사용해야 한다. 단, 시스템의 방화벽상에는 데이터 포트 및 클러스터 버스 포트를 모두 허용으로 추가해야만 레디스 클러스터가 정상적으로 동작한다. 

클라이언트를 위한 데이터/커맨드 포트와 클러스터 버스 포트의 번호는 항상 10000의 차이를 갖도록 고정이 되어 있다.

레디스 클러스터가 정상적으로 동작하기 위해서는 각 노드별로 아래의 조건을 충족해야 한다.

1. 일반적인 클라이언트 통신 포트 (기본 6379)는 모든 클라이언트들이 접근할 수 있어야 하고, 다른 모든 클러스터 노드들에서도 접근이 가능해야 한다. (클러스터 노드끼리 키를 migration 하는 데에 필요하다)
2. 클러스터 버스 포트 (기본 클라이언트 포트 + 10000)는 다른 모든 클러스터 노드로부터 접근 가능해야 한다.

둘 중 하나의 포트라도 연결되지 않을 경우 클러스터가 예상 한데로 동작하지 않을 것이다. 클러스터 버스 포트는 클라이언트 포트와 다르게, 낮은 대역폭과 낮은 프로세싱 타임을 제공하는 바이너리 프로토콜을 통해 데이터 및 정보교환을 수행한다.

### Docker 환경에서의 레디스 클러스터

현재 레디스 클러스터 버전은 NAT 시스템이나 IP 주소, TCP 포트 리매핑 시스템을 지원하지 않는다.

Docker의 경우 도커 내부에서 동작하는 프로그램들에 대해 도커 자체적인 포트를 부여하여 다수의 도커 컨테이너를 하나의 포트로 관리하는데 사용하는 port mapping 기술을 사용한다.

레디스 클러스터를 도커 내부에서 사용하기 위해서는 도커의 네트워크 모드를 **host networking mode**로 변경해서 직접적인 접근을 할 수 있도록 사용해야 한다. [도커 문서](https://docs.docker.com/engine/userguide/networking/dockernetworks/)에서 `--net=host` 옵션에 대해 참조하길 바란다.

### 레디스 클러스터의 데이터 샤딩

레디스 클러스터는 consistent 해싱을 사용하지 않고, 개념적으로 모든 키들을 하나의 **해시 슬롯**의 일부로 고려한 샤딩 기법을 사용한다.

레디스 클러스터는 기본적으로 키들에 대해 16,384개의 해시 슬롯을 유지하며, 주어진 키에 대한 해시 슬롯이 어느 위치인지를 판별한다. 판별하는 방법은 간단하게 키의 CRC16 에 대한 16384의 모듈러를 취한다.

레디스 클러스터의 모든 노드들은 각자의 해시 슬롯 파트에 대해 유지 책임이 있다. 예를 들어 3개의 클러스터 노드를 유지할 경우 아래와 같은 설정이 진행된다.

- A 노드는 0부터 5500번 해시 슬롯
- B 노드는 5501번부터 11000번 해시 슬롯
- C 노드는 11001 부터 16383번 해시 슬롯

이러한 단순 설정을 통해 새로운 노드를 추가하거나 기존 노드를 삭제할테 설정을 빠르게 바꿀 수 있도록 한다. 새로운 노드 D를 추가하고 싶을 경우 A, B, C 노드로부터 일부의 해시 슬롯을 D로 이관하기만 하면 된다. 노드 A를 삭제하고 싶은 경우에는 A 노드의 해시 슬롯들을 B와 C로 분배하면 된다. 이후 노드 A의 데이터를 삭제하는 작업을 수행한 뒤 모두 삭제 된 경우 클러스터에서 A 노드를 제거하면 된다.

해시 슬롯을 이용하여 데이터를 이동 시키는 방법은 전체 레디스 클러스터의 동작을 멈추지 않아도 되는 장점이 있다. 

레디스 클러스터는 multiple key operation을 지원하여 단일 커맨드에 들어 있는 모든 키들에 대해 다 같은 해시 슬롯에 저장되는 기술을 제공한다. 사용자는 *hash tags* 를 사용하여 다 같은 해시 슬롯으로 가도록 강제 할 수 있다.

해시 태그는 레디스 클러스터 스펙 문서에 작성되어 있다. 기능이 요지는 다수의 키들에 대해 중괄호 {}로 표시한 부분이 공통 부분 문자열일 경우 하나의 커맨드로 다수의 키를 인자로 사용할 수 있도록 하는 것이다. 예를 들어 this{foo}key라는 키와 another{foo}key라는 키가 있을 경우 괄호로 묶은 {foo} 부분이 동일하므로 하나의 같은 슬롯에 저장되어 있음이 보장된다. 이 때에는 {} 안의 값만 해싱이 된다.

### 레디스 클러스터의 마스터-슬레이브 모델

마스터 노드의 일부가 고장이 나거나 통신 불능 상태에 빠질 경우에도 전체 클러스터의 정상적인 동작을 위해, 레디스 클러스터는 마스터-슬레이브 모델도 지원한다. 이를 통해 모든 해시 슬롯에 대해 기본 1부터 N개의 레플리카를 유지할 수 있도록 한다.

위에서 예시를 들었던 A, B, C 노드로 구성된 클러스터 상황에서 B 노드가 더이상 동작할 수 없는 상황에 빠질 경우 5501번 부터 11000번 까지의 해시 슬롯에 대해 더 이상 레디스 서버를 유지 할 수 없게 된다.

그러나 각 클러스터 노드를 생성할 때 각 노드에 대한 슬레이브 노드를 추가로 생성한뒤, 레플리케이션을 할 경우 A, B, C 노드 중 어느 노드가 죽더라도 A1, B1, C1으로 정의되는 슬레이브 노드를 통해 죽은 노드를 대체 할 수 있다.

슬레이브 노드가 사용되는 경우 해당 슬레이브 노드를 마스터 노드로 승격 시키며, 이후에 들어오는 모든 노드를 승격된 노드에서 처리하도록 한다. 그러나 마스터-슬레이브 노드가 동시에 고장날 경우에는 레디스 클러스터 서버가 더 이상 동작하지 않게 된다. 

### 레디스 클러스터의 일관성 보장 기법

레디스 클러스터의 일관성 보장은 **가장 강력한 최대 일관성**까지는 보장하지 못한다. 이 말은 특정한 상황에서 레디스 클러스터는 사용자의 데이터를 잃어버릴 수도 있음을 의미한다.

레디스 클러스터가 데이터 분실을 할 수 있는 이유 중 첫번째는 레플리케이션을 비동기로 진행하기 때문이다. 비동기 레플리케이션은 아래와 같은 순서로 진행된다.

- 클라이언트가 B 노드에 write 요청을 한다.
- B 노드는 자기 노드에만 기록하고 곧바로 클라이언트에 OK 신호를 보낸다.
- 이후에 비동기적으로 B노드의 하위 슬레이브 노드에 해당 write를 replication 한다.

위의 순서에서 볼 수 있듯이 마스터 노드에서 슬레이브 노드들의 레플리케이션 응답을 받지 않고 지나간다. 이는 각 레플리케이션에 대해 응답을 기다릴 경우, 매우 높은 비용의 응답속도 지연이 발생하기 때문이다. 이러한 비동기식 레플리케이션을 하는 도중 마스터 노드가 복구 불능 상태로 빠지고, 아직 레플리케이션 데이터를 전달 받지 못한 슬레이브 노드가 마스터 노드로 승격될 경우, 사용자의 데이터를 잃어버리는 상황이 발생한다.

이러한 상황은 기존의 데이터베이스 시스템에서 사용자 데이터를 주기적으로 flush 하도록 설정된 환경과 동일하다. 기존 데이터베이스 시스템에서는 사용자가 데이터 쓰기를 요청할 때마다 강제로 디스크 까지 flush 하도록 설정하여 강력한 일관성을 보장하게 할 수 있다. 그러나 이러한 설정을 사용할 경우에는 매우 낮은 데이터베이스 성능을 보임이 명확하게 드러나 있다. 이와 유사하게 레디스 클러스터에서도 동기적 레플리케이션을 할 경우, 느린 성능이 나올 수 있다. 

위와 같이 기본적으로 성능과 일관성 보장기법 간에는 trade-off가 있다. 

레디스 클러스터를 사용하는 환경에서 강력한 일관성 기법을 사용하여 동기적 write를 진행하고 싶은 경우에는 [WAIT](https://redis.io/commands/wait) 커맨드를 사용할 수 있다. 이를 통해 기본 커맨드를 사용하는 것보다 강력한 일관성 보장기법을 기대할 수는 있지만, 레플리케이션이 항상 모든 슬레이브에 저장되는게 아니므로, 특정 데이터에 대한 레플리케이션 정보가 전혀 없는 슬레이브 노드가 마스터 노드로 격상 될 경우에는 여전히 데이터 유실이 발생할 수 있다. 

레디스 클러스터가 데이터를 유실하는 경우가 한가지 더 존재한다. 클라이언트 노드가 속한 네트워크가, 네트워크 파티셔닝을 통해 소수의 마스터 노드만을 갖는 네트워크 파티션으로 분리되는 경우이다. 

예제를 통해 알아보자. 각 알파벳은 노드번호를 의미하고 알파벳 뒤에 숫자 번호가 붙은 경우는 슬레이브 노드를 의미한다. 

A, B, C, A1, B1, C1으로 구성된 3마스터 3슬레이브 구조의 레디스 클러스터 노드 6개가 있고, 클라이언트 Z1이 있다고 가정하자. 네트워크 상의 노드 파티션을 진행한 뒤에 A, C, A1, B1, C1 노드가 한 네트워크 파티션 NA1에 존재하고 B, Z1 이 다른 네트워크 NA2에 존재하게 되었을 때, 아직 네트워크 파티션 이후 두 네트워크 간의 연결이 이루어지 않는 순간이 존재하게 된다. 이 떄에 Z1은 여전히 같은 네트워크 상에 있는 B 마스터 노드에 데이터 write를 요청할 수 있다. 네트워크 파티셔닝 이후 각 네트워크 그룹간에 연결이 매우 빠른 시기에 성공할 경우 전체 클러스터가 정상적으로 동작하게 되겠지만, 파티션 분리 이후 오랜 시간이 지나 B 마스터 노드가 NA1에서 유실 됐음으로 판단 되어 B1 노드를 마스터 노드로 승격시킬 경우 문제가 된다. 이 경우 중간에 NA2의 B로 write 했던 데이터들은 유실되게 된다. 

레디스 클러스터에서는 이러한 상황을 명시적으로 사용자에게 알려주고 방지하기 위해 Z1이 NA2의 B로 보낼 수 있는 write 양을 조절하는 **`maximum window`** 를 관리한다. 네트워크 분리 이후 지정된 시간만큼의 시간이 지난 경우 더 이상 NA2로는 어떤 write 도 하지 못하도록 방지한다.

레디스 클러스터를 사용할 때 이 maximum window 시간을 적절히 정하는 것이 매우 중요하고, 실제 설정파일에서 **`node timeout`** 으로 확인할 수 있다. node timeout이 모두 지난뒤에는, 같은 파티션에 존재하지 않는 마스터 노드는 모두 유실된것으로 판별하고 레플리케이션을 가지고 있는 슬레이브 노드들을 마스터로 격상 시킨다. 분리된 마스터 노드 또한 주변 마스터 노드들에 대한 감지가 node timeout 동안 실패할 경우 자체적으로 중단하고 들어오는 write 요청을 거부한다. 



Redis Cluster configuration parameters
===

We are about to create an example cluster deployment. Before we continue,
let's introduce the configuration parameters that Redis Cluster introduces
in the `redis.conf` file. Some will be obvious, others will be more clear
as you continue reading.

* **cluster-enabled `<yes/no>`**: If yes enables Redis Cluster support in a specific Redis instance. Otherwise the instance starts as a stand alone instance as usually.
* **cluster-config-file `<filename>`**: Note that despite the name of this option, this is not an user editable configuration file, but the file where a Redis Cluster node automatically persists the cluster configuration (the state, basically) every time there is a change, in order to be able to re-read it at startup. The file lists things like the other nodes in the cluster, their state, persistent variables, and so forth. Often this file is rewritten and flushed on disk as a result of some message reception.
* **cluster-node-timeout `<milliseconds>`**: The maximum amount of time a Redis Cluster node can be unavailable, without it being considered as failing. If a master node is not reachable for more than the specified amount of time, it will be failed over by its slaves. This parameter controls other important things in Redis Cluster. Notably, every node that can't reach the majority of master nodes for the specified amount of time, will stop accepting queries.
* **cluster-slave-validity-factor `<factor>`**: If set to zero, a slave will always try to failover a master, regardless of the amount of time the link between the master and the slave remained disconnected. If the value is positive, a maximum disconnection time is calculated as the *node timeout* value multiplied by the factor provided with this option, and if the node is a slave, it will not try to start a failover if the master link was disconnected for more than the specified amount of time. For example if the node timeout is set to 5 seconds, and the validity factor is set to 10, a slave disconnected from the master for more than 50 seconds will not try to failover its master. Note that any value different than zero may result in Redis Cluster to be unavailable after a master failure if there is no slave able to failover it. In that case the cluster will return back available only when the original master rejoins the cluster.
* **cluster-migration-barrier `<count>`**: Minimum number of slaves a master will remain connected with, for another slave to migrate to a master which is no longer covered by any slave. See the appropriate section about replica migration in this tutorial for more information.
* **cluster-require-full-coverage `<yes/no>`**: If this is set to yes, as it is by default, the cluster stops accepting writes if some percentage of the key space is not covered by any node. If the option is set to no, the cluster will still serve queries even if only requests about a subset of keys can be processed.

Creating and using a Redis Cluster
===

Note: to deploy a Redis Cluster manually it is **very important to learn** certain
operational aspects of it. However if you want to get a cluster up and running
ASAP (As Soon As Possible) skip this section and the next one and go directly to **Creating a Redis Cluster using the create-cluster script**.

To create a cluster, the first thing we need is to have a few empty
Redis instances running in **cluster mode**. This basically means that
clusters are not created using normal Redis instances as a special mode
needs to be configured so that the Redis instance will enable the Cluster
specific features and commands.

The following is a minimal Redis cluster configuration file:

```
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

As you can see what enables the cluster mode is simply the `cluster-enabled`
directive. Every instance also contains the path of a file where the
configuration for this node is stored, which by default is `nodes.conf`.
This file is never touched by humans; it is simply generated at startup
by the Redis Cluster instances, and updated every time it is needed.

Note that the **minimal cluster** that works as expected requires to contain
at least three master nodes. For your first tests it is strongly suggested
to start a six nodes cluster with three masters and three slaves.

To do so, enter a new directory, and create the following directories named
after the port number of the instance we'll run inside any given directory.

Something like:

```
mkdir cluster-test
cd cluster-test
mkdir 7000 7001 7002 7003 7004 7005
```

Create a `redis.conf` file inside each of the directories, from 7000 to 7005.
As a template for your configuration file just use the small example above,
but make sure to replace the port number `7000` with the right port number
according to the directory name.

Now copy your redis-server executable, **compiled from the latest sources in the unstable branch at GitHub**, into the `cluster-test` directory, and finally open 6 terminal tabs in your favorite terminal application.

Start every instance like that, one every tab:

```
cd 7000
../redis-server ./redis.conf
```

As you can see from the logs of every instance, since no `nodes.conf` file
existed, every node assigns itself a new ID.

    [82462] 26 Nov 11:56:55.329 * No cluster configuration found, I'm 97a3a64667477371c4479320d683e4c8db5858b1

This ID will be used forever by this specific instance in order for the instance
to have a unique name in the context of the cluster. Every node
remembers every other node using this IDs, and not by IP or port.
IP addresses and ports may change, but the unique node identifier will never
change for all the life of the node. We call this identifier simply **Node ID**.

Creating the cluster
---

Now that we have a number of instances running, we need to create our
cluster by writing some meaningful configuration to the nodes.

This is very easy to accomplish as we are helped by the Redis Cluster
command line utility called `redis-trib`, a Ruby program
executing special commands on instances in order to create new clusters,
check or reshard an existing cluster, and so forth.


The `redis-trib` utility is in the `src` directory of the Redis source code
distribution.
You need to install `redis` gem to be able to run `redis-trib`.

    gem install redis

 To create your cluster simply type:

    ./redis-trib.rb create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 \
    127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005

The command used here is **create**, since we want to create a new cluster.
The option `--replicas 1` means that we want a slave for every master created.
The other arguments are the list of addresses of the instances I want to use
to create the new cluster.

Obviously the only setup with our requirements is to create a cluster with
3 masters and 3 slaves.

Redis-trib will propose you a configuration. Accept the proposed configuration by typing **yes**.
The cluster will be configured and *joined*, which means, instances will be
bootstrapped into talking with each other. Finally, if everything went well,
you'll see a message like that:

    [OK] All 16384 slots covered

This means that there is at least a master instance serving each of the
16384 slots available.

Creating a Redis Cluster using the create-cluster script
---

If you don't want to create a Redis Cluster by configuring and executing
individual instances manually as explained above, there is a much simpler
system (but you'll not learn the same amount of operational details).

Just check `utils/create-cluster` directory in the Redis distribution.
There is a script called `create-cluster` inside (same name as the directory
it is contained into), it's a simple bash script. In order to start
a 6 nodes cluster with 3 masters and 3 slaves just type the following
commands:

1. `create-cluster start`
2. `create-cluster create`

Reply to `yes` in step 2 when the `redis-trib` utility wants you to accept
the cluster layout.

You can now interact with the cluster, the first node will start at port 30001
by default. When you are done, stop the cluster with:

3. `create-cluster stop`.

Please read the `README` inside this directory for more information on how
to run the script.

Playing with the cluster
---

At this stage one of the problems with Redis Cluster is the lack of
client libraries implementations.

I'm aware of the following implementations:

* [redis-rb-cluster](http://github.com/antirez/redis-rb-cluster) is a Ruby implementation written by me (@antirez) as a reference for other languages. It is a simple wrapper around the original redis-rb, implementing the minimal semantics to talk with the cluster efficiently.
* [redis-py-cluster](https://github.com/Grokzen/redis-py-cluster) A port of redis-rb-cluster to Python. Supports majority of *redis-py* functionality. Is in active development.
* The popular [Predis](https://github.com/nrk/predis) has support for Redis Cluster, the support was recently updated and is in active development.
* The most used Java client, [Jedis](https://github.com/xetorthio/jedis) recently added support for Redis Cluster, see the *Jedis Cluster* section in the project README.
* [StackExchange.Redis](https://github.com/StackExchange/StackExchange.Redis) offers support for C# (and should work fine with most .NET languages; VB, F#, etc)
* [thunk-redis](https://github.com/thunks/thunk-redis) offers support for Node.js and io.js, it is a thunk/promise-based redis client with pipelining and cluster.
* [redis-go-cluster](https://github.com/chasex/redis-go-cluster) is an implementation of Redis Cluster for the Go language using the [Redigo library client](https://github.com/garyburd/redigo) as the base client. Implements MGET/MSET via result aggregation.
* The `redis-cli` utility in the unstable branch of the Redis repository at GitHub implements a very basic cluster support when started with the `-c` switch.

An easy way to test Redis Cluster is either to try any of the above clients
or simply the `redis-cli` command line utility. The following is an example
of interaction using the latter:

```
$ redis-cli -c -p 7000
redis 127.0.0.1:7000> set foo bar
-> Redirected to slot [12182] located at 127.0.0.1:7002
OK
redis 127.0.0.1:7002> set hello world
-> Redirected to slot [866] located at 127.0.0.1:7000
OK
redis 127.0.0.1:7000> get foo
-> Redirected to slot [12182] located at 127.0.0.1:7002
"bar"
redis 127.0.0.1:7000> get hello
-> Redirected to slot [866] located at 127.0.0.1:7000
"world"
```

**Note:** if you created the cluster using the script your nodes may listen
to different ports, starting from 30001 by default.

The redis-cli cluster support is very basic so it always uses the fact that
Redis Cluster nodes are able to redirect a client to the right node.
A serious client is able to do better than that, and cache the map between
hash slots and nodes addresses, to directly use the right connection to the
right node. The map is refreshed only when something changed in the cluster
configuration, for example after a failover or after the system administrator
changed the cluster layout by adding or removing nodes.

Writing an example app with redis-rb-cluster
---

Before going forward showing how to operate the Redis Cluster, doing things
like a failover, or a resharding, we need to create some example application
or at least to be able to understand the semantics of a simple Redis Cluster
client interaction.

In this way we can run an example and at the same time try to make nodes
failing, or start a resharding, to see how Redis Cluster behaves under real
world conditions. It is not very helpful to see what happens while nobody
is writing to the cluster.

This section explains some basic usage of
[redis-rb-cluster](https://github.com/antirez/redis-rb-cluster) showing two
examples. The first is the following, and is the
[`example.rb`](https://github.com/antirez/redis-rb-cluster/blob/master/example.rb)
file inside the redis-rb-cluster distribution:

```
   1  require './cluster'
   2
   3  if ARGV.length != 2
   4      startup_nodes = [
   5          {:host => "127.0.0.1", :port => 7000},
   6          {:host => "127.0.0.1", :port => 7001}
   7      ]
   8  else
   9      startup_nodes = [
  10          {:host => ARGV[0], :port => ARGV[1].to_i}
  11      ]
  12  end
  13
  14  rc = RedisCluster.new(startup_nodes,32,:timeout => 0.1)
  15
  16  last = false
  17
  18  while not last
  19      begin
  20          last = rc.get("__last__")
  21          last = 0 if !last
  22      rescue => e
  23          puts "error #{e.to_s}"
  24          sleep 1
  25      end
  26  end
  27
  28  ((last.to_i+1)..1000000000).each{|x|
  29      begin
  30          rc.set("foo#{x}",x)
  31          puts rc.get("foo#{x}")
  32          rc.set("__last__",x)
  33      rescue => e
  34          puts "error #{e.to_s}"
  35      end
  36      sleep 0.1
  37  }
```

The application does a very simple thing, it sets keys in the form `foo<number>` to `number`, one after the other. So if you run the program the result is the
following stream of commands:

* SET foo0 0
* SET foo1 1
* SET foo2 2
* And so forth...

The program looks more complex than it should usually as it is designed to
show errors on the screen instead of exiting with an exception, so every
operation performed with the cluster is wrapped by `begin` `rescue` blocks.

The **line 14** is the first interesting line in the program. It creates the
Redis Cluster object, using as argument a list of *startup nodes*, the maximum
number of connections this object is allowed to take against different nodes,
and finally the timeout after a given operation is considered to be failed.

The startup nodes don't need to be all the nodes of the cluster. The important
thing is that at least one node is reachable. Also note that redis-rb-cluster
updates this list of startup nodes as soon as it is able to connect with the
first node. You should expect such a behavior with any other serious client.

Now that we have the Redis Cluster object instance stored in the **rc** variable
we are ready to use the object like if it was a normal Redis object instance.

This is exactly what happens in **line 18 to 26**: when we restart the example
we don't want to start again with `foo0`, so we store the counter inside
Redis itself. The code above is designed to read this counter, or if the
counter does not exist, to assign it the value of zero.

However note how it is a while loop, as we want to try again and again even
if the cluster is down and is returning errors. Normal applications don't need
to be so careful.

**Lines between 28 and 37** start the main loop where the keys are set or
an error is displayed.

Note the `sleep` call at the end of the loop. In your tests you can remove
the sleep if you want to write to the cluster as fast as possible (relatively
to the fact that this is a busy loop without real parallelism of course, so
you'll get the usually 10k ops/second in the best of the conditions).

Normally writes are slowed down in order for the example application to be
easier to follow by humans.

Starting the application produces the following output:

```
ruby ./example.rb
1
2
3
4
5
6
7
8
9
^C (I stopped the program here)
```

This is not a very interesting program and we'll use a better one in a moment
but we can already see what happens during a resharding when the program
is running.

Resharding the cluster
---

Now we are ready to try a cluster resharding. To do this please
keep the example.rb program running, so that you can see if there is some
impact on the program running. Also you may want to comment the `sleep`
call in order to have some more serious write load during resharding.

Resharding basically means to move hash slots from a set of nodes to another
set of nodes, and like cluster creation it is accomplished using the
redis-trib utility.

To start a resharding just type:

    ./redis-trib.rb reshard 127.0.0.1:7000

You only need to specify a single node, redis-trib will find the other nodes
automatically.

Currently redis-trib is only able to reshard with the administrator support,
you can't just say move 5% of slots from this node to the other one (but
this is pretty trivial to implement). So it starts with questions. The first
is how much a big resharding do you want to do:

    How many slots do you want to move (from 1 to 16384)?

We can try to reshard 1000 hash slots, that should already contain a non
trivial amount of keys if the example is still running without the sleep
call.

Then redis-trib needs to know what is the target of the resharding, that is,
the node that will receive the hash slots.
I'll use the first master node, that is, 127.0.0.1:7000, but I need
to specify the Node ID of the instance. This was already printed in a
list by redis-trib, but I can always find the ID of a node with the following
command if I need:

```
$ redis-cli -p 7000 cluster nodes | grep myself
97a3a64667477371c4479320d683e4c8db5858b1 :0 myself,master - 0 0 0 connected 0-5460
```

Ok so my target node is 97a3a64667477371c4479320d683e4c8db5858b1.

Now you'll get asked from what nodes you want to take those keys.
I'll just type `all` in order to take a bit of hash slots from all the
other master nodes.

After the final confirmation you'll see a message for every slot that
redis-trib is going to move from a node to another, and a dot will be printed
for every actual key moved from one side to the other.

While the resharding is in progress you should be able to see your
example program running unaffected. You can stop and restart it multiple times
during the resharding if you want.

At the end of the resharding, you can test the health of the cluster with
the following command:

    ./redis-trib.rb check 127.0.0.1:7000

All the slots will be covered as usually, but this time the master at
127.0.0.1:7000 will have more hash slots, something around 6461.

Scripting a resharding operation
---

Reshardings can be performed automatically without the need to manually
enter the parameters in an interactive way. This is possible using a command
line like the following:

    ./redis-trib.rb reshard --from <node-id> --to <node-id> --slots <number of slots> --yes <host>:<port>

This allows to build some automatism if you are likely to reshard often,
however currently there is no way for `redis-trib` to automatically
rebalance the cluster checking the distribution of keys across the cluster
nodes and intelligently moving slots as needed. This feature will be added
in the future.

A more interesting example application
---

The example application we wrote early is not very good.
It writes to the cluster in a simple way without even checking if what was
written is the right thing.

From our point of view the cluster receiving the writes could just always
write the key `foo` to `42` to every operation, and we would not notice at
all.

So in the `redis-rb-cluster` repository, there is a more interesting application
that is called `consistency-test.rb`. It uses a set of counters, by default 1000, and sends `INCR` commands in order to increment the counters.

However instead of just writing, the application does two additional things:

* When a counter is updated using `INCR`, the application remembers the write.
* It also reads a random counter before every write, and check if the value is what we expected it to be, comparing it with the value it has in memory.

What this means is that this application is a simple **consistency checker**,
and is able to tell you if the cluster lost some write, or if it accepted
a write that we did not receive acknowledgment for. In the first case we'll
see a counter having a value that is smaller than the one we remember, while
in the second case the value will be greater.

Running the consistency-test application produces a line of output every
second:

```
$ ruby consistency-test.rb
925 R (0 err) | 925 W (0 err) |
5030 R (0 err) | 5030 W (0 err) |
9261 R (0 err) | 9261 W (0 err) |
13517 R (0 err) | 13517 W (0 err) |
17780 R (0 err) | 17780 W (0 err) |
22025 R (0 err) | 22025 W (0 err) |
25818 R (0 err) | 25818 W (0 err) |
```

The line shows the number of **R**eads and **W**rites performed, and the
number of errors (query not accepted because of errors since the system was
not available).

If some inconsistency is found, new lines are added to the output.
This is what happens, for example, if I reset a counter manually while
the program is running:

```
$ redis-cli -h 127.0.0.1 -p 7000 set key_217 0
OK

(in the other tab I see...)

94774 R (0 err) | 94774 W (0 err) |
98821 R (0 err) | 98821 W (0 err) |
102886 R (0 err) | 102886 W (0 err) | 114 lost |
107046 R (0 err) | 107046 W (0 err) | 114 lost |
```

When I set the counter to 0 the real value was 114, so the program reports
114 lost writes (`INCR` commands that are not remembered by the cluster).

This program is much more interesting as a test case, so we'll use it
to test the Redis Cluster failover.

Testing the failover
---

Note: during this test, you should take a tab open with the consistency test
application running.

In order to trigger the failover, the simplest thing we can do (that is also
the semantically simplest failure that can occur in a distributed system)
is to crash a single process, in our case a single master.

We can identify a cluster and crash it with the following command:

```
$ redis-cli -p 7000 cluster nodes | grep master
3e3a6cb0d9a9a87168e266b0a0b24026c0aae3f0 127.0.0.1:7001 master - 0 1385482984082 0 connected 5960-10921
2938205e12de373867bf38f1ca29d31d0ddb3e46 127.0.0.1:7002 master - 0 1385482983582 0 connected 11423-16383
97a3a64667477371c4479320d683e4c8db5858b1 :0 myself,master - 0 0 0 connected 0-5959 10922-11422
```

Ok, so 7000, 7001, and 7002 are masters. Let's crash node 7002 with the
**DEBUG SEGFAULT** command:

```
$ redis-cli -p 7002 debug segfault
Error: Server closed the connection
```

Now we can look at the output of the consistency test to see what it reported.

```
18849 R (0 err) | 18849 W (0 err) |
23151 R (0 err) | 23151 W (0 err) |
27302 R (0 err) | 27302 W (0 err) |

... many error warnings here ...

29659 R (578 err) | 29660 W (577 err) |
33749 R (578 err) | 33750 W (577 err) |
37918 R (578 err) | 37919 W (577 err) |
42077 R (578 err) | 42078 W (577 err) |
```

As you can see during the failover the system was not able to accept 578 reads and 577 writes, however no inconsistency was created in the database. This may
sound unexpected as in the first part of this tutorial we stated that Redis
Cluster can lose writes during the failover because it uses asynchronous
replication. What we did not say is that this is not very likely to happen
because Redis sends the reply to the client, and the commands to replicate
to the slaves, about at the same time, so there is a very small window to
lose data. However the fact that it is hard to trigger does not mean that it
is impossible, so this does not change the consistency guarantees provided
by Redis cluster.

We can now check what is the cluster setup after the failover (note that
in the meantime I restarted the crashed instance so that it rejoins the
cluster as a slave):

```
$ redis-cli -p 7000 cluster nodes
3fc783611028b1707fd65345e763befb36454d73 127.0.0.1:7004 slave 3e3a6cb0d9a9a87168e266b0a0b24026c0aae3f0 0 1385503418521 0 connected
a211e242fc6b22a9427fed61285e85892fa04e08 127.0.0.1:7003 slave 97a3a64667477371c4479320d683e4c8db5858b1 0 1385503419023 0 connected
97a3a64667477371c4479320d683e4c8db5858b1 :0 myself,master - 0 0 0 connected 0-5959 10922-11422
3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 127.0.0.1:7005 master - 0 1385503419023 3 connected 11423-16383
3e3a6cb0d9a9a87168e266b0a0b24026c0aae3f0 127.0.0.1:7001 master - 0 1385503417005 0 connected 5960-10921
2938205e12de373867bf38f1ca29d31d0ddb3e46 127.0.0.1:7002 slave 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 0 1385503418016 3 connected
```

Now the masters are running on ports 7000, 7001 and 7005. What was previously
a master, that is the Redis instance running on port 7002, is now a slave of
7005.

The output of the `CLUSTER NODES` command may look intimidating, but it is actually pretty simple, and is composed of the following tokens:

* Node ID
* ip:port
* flags: master, slave, myself, fail, ...
* if it is a slave, the Node ID of the master
* Time of the last pending PING still waiting for a reply.
* Time of the last PONG received.
* Configuration epoch for this node (see the Cluster specification).
* Status of the link to this node.
* Slots served...

Manual failover
---

Sometimes it is useful to force a failover without actually causing any problem
on a master. For example in order to upgrade the Redis process of one of the
master nodes it is a good idea to failover it in order to turn it into a slave
with minimal impact on availability.

Manual failovers are supported by Redis Cluster using the `CLUSTER FAILOVER`
command, that must be executed in one of the **slaves** of the master you want
to failover.

Manual failovers are special and are safer compared to failovers resulting from
actual master failures, since they occur in a way that avoid data loss in the
process, by switching clients from the original master to the new master only
when the system is sure that the new master processed all the replication stream
from the old one.

This is what you see in the slave log when you perform a manual failover:

    # Manual failover user request accepted.
    # Received replication offset for paused master manual failover: 347540
    # All master replication stream processed, manual failover can start.
    # Start of election delayed for 0 milliseconds (rank #0, offset 347540).
    # Starting a failover election for epoch 7545.
    # Failover election won: I'm the new master.

Basically clients connected to the master we are failing over are stopped.
At the same time the master sends its replication offset to the slave, that
waits to reach the offset on its side. When the replication offset is reached,
the failover starts, and the old master is informed about the configuration
switch. When the clients are unblocked on the old master, they are redirected
to the new master.

Adding a new node
---

Adding a new node is basically the process of adding an empty node and then
moving some data into it, in case it is a new master, or telling it to
setup as a replica of a known node, in case it is a slave.

We'll show both, starting with the addition of a new master instance.

In both cases the first step to perform is **adding an empty node**.

This is as simple as to start a new node in port 7006 (we already used
from 7000 to 7005 for our existing 6 nodes) with the same configuration
used for the other nodes, except for the port number, so what you should
do in order to conform with the setup we used for the previous nodes:

* Create a new tab in your terminal application.
* Enter the `cluster-test` directory.
* Create a directory named `7006`.
* Create a redis.conf file inside, similar to the one used for the other nodes but using 7006 as port number.
* Finally start the server with `../redis-server ./redis.conf`

At this point the server should be running.

Now we can use **redis-trib** as usually in order to add the node to
the existing cluster.

    ./redis-trib.rb add-node 127.0.0.1:7006 127.0.0.1:7000

As you can see I used the **add-node** command specifying the address of the
new node as first argument, and the address of a random existing node in the
cluster as second argument.

In practical terms redis-trib here did very little to help us, it just
sent a `CLUSTER MEET` message to the node, something that is also possible
to accomplish manually. However redis-trib also checks the state of the
cluster before to operate, so it is a good idea to perform cluster operations
always via redis-trib even when you know how the internals work.

Now we can connect to the new node to see if it really joined the cluster:

```
redis 127.0.0.1:7006> cluster nodes
3e3a6cb0d9a9a87168e266b0a0b24026c0aae3f0 127.0.0.1:7001 master - 0 1385543178575 0 connected 5960-10921
3fc783611028b1707fd65345e763befb36454d73 127.0.0.1:7004 slave 3e3a6cb0d9a9a87168e266b0a0b24026c0aae3f0 0 1385543179583 0 connected
f093c80dde814da99c5cf72a7dd01590792b783b :0 myself,master - 0 0 0 connected
2938205e12de373867bf38f1ca29d31d0ddb3e46 127.0.0.1:7002 slave 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 0 1385543178072 3 connected
a211e242fc6b22a9427fed61285e85892fa04e08 127.0.0.1:7003 slave 97a3a64667477371c4479320d683e4c8db5858b1 0 1385543178575 0 connected
97a3a64667477371c4479320d683e4c8db5858b1 127.0.0.1:7000 master - 0 1385543179080 0 connected 0-5959 10922-11422
3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 127.0.0.1:7005 master - 0 1385543177568 3 connected 11423-16383
```

Note that since this node is already connected to the cluster it is already
able to redirect client queries correctly and is generally speaking part of
the cluster. However it has two peculiarities compared to the other masters:

* It holds no data as it has no assigned hash slots.
* Because it is a master without assigned slots, it does not participate in the election process when a slave wants to become a master.

Now it is possible to assign hash slots to this node using the resharding
feature of `redis-trib`. It is basically useless to show this as we already
did in a previous section, there is no difference, it is just a resharding
having as a target the empty node.

Adding a new node as a replica
---

Adding a new Replica can be performed in two ways. The obvious one is to
use redis-trib again, but with the --slave option, like this:

    ./redis-trib.rb add-node --slave 127.0.0.1:7006 127.0.0.1:7000

Note that the command line here is exactly like the one we used to add
a new master, so we are not specifying to which master we want to add
the replica. In this case what happens is that redis-trib will add the new
node as replica of a random master among the masters with less replicas.

However you can specify exactly what master you want to target with your
new replica with the following command line:

    ./redis-trib.rb add-node --slave --master-id 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 127.0.0.1:7006 127.0.0.1:7000

This way we assign the new replica to a specific master.

A more manual way to add a replica to a specific master is to add the new
node as an empty master, and then turn it into a replica using the
`CLUSTER REPLICATE` command. This also works if the node was added as a slave
but you want to move it as a replica of a different master.

For example in order to add a replica for the node 127.0.0.1:7005 that is
currently serving hash slots in the range 11423-16383, that has a Node ID
3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e, all I need to do is to connect
with the new node (already added as empty master) and send the command:

    redis 127.0.0.1:7006> cluster replicate 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e

That's it. Now we have a new replica for this set of hash slots, and all
the other nodes in the cluster already know (after a few seconds needed to
update their config). We can verify with the following command:

```
$ redis-cli -p 7000 cluster nodes | grep slave | grep 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e
f093c80dde814da99c5cf72a7dd01590792b783b 127.0.0.1:7006 slave 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 0 1385543617702 3 connected
2938205e12de373867bf38f1ca29d31d0ddb3e46 127.0.0.1:7002 slave 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 0 1385543617198 3 connected
```

The node 3c3a0c... now has two slaves, running on ports 7002 (the existing one) and 7006 (the new one).

Removing a node
---

To remove a slave node just use the `del-node` command of redis-trib:

    ./redis-trib del-node 127.0.0.1:7000 `<node-id>`

The first argument is just a random node in the cluster, the second argument
is the ID of the node you want to remove.

You can remove a master node in the same way as well, **however in order to
remove a master node it must be empty**. If the master is not empty you need
to reshard data away from it to all the other master nodes before.

An alternative to remove a master node is to perform a manual failover of it
over one of its slaves and remove the node after it turned into a slave of the
new master. Obviously this does not help when you want to reduce the actual
number of masters in your cluster, in that case, a resharding is needed.

Replicas migration
---

In Redis Cluster it is possible to reconfigure a slave to replicate with a
different master at any time just using the following command:

    CLUSTER REPLICATE <master-node-id>

However there is a special scenario where you want replicas to move from one
master to another one automatically, without the help of the system administrator.
The automatic reconfiguration of replicas is called *replicas migration* and is
able to improve the reliability of a Redis Cluster.

Note: you can read the details of replicas migration in the [Redis Cluster Specification](/topics/cluster-spec), here we'll only provide some information about the
general idea and what you should do in order to benefit from it.

The reason why you may want to let your cluster replicas to move from one master
to another under certain condition, is that usually the Redis Cluster is as
resistant to failures as the number of replicas attached to a given master.

For example a cluster where every master has a single replica can't continue
operations if the master and its replica fail at the same time, simply because
there is no other instance to have a copy of the hash slots the master was
serving. However while netsplits are likely to isolate a number of nodes
at the same time, many other kind of failures, like hardware or software failures
local to a single node, are a very notable class of failures that are unlikely
to happen at the same time, so it is possible that in your cluster where
every master has a slave, the slave is killed at 4am, and the master is killed
at 6am. This still will result in a cluster that can no longer operate.

To improve reliability of the system we have the option to add additional
replicas to every master, but this is expensive. Replica migration allows to
add more slaves to just a few masters. So you have 10 masters with 1 slave
each, for a total of 20 instances. However you add, for example, 3 instances
more as slaves of some of your masters, so certain masters will have more
than a single slave.

With replicas migration what happens is that if a master is left without
slaves, a replica from a master that has multiple slaves will migrate to
the *orphaned* master. So after your slave goes down at 4am as in the example
we made above, another slave will take its place, and when the master
will fail as well at 5am, there is still a slave that can be elected so that
the cluster can continue to operate.

So what you should know about replicas migration in short?

* The cluster will try to migrate a replica from the master that has the greatest number of replicas in a given moment.
* To benefit from replica migration you have just to add a few more replicas to a single master in your cluster, it does not matter what master.
* There is a configuration parameter that controls the replica migration feature that is called `cluster-migration-barrier`: you can read more about it in the example `redis.conf` file provided with Redis Cluster.

Upgrading nodes in a Redis Cluster
---

Upgrading slave nodes is easy since you just need to stop the node and restart
it with an updated version of Redis. If there are clients scaling reads using
slave nodes, they should be able to reconnect to a different slave if a given
one is not available.

Upgrading masters is a bit more complex, and the suggested procedure is:

1. Use CLUSTER FAILOVER to trigger a manual failover of the master to one of its slaves (see the "Manual failover" section of this documentation).
2. Wait for the master to turn into a slave.
3. Finally upgrade the node as you do for slaves.
4. If you want the master to be the node you just upgraded, trigger a new manual failover in order to turn back the upgraded node into a master.

Following this procedure you should upgrade one node after the other until
all the nodes are upgraded.

Migrating to Redis Cluster
---

Users willing to migrate to Redis Cluster may have just a single master, or
may already using a preexisting sharding setup, where keys
are split among N nodes, using some in-house algorithm or a sharding algorithm
implemented by their client library or Redis proxy.

In both cases it is possible to migrate to Redis Cluster easily, however
what is the most important detail is if multiple-keys operations are used
by the application, and how. There are three different cases:

1. Multiple keys operations, or transactions, or Lua scripts involving multiple keys, are not used. Keys are accessed independently (even if accessed via transactions or Lua scripts grouping multiple commands, about the same key, together).
2. Multiple keys operations, transactions, or Lua scripts involving multiple keys are used but only with keys having the same **hash tag**, which means that the keys used together all have a `{...}` sub-string that happens to be identical. For example the following multiple keys operation is defined in the context of the same hash tag: `SUNION {user:1000}.foo {user:1000}.bar`.
3. Multiple keys operations, transactions, or Lua scripts involving multiple keys are used with key names not having an explicit, or the same, hash tag.

The third case is not handled by Redis Cluster: the application requires to
be modified in order to don't use multi keys operations or only use them in
the context of the same hash tag.

Case 1 and 2 are covered, so we'll focus on those two cases, that are handled
in the same way, so no distinction will be made in the documentation.

Assuming you have your preexisting data set split into N masters, where
N=1 if you have no preexisting sharding, the following steps are needed
in order to migrate your data set to Redis Cluster:

1. Stop your clients. No automatic live-migration to Redis Cluster is currently possible. You may be able to do it orchestrating a live migration in the context of your application / environment.
2. Generate an append only file for all of your N masters using the BGREWRITEAOF command, and waiting for the AOF file to be completely generated.
3. Save your AOF files from aof-1 to aof-N somewhere. At this point you can stop your old instances if you wish (this is useful since in non-virtualized deployments you often need to reuse the same computers).
4. Create a Redis Cluster composed of N masters and zero slaves. You'll add slaves later. Make sure all your nodes are using the append only file for persistence.
5. Stop all the cluster nodes, substitute their append only file with your pre-existing append only files, aof-1 for the first node, aof-2 for the second node, up to aof-N.
6. Restart your Redis Cluster nodes with the new AOF files. They'll complain that there are keys that should not be there according to their configuration.
7. Use `redis-trib fix` command in order to fix the cluster so that keys will be migrated according to the hash slots each node is authoritative or not.
8. Use `redis-trib check` at the end to make sure your cluster is ok.
9. Restart your clients modified to use a Redis Cluster aware client library.

There is an alternative way to import data from external instances to a Redis
Cluster, which is to use the `redis-trib import` command.

The command moves all the keys of a running instance (deleting the keys from
the source instance) to the specified pre-existing Redis Cluster. However
note that if you use a Redis 2.8 instance as source instance the operation
may be slow since 2.8 does not implement migrate connection caching, so you
may want to restart your source instance with a Redis 3.x version before
to perform such operation.

