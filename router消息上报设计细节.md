## router--->controller:

### router发出消息：

执行结果的类型是ExecResult, 定义如下：

```go
type ExecResult struct {
	RuleID    string
	ProjectID string
	Status    string
	Error     ErrorMsg
}

type ErrorMsg struct {
	Detail    string
	Timestamp time.Time
}
```

将Execresult解析成message的形式，可参考downstream里syncRule()：

在statushandler()里调用：

```go
func NewMessage(parentID string) *Message {
	msg := &Message{}
	msg.Header.ID = uuid.NewV4().String()
	msg.Header.ParentID = parentID
	msg.Header.Timestamp = time.Now().UnixNano() / 1e6
	return msg
}

```

记录一下构造message的细节：

message router由这4部分构成：

Source:where the messsage come from

Group:where the message will broadcast to

Operation

Resource

其中构造resource的函数需要用到的常数在model里

```
resource = "node/nodename/rule.Namespace/rule/rule.id"
因为rule和节点不是强绑定关系，可以直接写nodename字符串.

const (
	InsertOperation        = "insert"
	DeleteOperation        = "delete"
	QueryOperation         = "query"
	UpdateOperation        = "update"
	ResponseOperation      = "response"
	ResponseErrorOperation = "error"

	ResourceTypePod          = "pod"
	ResourceTypeConfigmap    = "configmap"
	ResourceTypeSecret       = "secret"
	ResourceTypeNode         = "node"
	ResourceTypePodlist      = "podlist"
	ResourceTypePodStatus    = "podstatus"
	ResourceTypeNodeStatus   = "nodestatus"
	ResourceTypeRule         = "rule"
	ResourceTypeRuleEndpoint = "ruleendpoint"
)

```

其中构造router的常数在modules里面：

```
const (
	CloudHubModuleName  = "cloudhub"
	CloudHubModuleGroup = "cloudhub"

	EdgeControllerModuleName = "edgecontroller"
	EdgeControllerGroupName  = "edgecontroller"

	DeviceControllerModuleName  = "devicecontroller"
	DeviceControllerModuleGroup = "devicecontroller"

	SyncControllerModuleName  = "synccontroller"
	SyncControllerModuleGroup = "synccontroller"

	DynamicControllerModuleName  = "dynamiccontroller"
	DynamicControllerModuleGroup = "dynamiccontroller"

	CloudStreamModuleName = "cloudStream"
	CloudStreamGroupName  = "cloudStream"

	RouterModuleName = "router"
	RouterGroupName  = "router"

	UserGroup = "user"
)

```

然后调用Beehive发送：

```go
beehiveContext.Send(cml.SendModuleName, message)
```



### controller接收消息：

Upstream里接收message类型的消息，message类型的定义如下：

```go
type Message struct {
	Header  MessageHeader `json:"header"`
	Router  MessageRoute  `json:"route,omitempty"`
	Content interface{}   `json:"content"`
}



// MessageRoute contains structure of message
type MessageRoute struct {
   // where the message come from
   Source string `json:"source,omitempty"`
   // where the message will broadcast to
   Group string `json:"group,omitempty"`

   // what's the operation on resource
   Operation string `json:"operation,omitempty"`
   // what's the resource want to operate
   Resource string `json:"resource,omitempty"`
}

// MessageHeader defines message header details
type MessageHeader struct {
   // the message uuid
   ID string `json:"msg_id"`
   // the response message parentid must be same with message received
   // please use NewRespByMessage to new response message
   ParentID string `json:"parent_msg_id,omitempty"`
   // the time of creating
   Timestamp int64 `json:"timestamp"`
   // specific resource version for the message, if any.
   // it's currently backed by resource version of the k8s object saved in the Content field.
   // kubeedge leverages the concept of message resource version to achieve reliable transmission.
   ResourceVersion string `json:"resourceversion,omitempty"`
   // the flag will be set in sendsync
   Sync bool `json:"sync,omitempty"`
}
```

接收消息的接口是Receive()，其实就是使用beehive框架接收消息：

```Go
type MessageLayer interface {
	Send(message model.Message) error
	Receive() (model.Message, error)
	Response(message model.Message) error
}
```

```go
func (cml *ContextMessageLayer) Receive() (model.Message, error) {
	return beehiveContext.Receive(cml.ReceiveModuleName)
}
```

在upstream需要修改的操作是：

新增ruleStatusChan

修改dispatchMessage()

## controller ---> k8s:

新增函数ruleStatusUpdate():

可以直接调用get patch等方法实现更新

## 总结：

1.在结构体中定义的变量，不要忘记初始化

```go
type UpstreamController struct {
	kubeClient   kubernetes.Interface
	messageLayer messagelayer.MessageLayer
	crdClient    crdClientset.Interface
}

func NewUpstreamController(factory k8sinformer.SharedInformerFactory) (*UpstreamController, error) {
	uc := &UpstreamController{
		kubeClient:   client.GetKubeClient(),
		messageLayer: messagelayer.NewContextMessageLayer(),
		crdClient:    client.GetCRDClient(),
	}
```

2.关于statushandler的处理：使用了一个for循环，不停的从channel中提取消息。

```go
func do(stop chan bool) {
	ResultChannel = make(chan ExecResult, 1024)

	for {
		select {
		case r := <-ResultChannel:
			msg := model.NewMessage("")
			resource, err := messagelayer.BuildResourceForRouter(r.ProjectID, model.ResourceTypeRuleStatus, r.RuleID) //message构建时需要的常数都在model里定义过
			if err != nil {
				klog.Warningf("build message resource failed with error: %s", err)
				continue
			}
			msg.Content = r
			msg.BuildRouter(modules.RouterModuleName, constants.GroupResource, resource, model.UpdateOperation) //modules里定义了
			beehiveContext.Send(modules.EdgeControllerModuleName, *msg)
			klog.V(4).Infof("send message successfully, operation: %s, resource: %s", msg.GetOperation(), msg.GetResource())
		case _, ok := <-stop:
			if !ok {
				klog.Warningf("do stop channel is closed")
			}
			return
		}

	}
}
```

3.要注意当不同模块间消息的格式不同时，要在各自模块的目录下分别定义，后续可改为统一格式的消息。

```go
edgecontroller中的：
// BuildResourceForRouter return a string as "beehive/pkg/core/model".Message.Router.Resource
func BuildResourceForRouter(resourceType, resourceID string) (string, error) {
	if resourceID == "" || resourceType == "" {
		return "", fmt.Errorf("required parameter are not set (resourceID or resource type)")
	}
	return fmt.Sprintf("%s%s%s", resourceType, constants.ResourceSep, resourceID), nil
}

router中的：
// BuildResourceForRouter return a string as "beehive/pkg/core/model".Message.Router.Resource
func BuildResourceForRouter(namespace ,resourceType, resourceID string) (string, error) {
	if namespace == ""{
		namespace = "default"
	}
	if resourceID == "" || resourceType == "" {
		return "", fmt.Errorf("required parameter are not set (resourceID or resource type)")
	}
	return fmt.Sprintf("node/nodeid/%s%s%s%s%s", namespace, constants.ResourceSep, resourceType, constants.ResourceSep, resourceID), nil
}
```

