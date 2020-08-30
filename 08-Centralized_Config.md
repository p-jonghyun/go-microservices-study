이번에는 Go Microservice 설정 부분들을 Spring Cloud Config를 사용해 구성해보겠습니다. 

### Intro

마이크로서비스 아키텍처를 가져가는 것 자체가 시스템을 자기 역할만 하는 서비스들로 나누느 것인데 어떤 것을 중심화하는 것이 좀 이상하게 보일 수 있습니다. 그런데 우리가 나누고자 하는 것은 사실 프로세스 및 로직들입니다. 다른 부분들은 centralized하게 가져갈 수 있지 않을까요? 예를들어, 로그들은 elk stack에 기록이 되어야 하거나, 모니터링 등이 있습니다.

사실 컨테이너들과 노드들이 계속 없어지고 생성되는 마이크로서비스 아키텍처에서 각각의 서비스들이 자신만의 설정들을 가지게 하는 것도 또한 골치거리가 될 것입니다. Centralized Configuration을 제공하는 많은 서비스들이 있는데요 (etcd, consul, zookeper 등) 이러한 서비스들은 설정말고도 다른 서비스들을 사용합니다. 그래서 centralized config에만 집중하는 spring cloud configuration을 사용하여 우리 마이크로서비스들과 통합해 보겠습니다. 

### Spring Cloud Config

스프링 클라우드 Config 서버는 마이크로서비스들과 설정파일들 사이의 프록시라고 생각하시면됩니다. 스프링 클라우드 config는 

- 여러가지 configuration 백엔드를 지원합니다.
    - git, file system, etcd plugin, consul, zookeeper
- 암호화
- 푸시 메커니즘 : 컨피그 파일이 변경이 있을 때 알림
    - git hooks / rest api / spring cloud bus

이번 블로그 포스트에서는 accountservice를 스프링클라우드 컨비그 서버 + public git repo와 통합 해보겠습니다. 아래 이미지가 우리가 할 일에 대해서 간단히 설명해줍니다. 

![img/part8-springcloudconfig.png](img/part8-springcloudconfig.png)

### Overview

우리는 도커 스웜 환경에서 개발중이기 때문에, 도커 메커니즘을 사용할 건데요. 스웜 안에서 우리는 최소 한개의 Spring Cloud Configuration server를 두겠습니다. 마이크로서비스 하나가 시작되면 그녀석은 아래 정보만 알면 통신이 가능해집니다. 

- config server의 서비스명과 포트 번호 ex ) configserver, 8888
- 자신의 서비스 이름 ex ) accontservice
- 현재 개발 환경 프로파일 ex ) dev, test, prod

```bash
protocol://url:port/applicationName/profile/branch
```

### Setting up Spring Cloud Configuration Server

config server를 배포하는 법은 여러가지 방법이 있겠지만, 편의를 위해 

[https://github.com/p-jonghyun/go-microservices-study](https://github.com/p-jonghyun/go-microservices-study/tree/master)에서 support폴더에 필요한 3rd party 서비스들을 두었습니다. 필요한 각 외부 서비스들은 dockerfile로 정의가 되있습니다. 

### RabbitMQ

왜 spring cloud config를 설치하는데 rabbitmq가 필요할까요? spring cloud config는 configuration이 변경이 있을 때 Spring Cloud Bus를 통해 메시지를 전달하는데 이 때 rabbitmq가 필요합니다. 또한 블로그 시리즈 동안 rabbitmq를 쓸 일이 있으니 유용할 겁니다. 

```bash
➜ docker build -t jong1994/rabbitmq .
➜ docker push jong1994/rabbitmq
```

```bash
docker@swarm-manager-1:~$ docker pull jong1994/rabbitmq
docker@swarm-manager-1:~$ docker service create --name=rabbitmq --replicas=1 --network=my_network -p 1883:1883 -p 5672:5672 -p 15672:15672 jong1994/rabbitmq
```

```bash
➜ open http://$ManagerIP:15672/#
```

![img/rabbitmq1.png](img/rabbitmq1.png)

### Spring Cloud Server

/suppor/config-server에는 미리 설정이 완료된 spring boot applicaiton이 있습니다. 우리는 git repo를 저장소를 yml파일로 등록하겠습니다. 

### /support/config-server/src/main/resources/application.yml

```yaml
---
# For deployment in Docker containers
spring:
  profiles: docker
  cloud:
    config:
      server:
        git:
          uri: https://github.com/eriklupander/go-microservice-config.git
          
# Home-baked keystore for encryption. Of course, a real environment wouldn't expose passwords in a blog...          
encrypt:
  key-store:
    location: file:/server.jks
    password: letmein
    alias: goblogkey
    secret: changeme

# Since we're running in Docker Swarm mode, disable Eureka Service Discovery
eureka:
  client:
    enabled: false

# Spring Cloud Config requires rabbitmq, use the service name.
spring.rabbitmq.host: rabbitmq
spring.rabbitmq.port: 5672
```

- config-server에게 해당 uri 리포에서 가져오라고 합니다.
- self-sigend 키스토어
- 스웜 모드이기 때문에 유레카를 끕니다
- rabbitmq 서비스를 알려줍니다

```docker
FROM davidcaste/alpine-java-unlimited-jce

EXPOSE 8888

ADD ./build/libs/*.jar app.jar
ADD ./server.jks /

ENTRYPOINT ["java","-Dspring.profiles.active=docker","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

- 베이스 이미지를 encryption/decryption 기능이 있는 자바 이미지로 사용합니다

### Keystore 만들기

나중에 암호화된 설정을 사용하기 위해서 config server를 self-signed 인증서로 보안 설정을 하겠습니다 

```bash
keytool -genkeypair -alias goblogkey -keyalg RSA \
-dname "CN=Go Blog,OU=Unit,O=Organization,L=City,S=State,C=SE" \  
-keypass changeme -keystore server.jks -storepass letmein \
-validity 730
```

server.jks가 생성되었을겁니다!

### 배포

```bash
➜ ./gradlew build
➜ docker build -t jong1994/configserver .
➜ docker push jong1994/configserver
```

```bash
docker@swarm-manager-1:~$ docker pull jong1994/configserver
docker@swarm-manager-1:~$ docker service create --replicas 1 --name configserver -p 8888:8888 --network my_network --update-delay 10s --with-registry-auth  --update-parallelism 1 jong1994/configserver
```

```bash
> curl http://$ManagerIP:8888/accountservice/dev/master
     {"name":"accountservice","profiles":["dev"],"label":"master","version":"b8cfe2779e9604804e625135b96b4724ea378736",
     "propertySources":[
        {"name":"https://github.com/eriklupander/go-microservice-config.git/accountservice-dev.yml",
        "source":
            {"server_port":6767,"server_name":"Accountservice DEV"}
        }]
     }
```

### YAML config files

configuration-repo의 P8 브랜치를 한번 봅시다 

```bash
accountservice-dev.yml
accountservice-test.yml
......
server_port: 6767
server_name: Accountservice TEST
the_password: (we'll get back to this one)
```

현재 우리가 설정중인거는 사실 http port 밖에 없습니다. 

### Encryption / Decryption

Spring cloud Config가 유용한 점 중 하나는 encrypted된 정보를 transparently decrypt하는 부분인데요. accountservice-test.yml을 예로 봅시다 

```bash
server_port: 6767
server_name: Accountservice TEST
the_password: '{cipher}AQB1BMFCu5UsCcTWUwEQt293nPq0ElEFHHp5B2SZY8m4kUzzqxOFsMXHaH7SThNNjOUDGxRVkpPZEkdgo6aJFSPRzVF04SXOVZ6Rjg6hml1SAkLy/k1R/E0wp0RrgySbgh9nNEbhzqJz8OgaDvRdHO5VxzZGx8uj5KN+x6nrQobbIv6xTyVj9CSqJ/Btf/u1T8/OJ54vHwi5h1gSvdox67teta0vdpin2aSKKZ6w5LyQocRJbONUuHyP5roCONw0pklP+2zhrMCy0mXhCJSnjoHvqazmPRUkyGcjcY3LHjd39S2eoyDmyz944TKheI6rWtCfozLcIr/wAZwOTD5sIuA9q8a9nG2GppclGK7X649aYQynL+RUy1q7T7FbW/TzSBg='
```

{cipeher}을 Prefix로 둠으로서, 스프링 클라우드가 자동적으로 어떻게 이것을 decrypt하고 서비스에게 알려주는 것인데요. 실제로 서비스에서 이 정보를 가져올 때 

```json
"source": {
        "server_port": 6767,
        "server_name": "Accountservice TEST",
        "the_password": "password"
}
```

이런식으로 돌려줄것입니다. 

### Viper

이제 서버를 구축했으니, Go 마이크로서비스에서 Configuration Framework인 Viper를 통해서 설정을 해봅시다. Viper는 유용한 유연하고 application 코드를 건드리지도 않고도 설정을 하게 해주는 API들을 가지고 잇습니다. Viper가 스프링 클라우드 서버에서 설정을 가져오는 것을 지원하지 않기 때문에 우리가 직접 코드를 작성하겠습니다. Viper는 다양한 파일들을 지원하고, OS에서도 환경변수를 읽어올수 있습니다. 한번 초기화가 되면 viper.Get* 함수드을 통해 configuration들을 가져올 수 있습니다. 블로그 맨 위의 사진이 기억나시나요? 

![img/part8-springcloudconfig.png](img/part8-springcloudconfig.png)

우리의 마이크로서비스들이 HTTP request를 시작과 동시에 보낼 것이고, JSON Response에서 각 서비스들이 실행해야 할 port 정보를 가져오도록 하겠습니다 ! 

### Loading the Confguration

전에 curl을 이용하여 보여드렸듯이, config server에 http request를 서비스 이름과 profile만 보내서 정보들을 가져올 수 있습니다. accountservice Dockerfile에 flag를 두어서 필요한 정보들을 읽어오도록 수정하겠습니다. 

### /accountservice/main.go

```go
var appName = "accountservice"

// Init function, runs before main()
func init() {
        // Read command line flags
        profile := flag.String("profile", "test", "Environment profile, something similar to spring profiles")
        configServerUrl := flag.String("configServerUrl", "http://configserver:8888", "Address to config server")
        configBranch := flag.String("configBranch", "master", "git branch to fetch configuration from")
        flag.Parse()
        
        // Pass the flag values into viper.
        viper.Set("profile", *profile)
        viper.Set("configServerUrl", *configServerUrl)
        viper.Set("configBranch", *configBranch)
}

func main() {
        fmt.Printf("Starting %v\n", appName)

        // NEW - load the config
        config.LoadConfigurationFromBranch(
                viper.GetString("configServerUrl"),
                appName,
                viper.GetString("profile"),
                viper.GetString("configBranch"))
        initializeBoltClient()
        service.StartWebServer(viper.GetString("server_port"))    // NEW, use port from loaded config 
}
```

### /accountservices/config/loader.go

```go
package config

import (
	"encoding/json"
	"fmt"
	"github.com/spf13/viper"
	"io/ioutil"
	"net/http"
)

// Loads config from for example http://configserver:8888/accountservice/test/P8
func LoadConfigurationFromBranch(configServerUrl string, appName string, profile string, branch string) {
	url := fmt.Sprintf("%s/%s/%s/%s", configServerUrl, appName, profile, branch)
	fmt.Printf("Loading config from %s\n", url)
	body, err := fetchConfiguration(url)
	if err != nil {
		panic("Couldn't load configuration, cannot start. Terminating. Error: " + err.Error())
	}
	parseConfiguration(body)
}

// Make HTTP request to fetch configuration from config server
func fetchConfiguration(url string) ([]byte, error) {
	resp, err := http.Get(url)
	if err != nil {
		panic("Couldn't load configuration, cannot start. Terminating. Error: " + err.Error())
	}
	body, err := ioutil.ReadAll(resp.Body)
	return body, err
}

// Pass JSON bytes into struct and then into Viper
func parseConfiguration(body []byte) {
	var cloudConfig springCloudConfig
	err := json.Unmarshal(body, &cloudConfig)
	if err != nil {
		panic("Cannot parse configuration, message: " + err.Error())
	}

	for key, value := range cloudConfig.PropertySources[0].Source {
		viper.Set(key, value)
		fmt.Printf("Loading config property %v => %v\n", key, value)
	}
	if viper.IsSet("server_name") {
		fmt.Printf("Successfully loaded configuration for service %s\n", viper.GetString("server_name"))
	}
}

// Structs having same structure as response from Spring Cloud Config
type springCloudConfig struct {
	Name            string           `json:"name"`
	Profiles        []string         `json:"profiles"`
	Label           string           `json:"label"`
	Version         string           `json:"version"`
	PropertySources []propertySource `json:"propertySources"`
}

type propertySource struct {
	Name   string                 `json:"name"`
	Source map[string]interface{} `json:"source"`
}
```

사실상 우리는 config server에게 http get요청을 날리는건데요. 그담에 그 response json을 springCloudconfig 구조체로 unmarshalling합니다. 그런다음 그 구조체에서 우리가 필요한 값들을 가져오는거죵 

여기서 스프링 클라우드 서버랑 통신할 때 에러가 나면 바로 go application을 끄도록 panic처리 하였는데요. 패닉 시 Docker swarm이 이를 확인하고 새로운 instance를 다시 띄우도록 노력할 것입니다. 이러한 로직을 의도한 이유는 상대적으로 spring boot application인 config server가 go application보다 보통 로딩이 느리기 때문입니다. 

### 도커 파일 수정

```docker
FROM iron/base
EXPOSE 6767

ADD accountservice-linux-amd64 /
ADD healthchecker-linux-amd64 /

HEALTHCHECK --interval=3s --timeout=3s CMD ["./healthchecker-linux-amd64", "-port=6767"] || exit 1
ENTRYPOINT ["./accountservice-linux-amd64", "-configServerUrl=http://configserver:8888", "-profile=test", "-configBranch=P8"]
```

스웜에서도 서비스들을 업그레이드 후 

```bash
docker@swarm-manager-1:~$ docker logs -f 14bc9ef01268

Starting accountservice
Loading config from http://configserver:8888/accountservice/test/P8
Loading config property server_port => 6767
Loading config property server_name => Accountservice TEST
Loading config property amqp_server_url => amqp://guest:guest@rabbitmq:5672/
Loading config property config_event_bus => springCloudBus
Loading config property invalid.the_password => <n/a>
Successfully loaded configuration for service Accountservice TEST
Seeded 100 fake accounts...
2020/08/30 10:33:47 Starting HTTP service at 6767
```

config들을 잘 가져오는 것을 확인할 수 있습니다

### Live Configuration Updates

configuration 하나만 바꼈다고 애플리케이션을 rebuild를 한 경험이 다들 있을 겁니다. Spring Cloud는 `@RefreshScopes`라는 개념이 있는데요. 등록된 빈들이 git commit hook에 의해 bean들이 바로 업데이트되는 거죠

![img/img/part8-springcloudbusexchange.png](img/part8-springcloudbusexchange.png)

우리는 일단 깃허브 repository를 사용하고 있습니다. 현재 상황으로는 그러면 configuration server로 post-commit hook operation을 보낼 방법이 없는데요. 그래서 spring cloud config server의 /montior 엔드포인트로 commit hook을 보낼 겁니다. 

```bash
curl -H "X-Github-Event: push" -H "Content-Type: application/json" -X POST -d '{"commits": [{"modified": ["accountservice.yml"]}],"name":"some name..."}' -ki http://$ManagerIP:8888/monitor
```

스프링 클라우드는 이를 받으면, RabbitMQ에 `RefreshRemoteApplicationEvent` 를 publish합니다. RabbitMQ GUI환경에서도 실제로 확인 할 수 있죠. 

![img/part8-springcloudbusexchange.png](img/part8-springcloudbusexchange.png)

그래서 도대체 어떻게 흘러가는 것일 까요 

```
Publisher -> Exchange -> (Routing) -> Queue -> Consumer
```

메시지가 exchange로 Publish되면 dedicated consumer에게 전달하기 위해 queue로 메시지를 복사합니다. 

그래서 Go Application에서 이 `RefreshRemoteApplicationEvent` 을 소화하기 위해 go service가 springCloudbus exchange의 해당 토픽을 listening하면 되죠 

### GO - AMQP Protocol

RabbitMQ broker는 amqp 프로토콜로 접근이 가능합니다. Go에서는 `streadway/amqp` 라는 amqp 클라이언트 라이브러리가 있네요. 

### /accountservice/main.go

```go
func main() {
        fmt.Printf("Starting %v\n", appName)

        config.LoadConfigurationFromBranch(
                viper.GetString("configServerUrl"),
                appName,
                viper.GetString("profile"),
                viper.GetString("configBranch"))
        initializeBoltClient()
        
        // NEW
        go config.StartListener(appName, viper.GetString("amqp_server_url"), viper.GetString("config_event_bus"))   
        service.StartWebServer(viper.GetString("server_port"))
}
```

- amqp_server_url과 config_event_bus를 바이퍼로 통해서 속성값들을 읽어오겠습니다

### /accountservice/config/events.go

```go
package config

import (
	"fmt"
	"log"
	"github.com/streadway/amqp"
	"encoding/json"
	"strings"
	"github.com/spf13/viper"
)

func StartListener(appName string, amqpServer string, exchangeName string) {
	err := NewConsumer(amqpServer, exchangeName, "topic", "config-event-queue", exchangeName, appName)
	if err != nil {
		log.Fatalf("%s", err)
	}

	log.Printf("running forever")
	select {}   // Yet another way to stop a Goroutine from finishing...
}

type Consumer struct {
	conn    *amqp.Connection
	channel *amqp.Channel
	tag     string
	done    chan error
}

func NewConsumer(amqpURI, exchange, exchangeType, queue, key, ctag string) error {
	c := &Consumer{
		conn:    nil,
		channel: nil,
		tag:     ctag,
		done:    make(chan error),
	}

	var err error

	log.Printf("dialing %s", amqpURI)
	c.conn, err = amqp.Dial(amqpURI)
	if err != nil {
		return fmt.Errorf("Dial: %s", err)
	}

	log.Printf("got Connection, getting Channel")
	c.channel, err = c.conn.Channel()
	if err != nil {
		return fmt.Errorf("Channel: %s", err)
	}

	log.Printf("got Channel, declaring Exchange (%s)", exchange)
	if err = c.channel.ExchangeDeclare(
		exchange,     // name of the exchange
		exchangeType, // type
		true,         // durable
		false,        // delete when complete
		false,        // internal
		false,        // noWait
		nil,          // arguments
	); err != nil {
		return fmt.Errorf("Exchange Declare: %s", err)
	}

	log.Printf("declared Exchange, declaring Queue (%s)", queue)
	state, err := c.channel.QueueDeclare(
		queue, // name of the queue
		false,  // durable
		false, // delete when usused
		false, // exclusive
		false, // noWait
		nil,   // arguments
	)
	if err != nil {
		return fmt.Errorf("Queue Declare: %s", err)
	}

	log.Printf("declared Queue (%d messages, %d consumers), binding to Exchange (key '%s')",
		state.Messages, state.Consumers, key)

	if err = c.channel.QueueBind(
		queue,    // name of the queue
		key,      // bindingKey
		exchange, // sourceExchange
		false,    // noWait
		nil,      // arguments
	); err != nil {
		return fmt.Errorf("Queue Bind: %s", err)
	}

	log.Printf("Queue bound to Exchange, starting Consume (consumer tag '%s')", c.tag)
	deliveries, err := c.channel.Consume(
		queue, // name
		c.tag, // consumerTag,
		false, // noAck
		false, // exclusive
		false, // noLocal
		false, // noWait
		nil,   // arguments
	)
	if err != nil {
		return fmt.Errorf("Queue Consume: %s", err)
	}

	go handle(deliveries, c.done)

	return nil
}

func (c *Consumer) Shutdown() error {
	// will close() the deliveries channel
	if err := c.channel.Cancel(c.tag, true); err != nil {
		return fmt.Errorf("Consumer cancel failed: %s", err)
	}

	if err := c.conn.Close(); err != nil {
		return fmt.Errorf("AMQP connection close error: %s", err)
	}

	defer log.Printf("AMQP shutdown OK")

	// wait for handle() to exit
	return <-c.done
}

func handle(deliveries <-chan amqp.Delivery, done chan error) {
	for d := range deliveries {
		log.Printf(
			"got %dB consumer: [%v] delivery: [%v] routingkey: [%v] %s",
			len(d.Body),
			d.ConsumerTag,
			d.DeliveryTag,
			d.RoutingKey,
			d.Body,
		)
		handleRefreshEvent(d.Body, d.ConsumerTag)
		d.Ack(false)
	}
	log.Printf("handle: deliveries channel closed")
	done <- nil
}

func handleRefreshEvent(body []byte, consumerTag string) {
	updateToken := &UpdateToken{}
	err := json.Unmarshal(body, updateToken)
	if err != nil {
		log.Printf("Problem parsing UpdateToken: %v", err.Error())
	} else {
		if strings.Contains(updateToken.DestinationService, consumerTag) {
			log.Println("Reloading Viper config from Spring Cloud Config server")

			// Consumertag is same as application name.
			LoadConfigurationFromBranch(
				viper.GetString("configServerUrl"),
				consumerTag,
				viper.GetString("profile"),
				viper.GetString("configBranch"))
		}
	}
}

// {"type":"RefreshRemoteApplicationEvent","timestamp":1494514362123,"originService":"config-server:docker:8888","destinationService":"xxxaccoun:**","id":"53e61c71-cbae-4b6d-84bb-d0dcc0aeb4dc"}
type UpdateToken struct {
	Type string `json:"type"`
	Timestamp int `json:"timestamp"`
	OriginService string `json:"originService"`
	DestinationService string `json:"destinationService"`
	Id string `json:"id"`
}
```

accountservice 재배포후 

```bash
> docker logs -f [containerid]
Starting accountservice
... [truncated for brevity] ...
Successfully loaded configuration for service Accountservice TEST    <-- LOOK HERE!!!!
... [truncated for brevity] ...
2017/05/12 12:06:36 dialing amqp://guest:guest@rabbitmq:5672/
2017/05/12 12:06:36 got Connection, getting Channel
2017/05/12 12:06:36 got Channel, declaring Exchange (springCloudBus)
2017/05/12 12:06:36 declared Exchange, declaring Queue (config-event-queue)
2017/05/12 12:06:36 declared Queue (0 messages, 0 consumers), binding to Exchange (key 'springCloudBus')
2017/05/12 12:06:36 Queue bound to Exchange, starting Consume (consumer tag 'accountservice')
2017/05/12 12:06:36 running forever

> curl -H "X-Github-Event: push" -H "Content-Type: application/json" -X POST -d '{"commits": [{"modified": ["accountservice.yml"]}],"name":"what is this?"}' -ki http://localhost:8888/monitor

> docker logs -f [containerid]
2017/05/12 12:13:22 got 195B consumer: [accountservice] delivery: [1] routingkey: [springCloudBus] {"type":"RefreshRemoteApplicationEvent","timestamp":1494591202057,"originService":"config-server:docker:8888","destinationService":"accountservice:**","id":"1f421f58-cdd6-44c8-b5c4-fbf1e2839baa"}
2017/05/12 12:13:22 Reloading Viper config from Spring Cloud Config server
Loading config from http://configserver:8888/accountservice/test/P8
Loading config property server_port => 6767
Loading config property server_name => Temporary test string!
Loading config property amqp_server_url => amqp://guest:guest@rabbitmq:5672/
Loading config property config_event_bus => springCloudBus
Loading config property the_password => password
Successfully loaded configuration for service Temporary test string!      <-- LOOK HERE!!!!
```

제대로 환경 설정들이 dynamically하게 변경됨을 확인할 수 있습니다 ㅎㅎ
