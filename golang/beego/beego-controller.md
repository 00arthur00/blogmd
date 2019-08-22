title: beego 控制器分发
tags:
  - golang
  - beego
date: 2018-11-23 09:17:51
---
## beego+JWT遇到的问题
* 在使用[JWT](github.com/dgrijalva/jwt-go)生成token时，使用了如下结构：
``` golang
type Login struct {
	beego.Controller
	Session *mgo.Session
	jwtKey  []byte
	count   int
}
```

Session用于存储mongodb的session，jwtKey为JWT的密钥用于生成及解密token。

* 创建新的controller的工厂函数为：
``` golang
func NewLoginContoller() *Login {
	jwtKey := helper.JWTKey()
	beego.Info("jwtkey:", jwtKey)
	login := &Login{Session: phonepointerSession, jwtKey: jwtKey}
	return login
}
```

* 生成token的函数为：
``` golang
func (this *Login) generateToken(user, pwd string) (string, error) {
	fmt.Println(user, pwd, this.jwtKey)
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
		"user": user,
		"pwd":  pwd,
	})

	// Sign and get the complete encoded token as a string using the secret
	tokenString, err := token.SignedString(this.jwtKey)
	if err != nil {
		return "", fmt.Errorf("generateToken:%s", err)
	}
	return tokenString, nil
}
```

* 应为使用了[socket.io](github.com/googollee/go-socket.io)所以用beego的handler直接注册的，而登陆用的controller则用LoginContoller的对象指针创建：
``` golang
var login *controllers.Login
login = controllers.NewLoginContoller()
beego.Router("/login", login)
socketio, err := controllers.NewSocketIOController()
if err != nil {
	beego.Error("socketio controller:%s", err)
} else {
	beego.InsertFilter("/socket.io/", beego.BeforeExec, filterJWT)
	beego.Handler("/socket.io/", socketio)
}
```

* 同时需要注册filter函数：
``` golang
var filterJWT = func(c *context.Context) {
	if err := c.Request.ParseForm(); err != nil {
		beego.Error("filterJWT:", err)
		return
	}
	tokenString := c.Request.Form.Get("token")
	// fmt.Println(tokenString)
	token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
		// Don't forget to validate the alg is what you expect:
		if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
			return nil, fmt.Errorf("Unexpected signing method: %v", token.Header["alg"])
		}

		// hmacSampleSecret is a []byte containing your secret, e.g. []byte("my_secret_key")
		return helper.JWTKey(), nil
	})
	if claims, ok := token.Claims.(jwt.MapClaims); ok && token.Valid {
		c.Input.SetData("user", claims["user"])
		c.Input.SetData("pwd", claims["pwd"])
	}
	if err != nil {
		beego.Error("filterJWT:", err)
		c.Redirect(302, "/")
	}
}
beego.InsertFilter("/socket.io/", beego.BeforeExec, filterJWT)
```

* 运行时遇到错误：
``` html
2018/11/23 09:39:33.423 [E] [filter.go:34]  filterJWT: signature is invalid
```

## toubleshooting
* 注意到在generateToken函数中的log有一句
```
admin e10adc3949ba59abbe56e057f20f883e []
```
**this.jwtKey为空字节数组**，所以创建新的controller的工厂函数没有起作用。

* 进一步确认问题，在Post函数中每次打印login中的count。
```golang
func (this *Login) Post() {
	this.count++
	fmt.Println(this.count)
```
结果每次都是打印出1，所以明显注册的login不是所有共享的，而是每次请求到来时重新创建出来的。

## router的注册机制
先看看beego.Router("/login", login)干了什么。
Router的实现为：
``` golang
func Router(rootpath string, c ControllerInterface, mappingMethods ...string) *App {
	BeeApp.Handlers.Add(rootpath, c, mappingMethods...)
	return BeeApp
}
```
注册路径"/login"与login类型的对应关系，注意此处是类型而不是对象。深入到BeeApp.Handlers.Add函数，其调用的是addWithMethodParams函数：
``` golang
func (p *ControllerRegister) Add(pattern string, c ControllerInterface, mappingMethods ...string) {
	p.addWithMethodParams(pattern, c, nil, mappingMethods...)
}

func (p *ControllerRegister) addWithMethodParams(pattern string, c ControllerInterface, methodParams []*param.MethodParam, mappingMethods ...string) {
	reflectVal := reflect.ValueOf(c)
	t := reflect.Indirect(reflectVal).Type()//用reflect取出login对应的类型
    ...
    ...
    route := &ControllerInfo{}//初始化route
	route.pattern = pattern
	route.methods = methods
	route.routerType = routerTypeBeego
	route.controllerType = t
    route.initialize = func() ControllerInterface {//initialize用于对象的初始化
		vc := reflect.New(route.controllerType)//创建新对象
		execController, ok := vc.Interface().(ControllerInterface)
		if !ok {
			panic("controller is not ControllerInterface")
		}

		elemVal := reflect.ValueOf(c).Elem()
		elemType := reflect.TypeOf(c).Elem()
		execElem := reflect.ValueOf(execController).Elem()

		numOfFields := elemVal.NumField()
		for i := 0; i < numOfFields; i++ {
			fieldType := elemType.Field(i)
			elemField := execElem.FieldByName(fieldType.Name)
			if elemField.CanSet() {//只有CanSet()为true时才能设置New出来的对象字段
				fieldVal := elemVal.Field(i)
				elemField.Set(fieldVal)
			}
			fmt.Println("fieldType.Name:", fieldType.Name, " CanSet:", elemField.CanSet())
		}

		return execController
	}
    ....
    ...
    p.addToRouter(m, pattern, route)//将patten及route信息进行注册
    ...    
}
```
至此可以看到注册的是对象的类型，并附带一个initialize函数用于对象的初始化。在这里使用到了golang的reflect机制，只有CanSet()为true的字段才在初始化时设置。关于CanSet，其注释解释的很清楚只有可寻址的且是大写的字段才会返回true。
```
CanSet reports whether the value of v can be changed. A Value can be changed only if it is addressable and was not obtained by the use of unexported struct fields. If CanSet returns false, calling Set or any type-specific setter (e.g., SetBool, SetInt) will panic.
```

**初始化只是对原始（指针）对象进行了进行了拷贝并对相应的字段进行赋值，并不能用于状态的更新，因为每次都是拷贝的注册的时候的值，即使更新也会被原始值覆盖**

## router的分发
* beego实际上是一个App对象:

``` golang
var (
	// BeeApp is an application instance
	BeeApp *App
)

func init() {
	// create beego application
	BeeApp = NewApp()
}

// App defines beego application with a new PatternServeMux.
type App struct {
	Handlers *ControllerRegister //http handler
	Server   *http.Server //http server
}
```

其中Handlers是一个ControllerRegister对象，提供ServeHTTP方法供Server调用.

* beego.Run()会将Handlers赋值给Server的Handler。当server将htt prequest准备好之后会调用Handler的ServeHTTP函数。

``` golang
// Run beego application.
func (app *App) Run(mws ...MiddleWare) {
    ...
    ...
    //赋值并初始化server的readtimeout/writetimeout
    app.Server.Handler = app.Handlers
    ...
    app.Server.ReadTimeout = time.Duration(BConfig.Listen.ServerTimeOut) * time.Second
	app.Server.WriteTimeout = time.Duration(BConfig.Listen.ServerTimeOut) * time.Second
	app.Server.ErrorLog = logs.GetLogger("HTTP")
    ...
}
```

* 真正的分发在ServeHTTP中：
``` golang
func (p *ControllerRegister) ServeHTTP(rw http.ResponseWriter, r *http.Request) {
    var (
		runRouter    reflect.Type //Router类型，对"/login"来说就是Login
		findRouter   bool         // 路由是否找到router
		runMethod    string       // http方法
		methodParams []*param.MethodParam //中间件函数
		routerInfo   *ControllerInfo //路由信息
		isRunnable   bool            //是否可运行，对routerTypeRESTFul和routerTypeHandler为true，若为true则不生成新的对象。默认为false。
	)
    context := p.pool.Get().(*beecontext.Context)
	context.Reset(rw, r)
    ...
    ...
    routerInfo, findRouter = p.FindRouter(context)//找到routerInfo
    ...
    ...
    if routerInfo != nil {
		//store router pattern into context
		context.Input.SetData("RouterPattern", routerInfo.pattern)
		if routerInfo.routerType == routerTypeRESTFul {//routerTypeRESTFul，isRunnable=true
			if _, ok := routerInfo.methods[r.Method]; ok {
				isRunnable = true
				routerInfo.runFunction(context)
			} else {
				exception("405", context)
				goto Admin
			}
		} else if routerInfo.routerType == routerTypeHandler {//routerTypeHandler,socket.io使用的类型， isRunnable=true
			isRunnable = true
			routerInfo.handler.ServeHTTP(rw, r)//直接执行ServeHTTP
		} else {//routerTypeBeego,我们的Router使用的类型，默认isRunnable=false
			runRouter = routerInfo.controllerType
			methodParams = routerInfo.methodParams
			method := r.Method
            //确认method并赋值
			if r.Method == http.MethodPost && context.Input.Query("_method") == http.MethodPost {
				method = http.MethodPut
			}
			if r.Method == http.MethodPost && context.Input.Query("_method") == http.MethodDelete {
				method = http.MethodDelete
			}
			if m, ok := routerInfo.methods[method]; ok {
				runMethod = m
			} else if m, ok = routerInfo.methods["*"]; ok {
				runMethod = m
			} else {
				runMethod = method
			}
		}
	}
    ...
    // also defined runRouter & runMethod from filter
    if !isRunnable {
        //Invoke the request handler
		var execController ControllerInterface
		if routerInfo.initialize != nil {
			execController = routerInfo.initialize()//进行初始化
		} else {
			vc := reflect.New(runRouter)
			var ok bool
			execController, ok = vc.Interface().(ControllerInterface)
			if !ok {
				panic("controller is not ControllerInterface")
			}
		}
        ...
    }
    ...
}
```

首先根据request初始化context，根据context找到router，因为router类型即不是routerTypeRESTFul也不是routerTypeHandler，所以初始化runRouter，这里routerInfo.initialize已经设置了，直接进行初始化操作返回一个execController调用具体的函数。

## 结论
对于用beego.Router来注册controller的函数来说，controller是只是注册时（指针）对象的一个拷贝，而且只有CanSet的字段才能赋值，所以不适合用来存储状态。对于注册的socket.io，因为其类型是routerTypeHandler，beego会直接调用对象的ServeHTTP函数，因此可以保存状态。

## bug fix
创建新的helper包用于返回JWT key，加密解密使用相同的Key，在Login结构中不包含key的值。新的Login结构如下：
``` golang
type Login struct {
	beego.Controller
}
```
同理，将Session,jwtKey,count字段删除，删除NewLoginContoller函数。创建helper包，用于返回JWT key，用于生成token。
``` golang
func (this *Login) generateToken(user, pwd string) (string, error) {
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
		"user": user,
		"pwd":  pwd,
	})

	// Sign and get the complete encoded token as a string using the secret
	tokenString, err := token.SignedString(helper.JWTKey())
	if err != nil {
		return "", fmt.Errorf("generateToken:%s", err)
	}
	return tokenString, nil
}
```