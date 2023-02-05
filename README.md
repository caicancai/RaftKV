# Raft-kv

基于Raft和boltdb实现的一个分布式kv存储系统

## 设计接入协议

 使用 HTTP 协议，设计 HTTP RESTful API，作为访问接口 

```shell
curl -XGET http://raft-cluster-host01:8091/key/foo
```

实现路由访问，实现了多个 API，比如"/key"和"/join"，需要将 API 对应的请求和它对应的处理函数一一映射起来。

```go
func (s *Service) ServeHTTP(w http.ResponseWriter, r *http.Request) {    // 设置HTTP请求对应的路由信息
        if strings.HasPrefix(r.URL.Path, "/key") {
                s.handleKeyRequest(w, r)
        } else if r.URL.Path == "/join" {
                s.handleJoin(w, r)
        } else {
                w.WriteHeader(http.StatusNotFound)
        }
}
```

## 设计KV操作

### 前端交互接口

 赋值操作：我们可以通过 HTTP POST 请求，来对指定 key 进行赋值，就像下面的样子。

```shell
curl -XPOST http://raft-cluster-host01:8091/key -d '{"foo": "bar"}'
```

查询操作：我们可以通过 HTTP GET 请求，来查询指定 key 的值，就像下面的样子。

```shell
curl -XGET http://raft-cluster-host01:8091/key/foo
```

删除操作：我们可以通过 HTTP DELETE 请求，来删除指定 key 和 key 对应的值，就像下面的样子。

```shell
curl -XDELETE http://raft-cluster-host01:8091/key/foo 
```

### 后端接口

#### get

 我们需要根据实际场景特点进行权衡折中，这样，才能设计出最适合该场景特点的读操作。比如，我们可以实现类似 Consul 的 3 种读一致性模型。 

- default：偶尔读到旧数据。
- consistent：一定不会读到旧数据。
- stale：会读到旧数据

```go
// Represents the available consistency levels.
const (
	Default ConsistencyLevel = iota
	Stale
	Consistent
)

func (s *Store) Get(key string, lvl ConsistencyLevel) (string, error) {
	if lvl != Stale {
		if s.raft.State() != raft.Leader {
			return "", raft.ErrNotLeader
		}

	}

	if lvl == Consistent {
		if err := s.consistentRead(); err != nil {
			return "", err
		}
	}

	s.mu.Lock()
	defer s.mu.Unlock()
	return s.m[key], nil
}
```

#### set

一般而言，有 2 种方法来实现写操作。

方法 1：跟随者接收到客户端的写请求后，拒绝处理这个请求，并将领导者的地址信息返回给客户端，然后客户端直接访问领导者节点，直到该领导者退位，

方法 2：跟随者接收到客户端的写请求后，将写请求转发给领导者，并将领导者处理后的结果返回给客户端，也就是说，这时跟随者在扮演“代理”的角色

我选择方法一，如下：

```go
func (s *Store) Set(key, value string) error {
	if s.raft.State() != raft.Leader {
		return raft.ErrNotLeader
	}

	c := &command{
		Op:    "set",
		Key:   key,
		Value: value,
	}
	b, err := json.Marshal(c)
	if err != nil {
		return err
	}

	f := s.raft.Apply(b, raftTimeout)
	return f.Error()
}
```

delete

```go
func (s *Store) Delete(key string) error {
	if s.raft.State() != raft.Leader {
		return raft.ErrNotLeader
	}

	c := &command{
		Op:  "delete",
		Key: key,
	}
	b, err := json.Marshal(c)
	if err != nil {
		return err
	}

	f := s.raft.Apply(b, raftTimeout)
	return f.Error()
}
```

## 如何实现分布式集群

 在 Raft 算法中，我们可以这样创建集群。

- 先将第一个节点，通过 Bootstrap 的方式启动，并作为领导者节点。
- 其他节点与领导者节点通讯，将自己的配置信息发送给领导者节点，然后领导者节点调用 AddVoter() 函数，将新节点加入到集群中。 
