
# 节点网络

一个节点在启动时，必须连结到一些可用的节点上，才能共同区块数。保持全网络数据一致性。

## 节点发现

在第二章中，我们有看到创建地址管理服务时，其中一个参数就是peers.json。但是，第一次启动节点时，节点是没有任何可用的临近节点的，那这个节点是如何连结到主网的呢。下面就从代码中找找逻辑：

首先，还是回到peerhandler中，看看addrmgr.start()做了什么事。

```
// Start begins the core address handler which manages a pool of known
// addresses, timeouts, and interval based writes.
func (a *AddrManager) Start() {
    // Already started?
    if atomic.AddInt32(&a.started, 1) != 1 {
        return
    }

    log.Trace("Starting address manager")

    // Load peers we already know about from file.
    a.loadPeers()

    // Start the address ticker to save addresses periodically.
    a.wg.Add(1)
    go a.addressHandler()
}
```

### a.loadPeers()

看到了加载节点信息的代码：a.loadPeers()。它是从文件中加载记录的节点地址信息。
```
// loadPeers loads the known address from the saved file.  If empty, missing, or
// malformed file, just don't load anything and start fresh
func (a *AddrManager) loadPeers() {
    a.mtx.Lock()
    defer a.mtx.Unlock()

    err := a.deserializePeers(a.peersFile)
    if err != nil {
        log.Errorf("Failed to parse file %s: %v", a.peersFile, err)
        // if it is invalid we nuke the old one unconditionally.
        err = os.Remove(a.peersFile)
        if err != nil {
            log.Warnf("Failed to remove corrupt peers file %s: %v",
                a.peersFile, err)
        }
        a.reset()
        return
    }
    log.Infof("Loaded %d addresses from file '%s'", a.numAddresses(), a.peersFile)
}
```
读取文件会用到锁。然后调用a.deserializePeers(a.peersFile)读取文件，文件内容是json格式，因此，会反序列化为对象serializedAddrManager。我们来看下相关内容：

**peers.json内容格式**
```
{
    "Version": 1,
    "Key": [
        249,
        ...
        232,
        239
    ],
    "Addresses": [
        {
            "Addr": "80.219.27.212:8333",
            "Src": "13.115.194.245:8333",
            "Attempts": 0,
            "TimeStamp": 1540282028,
            "LastAttempt": -62135596800,
            "LastSuccess": -62135596800
        },
        {
            "Addr": "2.92.255.216:8333",
            "Src": "13.115.194.245:8333",
            "Attempts": 0,
            "TimeStamp": 1540282028,
            "LastAttempt": -62135596800,
            "LastSuccess": -62135596800
        }
    ],
    "NewBuckets": [
        [],
        [],
        [],
        [
            "[2001:0:9d38:6ab8:f8:10c3:fad3:562d]:8333"
        ],
        [
            "18.194.141.225:8333",
            "[2605:6000:101c:6cc:7062:2ca7:9174:731c]:8333",
            "136.56.155.94:8333",
            "49.81.66.104:8333",
            "95.114.56.217:8333",
            "49.81.65.19:8333",
            "[2605:6000:1105:111f:dc86:e4e6:c82e:7556]:8333",
            "77.37.102.6:8333",
            "1.65.182.72:8333",
            "49.81.66.69:8333",
            "18.144.39.12:8333",
            "114.206.60.98:8333"
        ]
        ...省略
    ]
    "TriedBuckets": [
        [],
        [],
        [],
        [],
        [
            "74.14.27.89:8333"
        ],
        [
            "13.115.194.245:8333"
        ]
        ...省略
    ]
}
```
**对应的结构体如下**
```
type serializedAddrManager struct {
    Version      int
    Key          [32]byte
    Addresses    []*serializedKnownAddress
    NewBuckets   [newBucketCount][]string // string is NetAddressKey
    TriedBuckets [triedBucketCount][]string
}

type serializedKnownAddress struct {
    Addr        string
    Src         string
    Attempts    int
    TimeStamp   int64
    LastAttempt int64
    LastSuccess int64
    // no refcount or tried, that is available from context.
}

// KnownAddress tracks information about a known network address that is used
// to determine how viable an address is.
type KnownAddress struct {
    na          *wire.NetAddress
    srcAddr     *wire.NetAddress
    attempts    int
    lastattempt time.Time
    lastsuccess time.Time
    tried       bool
    refs        int // reference count of new buckets
}

```
这里先不管它们的意思。大致知道节点地址配置就行了。后面一步步看。

读取的KnownAddress最终会保存到addrIndex中：

```

func (a *AddrManager) deserializePeers(filePath string) error {

    _, err := os.Stat(filePath)
    if os.IsNotExist(err) {
        return nil
    }
    r, err := os.Open(filePath)
    if err != nil {
        return fmt.Errorf("%s error opening file: %v", filePath, err)
    }
    defer r.Close()

    var sam serializedAddrManager
    dec := json.NewDecoder(r)
    err = dec.Decode(&sam)
    if err != nil {
        return fmt.Errorf("error reading %s: %v", filePath, err)
    }

    if sam.Version != serialisationVersion {
        return fmt.Errorf("unknown version %v in serialized "+
            "addrmanager", sam.Version)
    }
    copy(a.key[:], sam.Key[:])

    for _, v := range sam.Addresses {
        ka := new(KnownAddress)
        ka.na, err = a.DeserializeNetAddress(v.Addr)
        if err != nil {
            return fmt.Errorf("failed to deserialize netaddress "+
                "%s: %v", v.Addr, err)
        }
        ka.srcAddr, err = a.DeserializeNetAddress(v.Src)
        if err != nil {
            return fmt.Errorf("failed to deserialize netaddress "+
                "%s: %v", v.Src, err)
        }
        ka.attempts = v.Attempts
        ka.lastattempt = time.Unix(v.LastAttempt, 0)
        ka.lastsuccess = time.Unix(v.LastSuccess, 0)
        a.addrIndex[NetAddressKey(ka.na)] = ka
    }
    ...
}
```
NetAddressKey返回的格式为：
- IPv4
    ip:port
- IPv6
    [ip]:port

### go a.addressHandler()

同时，启动一个goroutine去定时处理地址。

```
// addressHandler is the main handler for the address manager.  It must be run
// as a goroutine.
func (a *AddrManager) addressHandler() {
    dumpAddressTicker := time.NewTicker(dumpAddressInterval)
    defer dumpAddressTicker.Stop()
out:
    for {
        select {
        case <-dumpAddressTicker.C:
            a.savePeers()

        case <-a.quit:
            break out
        }
    }
    a.savePeers()
    a.wg.Done()
    log.Trace("Address handler done")
}
```
这里，代码比较简单。间隔10分钟保存一次内存中的节点数据到配置文件peers.json中:

a.savePeers()