# burpDevNote
burp插件开发笔记，简单记一下方便查阅。刚入门最好的建议就是模仿，多看别人写的，尝试着修改，尝试着自己写一个。





## 基础知识

burp suite支持三种编程语言开发的插件:

- Java
- python
- ruby

我选java

官方各种示例代码：https://portswigger.net/burp/extender

官方API文档：https://portswigger.net/burp/extender/api/



maven依赖

```xml
		 <dependency>
            <groupId>net.portswigger.burp.extender</groupId>
            <artifactId>burp-extender-api</artifactId>
            <version>1.7.22</version>
        </dependency>
```



- 所有的burp插件都必须实现IBurpExtender这个接口
- 实现类的包名称必须是burp
- 实现类的名称必须是BurpExtender，burp插件入口
- 实现类比较是public的
- 实现类必须有默认构造函数（public，无参），如果没有定义构造函数就是默认构造函数



### callbacks对象的作用

通过 callbacks 这个实例对象，传递给插件一系列burp的原生方法。我们需要实现的很多功能都需要调用这些方法。当前拓展用来和burp进行交互，如各种注册，设置拓展名字，获取helper，添加菜单等等。



### 需要主动注册的对象



也就是需要写在registerExtenderCallbacks方法里面的

他们的共通特定是：会被burp主程序主动调用或者主动通知，burp主程序必须知道它们的存在，才能完成对应的功能。

#### 各种事件监听器

ExtensionStateListener

HttpListener

ProxyListener

ScannerListener

ScopeChangeListener

#### 各种对象构造工厂

ContextMenuFactory

MessageEditorTabFactory

IntruderPayloadGeneratorFactory

#### 其他

ScannerInsertionPointProvider

ScannerCheck

IntruderPayloadProcessor

SessionHandlingAction

MenuItem



## 关键对象



### IExtensionHelpers

用途：主要用来处理数据包

初始化,一般写在registerExtenderCallbacks方法里面

```java
IExtensionHelpers helpers = callbacks.getHelpers();
```

Request和Response获取

```
IRequestInfo analyzeRequest = helpers.analyzeRequest(requestOrResponse);
IResponseInfo analyzeResponse = helpers.analyzeResponse(requestOrResponse);
```

获取header

```
IRequestInfo analyzeRequest = helpers.analyzeRequest(requestOrResponse);
			List<String> headers = analyzeRequest.getHeaders();
```

获取body

```java
	int bodyOffset = -1;
		if(isRequest) {
			IRequestInfo analyzeRequest = helpers.analyzeRequest(requestOrResponse);
			bodyOffset = analyzeRequest.getBodyOffset();
		}else {
			IResponseInfo analyzeResponse = helpers.analyzeResponse(requestOrResponse);
			bodyOffset = analyzeResponse.getBodyOffset();
		}
		byte[] byte_body = Arrays.copyOfRange(requestOrResponse, bodyOffset, requestOrResponse.length);
//not length-1
		//String body = new String(byte_body); //byte[] to String
		return byte_body;
```



修改http包

```
byte[] RequestOrResponse = helpers.buildHttpMessage(headers, body);
```



具体看大佬封装的工具类https://github.com/bit4woo/burp-api-common/blob/master/src/main/java/burp/HelperPlus.java





### IHttpListener

用途：任何组件产生的请求和响应都会触发http监听器，有点被动扫描的味道。

必须实现processHttpMessage方法

```java
processHttpMessage(int toolFlag,boolean messageIsRequest,IHttpRequestResponse messageInfo)
```

toolFlag

不同的toolFlag代表了不同的burp组件,一共如下

 https://portswigger.net/burp/extender/api/constant-values.html#burp.IBurpExtenderCallbacks

![image-20211220212403938](https://gitee.com/safe6/img/raw/master/image-20211220212403938.png)

messageIsRequest，判断当前是不是request

IHttpRequestResponse，具体的消息体对象。可用`helpers#analyzeRequest`对消息体进行解析,messageInfo是整个HTTP请求和响应消息体的总和，各种HTTP相关信息的获取都来自于它，HTTP流量的修改都是围绕它进行的。





```java
package burp;

import java.io.PrintWriter;
import java.util.Arrays;
import java.util.List;

public class BurpExtender implements IBurpExtender, IHttpListener
{//所有burp插件都必须实现IBurpExtender接口，而且实现的类必须叫做BurpExtender
	private IBurpExtenderCallbacks callbacks;
	private IExtensionHelpers helpers;

	private PrintWriter stdout;
	private PrintWriter stderr;
	private String ExtenderName = "burp extender api drops by bit4woo";

	@Override
	public void registerExtenderCallbacks(IBurpExtenderCallbacks callbacks)
	{//IBurpExtender必须实现的方法
		stdout = new PrintWriter(callbacks.getStdout(), true);
		stderr = new PrintWriter(callbacks.getStderr(), true);
		callbacks.printOutput(ExtenderName);
		//stdout.println(ExtenderName);
		this.callbacks = callbacks;
		helpers = callbacks.getHelpers();
		callbacks.setExtensionName(ExtenderName); 
		callbacks.registerHttpListener(this); //如果没有注册，下面的processHttpMessage方法是不会生效的。处理请求和响应包的插件，这个应该是必要的
	}

	@Override
	public void processHttpMessage(int toolFlag,boolean messageIsRequest,IHttpRequestResponse messageInfo)
	{
		if (toolFlag == IBurpExtenderCallbacks.TOOL_PROXY){
			//不同的toolFlag代表了不同的burp组件 https://portswigger.net/burp/extender/api/constant-values.html#burp.IBurpExtenderCallbacks
			if (messageIsRequest){ //对请求包进行处理
				IRequestInfo analyzeRequest = helpers.analyzeRequest(messageInfo); 
				//对消息体进行解析,messageInfo是整个HTTP请求和响应消息体的总和，各种HTTP相关信息的获取都来自于它，HTTP流量的修改都是围绕它进行的。
				
				/*****************获取参数**********************/
				List<IParameter> paraList = analyzeRequest.getParameters();
				//获取参数的方法
				//当body是json格式的时候，这个方法也可以正常获取到键值对；但是PARAM_JSON等格式不能通过updateParameter方法来更新。
				//如果在url中的参数的值是 key=json格式的字符串 这种形式的时候，getParameters应该是无法获取到最底层的键值对的。

				for (IParameter para : paraList){// 循环获取参数，判断类型，进行加密处理后，再构造新的参数，合并到新的请求包中。
					String key = para.getName(); //获取参数的名称
					String value = para.getValue(); //获取参数的值
					int type = para.getType();
					stdout.println("参数 key value type: "+key+" "+value+" "+type);
				}
				
				/*****************修改并更新参数**********************/
				IParameter newPara = helpers.buildParameter("testKey", "testValue", IParameter.PARAM_BODY); //构造新的参数
				byte[] new_Request = messageInfo.getRequest();
				new_Request = helpers.updateParameter(new_Request, newPara); //构造新的请求包
				messageInfo.setRequest(new_Request);//设置最终新的请求包
				
				/*****************删除参数**********************/
				for (IParameter para : paraList){// 循环获取参数，判断类型，进行加密处理后，再构造新的参数，合并到新的请求包中。
					String key = para.getName(); //获取参数的名称
					if (key.equals("aaa")) {
						new_Request = helpers.removeParameter(new_Request, para); //构造新的请求包
					}
				}
				

				/*****************获取header**********************/
				List<String> headers = analyzeRequest.getHeaders();

				for (String header : headers){// 循环获取参数，判断类型，进行加密处理后，再构造新的参数，合并到新的请求包中。
					stdout.println("header "+header);
					if (header.startsWith("referer")) {
				/*****************删除header**********************/
						headers.remove(header);
					}
				}
				
				/*****************新增header**********************/
				headers.add("myheader: balalbala");
				
				
				/*****************获取body 方法一**********************/
				int bodyOffset = analyzeRequest.getBodyOffset();
				byte[] byte_Request = messageInfo.getRequest();
				
				String request = new String(byte_Request); //byte[] to String
				String body = request.substring(bodyOffset);
				byte[] byte_body = body.getBytes();  //String to byte[]
				
				/*****************获取body 方法二**********************/
				
				int len = byte_Request.length;
				byte[] byte_body1 = Arrays.copyOfRange(byte_Request, bodyOffset, len);
				
				new_Request = helpers.buildHttpMessage(headers, byte_body); 
				//如果修改了header或者数修改了body，不能通过updateParameter，使用这个方法。
				messageInfo.setRequest(new_Request);//设置最终新的请求包
			}
		}
		else{//处理响应包
			IResponseInfo analyzedResponse = helpers.analyzeResponse(messageInfo.getResponse()); //getResponse获得的是字节序列
			short statusCode = analyzedResponse.getStatusCode();
			List<String> headers = analyzedResponse.getHeaders();
			String resp = new String(messageInfo.getResponse());
			int bodyOffset = analyzedResponse.getBodyOffset();//响应包是没有参数的概念的，大多需要修改的内容都在body中
			String body = resp.substring(bodyOffset);

			if (statusCode==200){
				String newBody= body+"&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&";
				byte[] bodybyte = newBody.getBytes();
				messageInfo.setResponse(helpers.buildHttpMessage(headers, bodybyte));
			}
		}
	}
}
```



### IContextMenuFactory

![image-20211220215113980](https://gitee.com/safe6/img/raw/master/image-20211220215113980.png)

用途：注册右键菜单，必须实现createMenuItems方法。

如下创建菜单，添加点击事件

```java
@Override
	public List<JMenuItem> createMenuItems(IContextMenuInvocation invocation) {
		
		ArrayList<JMenuItem> menu_item_list = new ArrayList<JMenuItem>();

		//常用
		JMenuItem printEmails = new JMenuItem("Print Emails");
		printEmails.addActionListener(new printEmails(invocation));
		menu_item_list.add(printEmails);

		JMenuItem printCookie = new JMenuItem("find last cookie of Url");
		printCookie.addActionListener(new printCookies(invocation));
		menu_item_list.add(printCookie);

		JMenuItem scan = new JMenuItem("scan this url");
		scan.addActionListener(new printCookies(invocation));
		menu_item_list.add(scan);
		
		return menu_item_list;

	}
```



- 鼠标右键的创建
- 从Scanner issues中收集邮箱地址
- 从Proxy history中查找最新cookie
- 发起扫描任务、发起爬行任务
- 查询和更新scope

```java
package burp;

import java.awt.MenuItem;
import java.awt.event.ActionEvent;
import java.awt.event.ActionListener;
import java.io.PrintWriter;
import java.lang.reflect.Array;
import java.net.URL;
import java.net.URLDecoder;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

import javax.swing.JMenuItem;


public class BurpExtender implements IBurpExtender, IContextMenuFactory
{//所有burp插件都必须实现IBurpExtender接口，而且实现的类必须叫做BurpExtender
	private IBurpExtenderCallbacks callbacks;
	private IExtensionHelpers helpers;

	private PrintWriter stdout;
	private PrintWriter stderr;
	private String ExtenderName = "burp extender api drops by bit4woo";

	@Override
	public void registerExtenderCallbacks(IBurpExtenderCallbacks callbacks)
	{//IBurpExtender必须实现的方法
		stdout = new PrintWriter(callbacks.getStdout(), true);
		stderr = new PrintWriter(callbacks.getStderr(), true);
		callbacks.printOutput(ExtenderName);
		//stdout.println(ExtenderName);
		this.callbacks = callbacks;
		helpers = callbacks.getHelpers();
		callbacks.setExtensionName(ExtenderName); 
		callbacks.registerContextMenuFactory(this);
	}

	@Override
	public List<JMenuItem> createMenuItems(IContextMenuInvocation invocation) {
		
		ArrayList<JMenuItem> menu_item_list = new ArrayList<JMenuItem>();

		//常用
		JMenuItem printEmails = new JMenuItem("Print Emails");
		printEmails.addActionListener(new printEmails(invocation));
		menu_item_list.add(printEmails);

		JMenuItem printCookie = new JMenuItem("find last cookie of Url");
		printCookie.addActionListener(new printCookies(invocation));
		menu_item_list.add(printCookie);

		JMenuItem scan = new JMenuItem("scan this url");
		scan.addActionListener(new printCookies(invocation));
		menu_item_list.add(scan);
		
		return menu_item_list;

	}
	
	public class printEmails implements ActionListener{
		private IContextMenuInvocation invocation;

		public printEmails(IContextMenuInvocation invocation) {
			this.invocation  = invocation;
		}
		
		@Override
		public void actionPerformed(ActionEvent event) {
			IScanIssue[]  issues =  callbacks.getScanIssues(null);

			final String REGEX_EMAIL = "[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\\.[a-zA-Z0-9-.]+";
			Pattern pDomainNameOnly = Pattern.compile(REGEX_EMAIL);

			for (IScanIssue issue:issues) {
				if (issue.getIssueName().equalsIgnoreCase("Email addresses disclosed")) {
					String detail = issue.getIssueDetail();
					Matcher matcher = pDomainNameOnly.matcher(detail);
					while (matcher.find()) {//多次查找
						String email = matcher.group();
						System.out.println(matcher.group());
					}
				}
			}
		}
	}
	
	public class printCookies implements ActionListener{
		private IContextMenuInvocation invocation;

		public printCookies(IContextMenuInvocation invocation) {
			this.invocation  = invocation;
		}
		
		@Override
		public void actionPerformed(ActionEvent event) {
			try {
				IHttpRequestResponse[] messages = invocation.getSelectedMessages();
				byte[] req = messages[0].getRequest();
				String currentShortUrl = messages[0].getHttpService().toString();
				stdout.println(currentShortUrl);
				
				/*******************从Proxy history中查找最新cookie***************************/
		        IHttpRequestResponse[]  historyMessages = callbacks.getProxyHistory();
		        int len =  historyMessages.length;
		        for (int index=len; index >=0; index--) {
		        	IHttpRequestResponse item = historyMessages[index];
		        	
		            String hisShortUrl = item.getHttpService().toString();
		            if (currentShortUrl.equals(hisShortUrl)) {
			            IRequestInfo hisanalyzedRequest = helpers.analyzeRequest(item);
			            List<String> headers = hisanalyzedRequest.getHeaders();

			            for (String header:headers) {
			            	if (header.startsWith("Cookie:")) {
			            		stdout.println("找到cookie---"+header);
			            	}
			            }
		            }
		        }
			} catch (Exception e) {
				callbacks.printError(e.getMessage());
			}
		}
	}
	
	public class scan implements ActionListener{
		private IContextMenuInvocation invocation;

		public scan(IContextMenuInvocation invocation) {
			this.invocation  = invocation;
		}
		
		@Override
		public void actionPerformed(ActionEvent event) {
			try {
				IHttpRequestResponse[] messages = invocation.getSelectedMessages();
				for (IHttpRequestResponse message:messages) {
					byte[] req = message.getRequest();
					IRequestInfo analyzedRequest = helpers.analyzeRequest(req);
					URL url = analyzedRequest.getUrl();
					IHttpService service = message.getHttpService();
					boolean useHttps = service.getProtocol().equalsIgnoreCase("https");
					/******************发起扫描任务************************/
					callbacks.doActiveScan(service.getHost(), service.getPort(), useHttps, req);
					stdout.println(url.toString()+"被加入了扫描队列");
					/******************发起爬行任务************************/
					callbacks.sendToSpider(url);
					
					/******************查询URL是否在scope中************************/
					if (callbacks.isInScope(url)) {
						/******************从scope中移除************************/
						callbacks.excludeFromScope(url);
					}
					URL shortUrl = new URL(service.toString());
					/******************加入scope中************************/
					callbacks.includeInScope(shortUrl);
				}
			} catch (Exception e) {
				callbacks.printError(e.getMessage());
			}
		}
	}
}
```

### IMessageEditorTab

用途：有点像repeater模块，主要就是用来展示和编辑request和response。





```java
//添加一个editor，用于显示HTTP数据包
		IMessageEditor editor = BurpExtender.getCallbacks().createMessageEditor(this, false);
		contentPane.add(editor.getComponent(), BorderLayout.CENTER);
		
			JButton btnNewButton = new JButton("displayRequest");
		btnNewButton.addActionListener(new ActionListener() {
			public void actionPerformed(ActionEvent e) {
				editor.setMessage(getRequest(), true);//调用IMessageEditorController的函数来显示请求包
			}
		});
```



```java
 public Tags(final IBurpExtenderCallbacks callbacks, String name) {
        this.callbacks = callbacks;
        this.tagName = name;

        SwingUtilities.invokeLater(new Runnable() {


            @Override
            public void run() {


                jPanel = new JPanel();

               splitPanel = new JSplitPane(0);
                splitPanel.setDividerLocation(0.5D);
                jTabbedPane = new JTabbedPane();
                jTabbedPane1 = new JTabbedPane();
                request  = callbacks.createMessageEditor(Tags.this, false);
                response = callbacks.createMessageEditor(Tags.this,false);
                jTabbedPane.addTab("Request",request.getComponent());
                jTabbedPane.addTab("Response",response.getComponent());
                splitPanel.add(jTabbedPane);
                splitPanel.setTopComponent(jTabbedPane1);
                splitPanel.setBottomComponent(jTabbedPane);
                jPanel.add(splitPanel);

//                // 设置自定义组件并添加标签
                callbacks.customizeUiComponent(jPanel);//根据Burp的UI样式自定义UI组件，包括字体大小、颜色、表格行距等。
                callbacks.addSuiteTab(Tags.this);

            }
        });
    }
```



### IBurpCollaboratorClientContext

如何与burp自生DNSlog进行交互

```java
IBurpCollaboratorClientContext ccc = callbacks.createBurpCollaboratorClientContext();

//返回结果类似：053bsqoev8gezev8oq59zylgv71xpm, 053bsqoev8gezev8oq59zylgv71xpm.burpcollaborator.net
public static String[] getFullDnsDomain(){
    String subdomain = ccc.generatePayload(false);
    String interactionID = subdomain;
    String server = ccc.getCollaboratorServerLocation();
    String fullPayload = subdomain+"."+server;
    String[] result = {interactionID,fullPayload};
    return result;
}
```





## 插件解析

解析P喵呜大佬写的，shiro被动扫描插件。

前提：了解shiro反序列化漏洞原理，了解burp插件开发。



https://github.com/pmiaowu/BurpShiroPassiveScan.git



找到入口类开始看，发现继承了IScannerCheck。

IScannerCheck和IHttpListener有点相似，简单说下我理解的区别，如有不对欢迎指出。

IHttpListener有点像proxy抓包拦截，能在收到某某request或者response就马上进行处理。而IScannerCheck则是发现你请求了某某url之后，单独对这个url进行漏洞扫描。

![image-20211226144135012](https://gitee.com/safe6/img/raw/master/image-20211226144135012.png)



先看关键注册方法，主要是一些初始化操作

![image-20211226144243446](https://gitee.com/safe6/img/raw/master/image-20211226144243446.png)



初始化关键对象

![image-20211226144619862](https://gitee.com/safe6/img/raw/master/image-20211226144619862.png)

初始化两个用于存放扫描url和domain的对象

![image-20211226144729787](https://gitee.com/safe6/img/raw/master/image-20211226144729787.png)

这两个对象主要是用了单例各种维护一个了map

![](https://gitee.com/safe6/img/raw/master/image-20211226145014584.png)



初始化插件界面，设置插件名称，注册扫描，打印一些作者信息。

![image-20211226145255622](https://gitee.com/safe6/img/raw/master/image-20211226145255622.png)



看看插件界面代码，就是表格加req和res

![](https://gitee.com/safe6/img/raw/master/image-20211226151549825.png)



接下来看，被动扫描。核心代码都在这。

![image-20211226152831719](https://gitee.com/safe6/img/raw/master/image-20211226152831719.png)

先判断域名有没有被扫描过

![image-20211226153017814](https://gitee.com/safe6/img/raw/master/image-20211226153017814.png)

再检测url有没有被扫描过（此处有重复代码的感觉，离谱）

![image-20211226153117727](https://gitee.com/safe6/img/raw/master/image-20211226153117727.png)



都没有扫描过，则加到前面初始化的map里面去。(大佬居然管这个叫数组，有点离谱，问题不大)

![image-20211226153559483](https://gitee.com/safe6/img/raw/master/image-20211226153559483.png)



开始核心的各种检测

![image-20211226155021186](https://gitee.com/safe6/img/raw/master/image-20211226155021186.png)



后面就是一些添加issues，爆破key什么的，不想看了。

突然发现写这种东西，对我完全没任何好处。而且还及其浪费时间，疯狂截图，各种写。



## 致谢

感谢大佬开源，向大佬致敬。

- https://github.com/bit4woo/burp-api-drops
- https://xz.aliyun.com/t/7065
